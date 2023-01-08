---
layout: post
title: kafka, zoookeeper, python 실습
author: lazyer
tags: [kafka]

---



카프카의 개념에 대해 알아보았으니 이제 간단한 실습을 해보자.

<br>

## 설치 및 실행

<br>

**[JAVA 설치]**

실행하려는 kafka 는 apache kafka로 java가 설치되어있어야한다. 따라서 **java를 먼저 설치**해주었다.

```bash
# install java
brew tap AdoptOpenJDk/openjdk
brew install --cask adoptopenjdk8
```

<br>

**[kafka & zookeeper 다운로드 및 실행]**

다음으로 해당 [사이트](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.8.0/kafka_2.13-2.8.0.tgz)에서 kafka를 다운받았는다. ([`kafka_2.13-2.8.0.tgz`](https://archive.apache.org/dist/kafka/2.8.0/kafka_2.13-2.8.0.tgz) 를 다운받았다.) 압축을 해제하면 폴더 내부에 bin 과 config 폴더를 확인할 수 있다. config 파일 내부에서 **zookeeper의 port가 2181로 설정**되어 있는것을 확인할 수 있었다.

터미널에서 zookeeper를 실행한다.

```bash
$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

별도의 터미널에서 kafka도 함께 실행한다.

```bash
$ bin/kafka-server-start.sh config/server.properties
```

<br>

## Zookeeper

<br>

kafka는 비동기 처리를 위한 메시지 브로커이다. 메시지를 전송하기 위해 브로커를 제공한다.

<br>

zookeeper는 이러한 브로커의 메타정보를 관리하고 컨트롤러를 선출하는 역할을 한다.  kafka는 고가용성을 위해 다수의 broker를 지원하고, 이러한 broker들의 관리를 위해 zookeeper를 사용하여 클러스터 상태를 유지한다.

**따라서 producer는 zookeeper를 통하여 kafka의 어느 broker를 이용해야할지 정보를 전달받고, consumer는 broker의 topic 내부에 존재하는 partition에서 어느 위치의 message를 읽어야할지 offset 정보를 전달**받는다.

<br>

consumer에서 broker에 대한 정보를 별도로 전달받지않는 이유는, 앞서 블로그에서 설명한 내용과 같이 consumer group의 consumer의 topic의 partition과 1:1로 매칭되기 때문에 broker의 정보는 추가로 필요하지 않다.

zookeeper가 메타데이터의 정보를 관리하지만, producer와 consumer가 zookeeper와 통신을 하지는 않는다.

zookeeper는 구조적으로 kafka 뒷쪽에서 동작한다.

<br>

## Topic

<br>

zookeper, kafka를 실행하였으니 이제 새로운 **터미널에서 producer를 생성**해보자.

```bash
$ bin/kafka-topics.sh --create --partitions 1 --replication-factor 1 --topic test --bootstrap-server localhost:9092
```

topic 생성 후 아래 명령어로 생성된 topic의 목록을 확인할 수 있다.

```bash
$ bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

<br>

## Producer

이제 **이벤트를 발행하는 producer를 실행**해보자.

```bash
$ bin/kafka-console-producer.sh --topic test --bootstrap-server localhost:9092
```

<br>

`--bootstrap-server` 대신 `--broker-list` 변수도 존재하지만, kafka에서는 현재 `--bootstrap-server` 를 사용할 것을 권하고있다.

`9092`포트는 kafka의 포트이며 server.properties 파일에서 확인 가능하다. 따라서 해당 포트로 이벤트를 발생시키면 zookeeper가 아닌 kafka에서 이벤트를 수신하게된다.

발생시킨 이벤트는 로컬에 저장되고 `log.dirs` 파일의 값을 확인하여 저장된 경로를 알 수 있다. 기본적으로 `'tmp/kafka-logs`에 저장되고있다. 이벤트 발행 후 해당 경로에서 저장된 이벤트를 확인해보자.

<br>

## Consumer

마지막으로 producer가 발행한 event를 구독하는 consumer를 생성해보자.

```bash
$ bin/kafka-console-producer.sh --topic test --bootstrap-server localhost:9092
```

producer를 실행하면 콘솔에서 `>`가 표시되며 메시지를 입력할 수 있다.  메시지를 입력하면 consumer에서 event를 구독하여 메시지를 출력하는 것을 볼 수 있다.

<br>

## Python

위에서 생성한 producer에서 발행하는 event를 구독하는 consumer를 python으로 작성해보자.

먼저 kafka를 실행하기 위해 파이썬 라이브러리를 설치해준다.

```bash
pip install kafka-python
from kafka import KafkaProducer, KafkaConsumer
from json import dumps, loads

consumer = KafkaConsumer(
    'test',
    bootstrap_servers=['localhost:9092'],
)

for message in consumer:
    print(f'{message.topic=}, {message.partition=}, {message.value=}')
```

<br>

위 코드를 실행한 후 터미널에서 실행되고있는 producer에서 값을 입력하면 message가 출력되는 것을 확인 할 수 있다.

```python
message.topic='test', message.partition=0, message.value=b'hello’
```

<br>

`KafkaConsumer` 파라미터 값에는 여러가지가 있는데, 하나씩 테스트해보자.

- **auto_offset_reset = ‘earlist’**

  위 파이썬 consumer를 실행해보면 실행 후에 producer가 발행한 이벤트에 대해 출력하는 것을 볼 수 있다.

  `auto_offset_reset='earlist'`로 설정해보자. 해당 파라미터 값을 earlist로 지정하면 **카프카 토픽의 메시지 큐에 남아있는 모든 이벤트를 앞에서부터 순차적으로 구독**한다. 파이썬 코드를 실행하기 이전에 producer에서 발행한 이벤트가 함께 출력된다.

- **value_deserializer**

  위 메시지 출력 내용에서 `message.value=b’hello’` 를 확인할 수 있다.  카프카의 메시지 포맷은 기본적으로 byte array 이기 때문이다.

  `value_deserializer=lambda x: x.decode(’utf-8’)` 로 설정해보자. producer에서 이벤트를 발행하면 `message.value='hello'` 로 메시지가 디코딩되어 출력되는 것을 확인할 수 있다.

  **value_deserializer는 메시지를 원하는 포맷으로 변환시켜주는 역할**을 한다.

- **consumer_timeout_ms** `consumer_time_ms=10000`으로 설정해보자. 해당 파라미터값을 설정하지않으면 consumer은 계속하여 이벤트 발행을 대기하게 된다. **해당값을 설정하면 해당 값(10000ms)동안 이벤트가 발생하지 않을 시 구독을 종료**한다.

  <br>

------