---
layout: post
title: aws ec2 볼륨 축소
author: lazyer
tags: aws
---



AWS를 사용하다보니 EC2 비용 못지않게 EBS 비용이 많이 나왔다.

AMI로 EC2 생성 시 용량 낮게 설정하여 새로운 AMI를 생성하려고 하였는데 AMI 생성 당시의 스토리지 용량보다 적은 용량으로는 EC2 생성이 불가능 하였다. 

따라서 용량을 낮춘 빈 볼륨을 생성하여 기존 EC2의 모든 데이터를 복사하는 방법으로 진행하려고 한다.



위 방법에 대한 설명은 [AWS EC2 루트 EBS 볼륨 용량 줄이기](https://stack.news/2020/11/23/aws-ec2-%EB%A3%A8%ED%8A%B8-ebs-%EB%B3%BC%EB%A5%A8-%EC%9A%A9%EB%9F%89-%EC%A4%84%EC%9D%B4%EA%B8%B0/) 해당 링크에 잘 설명되어있는데, 볼륨 포맷이 ext4 기준으로 설명되어있다. 나는 볼륨 포맷이 xfs 였기 때문에 포맷, 라벨 설정 등의 명령어가 조금 달랐다.

xfs 포맷의 인스턴스 용량을 줄이는 법은 아래와 같다.



**[준비]**

- AWS 이미지 > AMI에서 용량을 줄일 대상 EC2 생성 (`ec2-old`)
  (ec2-old의 볼륨을 `VA`라고 한다.)
- AWS EBS > 볼륨에서 내용을 복사할 볼륨(`VB`) 생성
- 작업을 수행할 임시 EC2 생성. ec2-old와 동일한 운영체제로 생성한다. (`ec2-main`)
- ec2-old, ec2-main 종료
- ec2-old에서 VA 분리 후 VA, VB를 ec2-main에 연결.

작업을 편하게 진행하기 위해 public subnet에 ec2-main, ec2-old를 생성하였다.



작업 진행 단계는 아래와 같다.

1. VB 포맷 및 라벨 변경

2. 마운트 및 복사

3. VB UUID 변경

4. 부트로더 설치



### 1. VB 포맷

아래 명령어들을 실행하기 위해 root 계정으로 접속한다.



```shell
$lsblk -f
```

```
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda
└─xvda1 xfs    /     8562e9fb-f45b-4a09-9778-bde97be4afb3 /
xvdf
xvdg
└─xvdg1 xfs    /     417df3d5-5cb9-4b5e-a1a2-8475fff8efc9 /source
```

볼륨이 EC2에 연결되었다면 위 명령어로 확인이 가능하다.

xvdf, xvdg 중 어디에 마운트 되어있는지는 변경될 수 있는데, 여기서는 VA가 xvdg, VB가 xvdf에 마운트되었다. VB는 새로 생성한 볼륨으로 파티션 및 LABEL이 없는 것을 확인 할 수 있다.



새로 생성한 VB에 파티션을 생성해준다.

```shell
$ fdisk /dev/xvdf
```

`n > p > enter > enter > enter > w` 순으로 입력하여 파티션을 생성 할 수 있다.



다시한번 확인해보면 파티션(xvdf1)이 생성되어 있는것을 확인 할 수 있다.

```shell
$lsblk -f
```

```
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda
└─xvda1 xfs    /     8562e9fb-f45b-4a09-9778-bde97be4afb3 /
xvdf
└─xvdf1
xvdg
└─xvdg1 xfs    /     417df3d5-5cb9-4b5e-a1a2-8475fff8efc9
```



xvdf1 하위에 FSTYPE, LABEL, UUID 부분이 비어있는데, FSTYPE과 LABEL을 먼저 생성해준다. 

VA의 FSTYPE이 ext4라면 ext4로 포맷해주면 된다. 나는 xfs 포맷이기 때문에 xfs 로 포맷하였다.

(FSTYPE은 File System Type의 약자이다.)

```shell
$mkfs -t xfs /dev/xvdf1
```



### 2. 복사 및 부트로더 설치

```shell
$mkdir /target /source
$mount -t xfs /dev/xvdg1 /source
$mount -t xfs /dev/xvdf1 /target
$rsync -vaxSHAX /source/ /target
```

VA를 /source에, VB를 /target 디렉터리에 마운트 해주고 복사를 시작한다. 여기서 부터는 시간이 조금 소요된다.

UUID 관련 에러가 발생하는 경우 아래와 같이 nouuid 옵션으로 실행한다.

```shell
$mount -t xfs -o nouuid /dev/xvdf1 /target
```

복사가 완료되면 VB에 부트로더를 설치해준다.

```shell
$grub-install --root-directory=/target /dev/xvdf
```

만약 `grub-install` 명령어가 없다는 에러가 발생하면 `grub2-install`로 바꿔준다.

부트로더 설치까지 완료화면 마운트를 해제한다.

```shell
$umount /target /source
```



### 3. VB UUID/LABEL 변경

복사를 끝내고나면 VB에 UUID가 생성되는데 /dev/xvdg1와 /dev/xvdf1의 LABEL과 UUID가 다른것을 확인 할 수 있다.

```shell
$lsblk -f
```

```
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda
└─xvda1 xfs    /     8562e9fb-f45b-4a09-9778-bde97be4afb3 /
xvdf
└─xvdf1 xfs          5e75d92d-2b7d-4b12-b669-10be5159418f
xvdg
└─xvdg1 xfs    /     417df3d5-5cb9-4b5e-a1a2-8475fff8efc9
```

VB의 UUID를 VA의 UUID와 동일하게 설정한다.

```shell
$xfs_admin  -U 417df3d5-5cb9-4b5e-a1a2-8475fff8efc9 /dev/xvdf1
```

```shell
$lsblk -f
```

```
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda
└─xvda1 xfs    /     8562e9fb-f45b-4a09-9778-bde97be4afb3 /
xvdf
└─xvdf1 xfs          417df3d5-5cb9-4b5e-a1a2-8475fff8efc9
xvdg
└─xvdg1 xfs    /     417df3d5-5cb9-4b5e-a1a2-8475fff8efc9
```

라벨도 VA와 동일한 값으로 설정해주면 된다. VA의 라벨이 `/`이기 때문에 동일하게 설정하였다.

```shell
$xfs_admin -L / /dev/xvdf1
```

다시 확인해보면 VB의 FSTYPE과 LABEL을 모두 확인 할 수 있다.

```shell
$lsblk -f
```

```
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda
└─xvda1 xfs    /     8562e9fb-f45b-4a09-9778-bde97be4afb3 /
xvdf
└─xvdf1 xfs    /     417df3d5-5cb9-4b5e-a1a2-8475fff8efc9
xvdg
└─xvdg1 xfs    /     417df3d5-5cb9-4b5e-a1a2-8475fff8efc9
```






볼륨 복사 작업은 모두 마무리 되었다!

이제 ec2-main EC2를 종료하고 VA, VB 볼륨을 모두 분리 한 뒤 ec2-old EC2에 VB를 연결하여 인스턴스를 시작하면 축소된 볼륨을 이용하여 부팅되는 것을 확인 할 수 있다.