---
layout: post
title:  "Apache Kafka - Quick Start"
date:   2022-01-17 16:00:00 +0900
categories: Platform
---
created : 2022-01-17, updated : 2022-01-17


# Introduction
Kafka는 기존의 Message Broker 의 기능을 업그레이드하여 BigData의 분석, Pipeline 등 대용량 분산환경에서 사용할 수 있는 Opne Source 이다. 대용량, 안정성(Partitioning, Relipcation) 을 보장하고 있어 운영환경에서는 상당한 하드웨어/VM을 필요하여 나도 차일피일 사용을 미루고 있었다. 이제는 Kafka을 Cloud 환경에서 제공하는 업체(Confluent, AWS, GCP, Azure 등)가 생겨나 대용량 시스템 운영을 개발자도 할수 있기에 이제 다시 마음먹고 시작해본다. 

먼저 Kafka를 Local 환경(Linux)에서 구동을 하고 Console에서 메세지를 보내고 받도록 해볼 것이다. Kafka은 단순한 Message Queue/Broker를 넘어 다양한 Connector등을 제공하고 정보를 휘발성이 아닌 원하는 만큼 저장할 수 있는 특징도 가지고 있다. 


# Use Cases
Kafka가 사용되는 몇가지 사용 사례를 정리해보겠다. 공식 문서의 내용을 근거로 정리를 해보았다.

## Messaging
기존의 Open Source Message broker보다 더 좋은 점이 많다. Partitioning, Replication 과 Fault-tolerance가 보장되어 Large scale 메시지 Application에 사용하기에 적합하다.

## Website Activity Tracking
Kafka의 기본 사용은 실시간 Publish-subscribe feed에 사용할 수 있다. 이는 Site에서의 Activity를 각각의 topic으로 전송하여 Real-time processing, Real-time monitoring을 할 수 있다. 이런 Activity Tracking은 많은 사용자가 사용하는 activity를 메세지로 전송하기에 큰 볼륨이 필요하다.

## Metrics
모니터링 데이터를 운영/관리하는데 사용한다. 분산된 Application 에서 발생된 Metrics를 중앙으로 수집할때 사용한다. 

## Log Aggregation
Log Aggregation은 물리 Log 파일을 수집하여 중앙의 저장소(HDFS 같은)로 저장을 한다. 이때 kafka를 이용하여 분산된 다양한 Data들을 적은 지연으로 수집할 수 있다. 

## Streaming Processing
다양한 계층에서의 프로세싱 Pipeline으로 사용합니다. 여러 Topic을 이용하여 데이터의 집계를 하거나 다음 프로세스로 전달하는데 사용하고 있습니다. Kafka Stream Linrary로 Application/Client 개발을 도와주고 있습니다.  

## Event Sourcing
Event에 따른 상태를 지속적으로 저장하여 관리하는 기법이다. Event-Driven Architecture차이가 있지만, 두가지 모두에서 사용될 수있다. Event Sourcing 관점에서는 많은 Application에서 발생되는 Event를 한곳으로 모으고 관리하는데 사용할 수 있다. Event-Driven 관점에서는 Event가 정확하게 다른 시스템에 전달되도록 할수 있기에 많이 사용되고 있다.  

> Note. Kafka 이전에도 RabbitMQ, Redis 등 다양한 Message Broker가 있었다. 하지만, Production 환경에서 신뢰를 가지기에 시스템 구축이 쉽지 않았었다. Kafka는 기존 Broker들의 문제점을 보강하였다. (현재(2022년초)에는 다른 RabbitMQ, Redis도 많은 발전을 하여 신뢰성과 운영이 많이 좋아졌다고는 한다.)


