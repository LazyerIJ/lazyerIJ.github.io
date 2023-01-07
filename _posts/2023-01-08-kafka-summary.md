---
layout: post
title: kafka 개념과 구성요소
author: lazyer
tags: [kafka]
---



## 1. 개념

**메시지의 송신자와 수신자를 중개하는 시스템이다.** 대용량, 대규모 메시지 데이터를 빠르게 처리하도록 개발되었다.

<br>

**queue**와 같이 동작할 수 있지만 queue는 아니다. queue의 경우 consumer가 데이터를 처리하면 해당 데이터가 큐에서 삭제된다. 하지만 kafka에서 consumer가 message를 처리하여도 스트림의 데이터가 변경되지 않는다. 따라서 하나의 토픽(message)을 여러 시스템(consumer)에서 공유할 수 있다.

<br>

consumer가 데이터를 처리하는 중 오류가 발생하여도 데이터는 사라지지않기 때문에 해당 데이터를 다시 처리할 수 있다. 데이터간의 순서는 offset으로 별도로 관리되어 offset을 기준으로 에러가 발생한 데이터를 재처리 할 수 있다.

<br>

**[메시징 시스템]**

“publisher로부터 데이터를 일단 수신하고 consumer에게 적절한 타이밍에 데이터를 보내는 시스템”

![https://www.devkuma.com/docs/kafka/kafka-message-system.png](https://www.devkuma.com/docs/kafka/kafka-message-system.png)



우선 구조적인 부분만 살펴보면 **publisher는 broker에게 message를 보내고 subscriber는 broker로부터 message를 받는다.** 서로 message를 직접 주고받는 대신 broker를 이용하여 topic별로 message를 주고받는다.

<br>

topic은 message의 내용에 따라 분류되는 하나의 주제로 볼 수 있다. **publisher는 message를 어느 topic으로 저장할지를 지정하고,  subscriber는 어느 topic에서 message를 읽을지 선택**한다.

<br>

broker가 메시지를 중개함에 따라  **producer와 consumer는 서로에 대해 독립적**이다. 따라서 producer와 consumer의 추가/삭제가 상대적으로 편하게되어 구조적인 결합도가 낮아진다는 장점이 있다. **Scaling에서 유리**하기때문에 대용량의 데이터를 고속으로 처리할 수 있다.

<br>

앞서 말한바와 같이 broker의 topic에서 메시지를 처리하여도 해당 메시지는 삭제가 되지않기 때문에, **여러 subscriber에서 동일한 topic에 존재하는 message를 처리할 수 있다.**

위 그림에서 Topic A는 Subscriber 1, Subscriber 2 두 군데에서 처리되어지고 있다.

<br>

<br>

## 2. 구성요소



![https://www.devkuma.com/docs/kafka/kafka-message-system-detail.png](https://www.devkuma.com/docs/kafka/kafka-message-system-detail.png)

<br>

**[message]**

구성도에서 주고받는 개별 데이터를 message(event)라고 한다. producer와 consumer가 주고받는 데이터의 단위이다.  message는 key-value로 구성되어있다. 데이터 타입은 byte array이다.

<br>

**[producer]**

이벤트를 게시(post)하는 클라이언트 어플리케이션이다.

publisher의 역할을 하며, broker에 message를 송신한다.

<br>

**[consumer]**

producer가 게시하는 topic을 구독하고 처리하는 클라이언트 어플리케이션이다.

subscriber의 역할을 하며, broker에서 message를 수신한다.

수신한 데이터가 계속 보존되어있기때문에, 임의의 시점부터 message를 다시 요청할 수 있다.

<br>

**[broker]**

producer/consumer의 요구에 따라 message의 송수신을 시행한다. 더 빠른 처리를 위해 다수의 broker로 구성하는 것이 일반적이다. 메시지를 디스크에 쓰는 것도 가능하다. message는 모든 broker에 발행된다.

<br>

**[topic]**

broker내에서 message를 보관하는 영역이다. 같은 분류의 message는 같은 topic에 보존된다.

producer는 message를 특정 topic별로 보내고, consumer은 원하는 topic에 대해서만 message를 가져온다.

<br>

**[partition]**

broker에 존재하는 topic은 다시 1개 이상의 파티션으로 나눠져 저장된다. message가 kafka의 topic에 쓰여지는데에 시간이 소요되는데, 조금 더 빠른 처리를 하기 위하여 message를 병렬적으로 쓰기 위해 다수의 partition을 생성한다.

partition내부에서는 producer가 보낸 message가 순차적으로 쓰여지겠지만, partition 서로간에 message는 순차적으로 볼 수 없다.

partition은 확장은 가능하지만 축소는 불가능하다.

고가용성을 위한 각각의 broker내의 topic에도 partition이 존재하는데, broker 사이의 partition의 데이터 분산/복제 관리를 위해 replication 설정이 가능하다. (내용이 많아 다음에 따로 정리를 해보려고 한다.)

<br>

**[consumer group]**

partition 개념으로 producer-broker-consumer 구조가 조금 복잡해 보일 수 있다. 쉽게 표현하면 producer → broker - topic → consumer group으로 볼 수 있다.

topic 내에 message의 분산 처리를 위하여 partition이 1개 이상 존재하는데, 이 partition과 대응되는 것이 consumer group의 consumer이다.

하나의 consumer는 하나의 partition에서만 데이터를 받을 수 있기때문에, consumer group은 구독하는 topic 내의 partition 수보다 많은 consumer를 가질 수 없다.

---

<br>

**Reference**

- [Apache Kafka 기본](https://www.devkuma.com/docs/apache-kafka/)