# Requirements
본문서는 Kafka의 공식 Document에서 [Quick Start](https://kafka.apache.org/documentation/#quickstart)를 따라 하였다.

기본적으로 필요한 환경은 아래와 같다. 
- Java Runtime 8+
- Linux system (Ubuntu 20.04를 VirtualBox에 설치하였다) [설치문서](https://ahnchan.github.io/posts/Platform_virtualbox_ubuntu/)
- 여러개의 터미널을 연결이 필요하다. (ZooKeeper, Kafka, Produder, Consumer)

> Note: MacOS에서는 Consumer가 종료후 다시 접속하면 오류가 나와 Linux에서 진행을 하였다. 이 오류가 나오면 Kafka를 다시 시작해야했다. (원인을 찾지 못함) 

> Note. Windows에서 진행하려면 Docker 환경에 구성하는 것을 추천한다. (향후 Tutorial을 올릴것이다.)


# Apache Kafka 다운로드/설치
공식 사이트에서 최신 Apache Kafka 를 [다운로드](https://kafka.apache.org/downloads) 받아야 한다. Ubuntu를 VirtualBox에 설치하는 것을 권장하였기으니 Ubuntu Server내에서는 아래의 명령으로 다운로드를 받을 수 있다. 파일의 버전은 사이트에 명시된 최신 버전으로 변경하여 실행을 하자.

```
$ wget https://dlcdn.apache.org/kafka/3.0.0/kafka_2.13-3.0.0.tgz
```

다운로드가 완료되면 압축을 풀고 해당 디렉토리로 이동한다.

```
$ tar -xzf kafka_2.13-3.0.0.tgz
$ cd kafka_2.13-3.0.0
```

# Kafka를 시작한다.
Kafka구동을 위해서는 Java 8+ 이상이 필요하다. ZooKeeper를 먼저 구동하야한다. ZooKeeper는 Kafka를 클러스터로 구성하기 위해 각각의 Broker를 관리해준다. 한개의 Kafka Broker를 구동하여도 ZooKeeper의 구동이 필요하다.

> Note. 공식문서를 보니 ZooKeeper는 향후에는 필요없어질 거라고 한다.

```
$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

터미널을 새로 연결하고 Kafka를 시작한다.

```
$ bin/kafka-server-start.sh config/server.properties
```

구동이 되면서 Log가 화면에 표시된다. 오류 메세지가 없으면 정상적으로 구동된 것이다. 


# Topic을 생성한다.
Topic은 Event/Message가 이동하는 통로로 생각하면된다. 보통 업무 특성에 따라 여러개의 Topic을 사용하여 분산된 Application이 상호연동을 하여 사용한다. 

```
$ bin/kafka-topics.sh --create --partitions 1 --replication-factor 1 --topic quickstart-events --bootstrap-server localhost:9092
Created topic quickstart-events.
```

- topic: quickstart-events 라는 이름의 Topic을 생성한다.
- bootstrap-server: Broker의 접속정보를 입력한다. 
- partions : 몇개의 파티션으로 나누어서 관리할지를 명시한다. 개발환경이니 1로 정한다.
- replication-factor : 몇개의 복제본을 가지고 있을지를 명시한다. 개발환경이니 1로 정한다.

생성이 잘되었는지 확인를 해본다. 전체 Topic의 리스트를 요청한다. 방금 생성한 quickstart-events 가 보이면 문제없이 잘 생성된 것이다. 

```
$ bin/kafka-topics.sh --list --bootstrap-server localhost:9092
quickstart-events
```

quickstart-events 의 자세한 정보를 확인해보자. 생성시 설정한 Partition, Relication 정보를 볼수 있다.

```
$ bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092
Topic: quickstart-events        TopicId: -CQEUdJiTmWih8mPpDZATg PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: quickstart-events        Partition: 0    Leader: 0       Replicas: 0     Isr: 0
```


# Event를 발생하고 받아보기
이제 실제로 Event를 발생해보자. Kafka에서는 기본으로 몇가지 Client를 제공하고 있다. 이중 Console에서 입력이 가능한 Console producer를 실행해보자. 아래의 명령을 실행하고 > 가 나오면 메세지를 입력해본다. 

```
$ bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
>event #1
>event #2
```

다른 터미널에서 Console consumer를 실행해보자. producer에서 보낸 이벤트를 확인할 수 있다. 

```
$ bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
event #1
event #2
```

- from-beginning: 처음부터 모든 event를 수신한다. 이 명령이 없으면 접속후에 발생된 Event들만 수신한다.

Consumer를 새로 시작해도 언제나 위와 같이 모든 메세지를 수신한다.(--from-beginning을 사용시) kafka의 특징이 이벤트/메세지를 원할때까지 정보를 저장하도록 되어 있다. 대용량으로 처리를 위해 Partitioning을 하고 데이터의 안정적인 저장을 위해 Replication까지 설정할 수 있게 되어 있다. Production 환경에서는 ksqlDB를 이용하여 더욱더 정보의 안정성을 높이고 있다. 


# Conclusions
Kafka는 기존의 Message Queue와는 차별되게 대용량의 처리를 기본으로 설계되었다. 그래서 구성하는데 다소 많은 VM/Hardware가 필요하다. 추가로 정보도 원하는 시간까지 저장이 가능하다. 이러한 특성이 단순한 Queue가 아닌 Event-Driven Architecture를 구성할 때 사용하고 있다. 
개발자의 입장에서는 Kafka의 운영이 부담스러울것이다. 이제는 Confluence Cloud나 Cloud Service (GCP, Azure, AWS 등)의 서비스를 이용하면 큰 부담없이 대용량의 분산처리를 위해 중심이 되는 Message Broker를 운영할 수 있다. 


# References
[Kafka Quick Start](https://kafka.apache.org/documentation/#quickstart)

[Ubuntu 20.04를 VirtualBox에 설치하기](https://ahnchan.github.io/posts/Platform_virtualbox_ubuntu/)


