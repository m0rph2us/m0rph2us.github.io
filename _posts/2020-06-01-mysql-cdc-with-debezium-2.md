---
layout: post
excerpt_separator: <!--more-->
title: 'MySQL CDC With Debezium #2'
categories: mysql cdc debezium
---

# 디비지움으로 MySQL CDC 하기 #2
## 개요

이번 포스팅에서는 작은 예제를 통해 디비지움을 어떻게 사용하는지 간략하게 살펴보려고 한다.
<!--more-->

## 아키텍처

우리가 예제로 만들 작은 시스템의 아키텍처는 다음과 같다. 실제 프로덕션에서 사용하는 아키텍처도 이와 크게 다르지 않다. 기본 뼈대는 같고 단지 HA 구성을 위해 
Zookeeper, Kafka 가 더 추가되는 정도이다. 프로덕션 고려사항에 대해서는 다음 포스팅에서 살펴보도록 할 것이다.

{:refdef: style="text-align: center;"}
![architecture](/assets/simple-debezium-usage-architecture.png)
{: refdef}

위의 아키텍처에서 각각은 다음과 같다.

* MySQL Master DB #1
  * 데이터 변경을 캡처하길 원하는 원본 DB 이다.
* Debezium
  * 디비지움 커넥터이다. 원본 MySQL 데이터베이스에 리플리케이션 클라이언트로 붙어 데이터 변경을 캡처하고, 변경 메시지를 카프카에 프로듀싱하는 역할을 한다.
* Schema Registry
  * 스키마 레지스트리이다. 보통 JSON 같은 직렬화 규격을 사용하기도 하지만, JSON 은 타입을 엄격하게 처리할 수 없다는 특성이 있다. 따라서 AVRO 직렬화 규격을 
  사용하는데, 이때 공용 스키마 저장소 같은 역할을 하는 것이 바로 스키마 레지스트리이다. 이 레지스트리에 스키마 정보를 저장해두고, 직렬화와 역직렬화를 수행할 때 
  이를 참조하여 수행하게 된다.
  * 이 부분은 필요하다면 바로 앞단에 LB 구성을 추가하여 가용성을 좀 더 높일 수 있다.  
* Zookeeper
  * 카프카 클러스터의 상태를 관리한다.
* Sample App
  * 카프카 토픽에 인입된 메시지를 읽어서 처리하는 애플리케이션이다. 이는 스트림 프로세싱 애플리케이션이 될 수도, 데이터를 말단에서 처리하는 싱크 애플리케이션이 
  될 수도 있다. 우리가 주로 코딩하는 부분은 이 부분이며, 주로 데이터를 변형(transform)하고 대상 데이터베이스에 적용하는 로직을 수행한다. 
* MySQL Master DB #2
  * Sample App 에서 처리한 데이터가 최종적으로 저장되는 DB 이다.

미리 예제를 만들어 놓았으며, 해당 예제는 단순히 DB1 에서 일어나는 C(Create), U(Update), D(Delete)를 그대로 DB #2 에 적용하는 아주 간단한 예제이다. 
사실 단순히 테이블 대 테이블로 데이터를 적용할 목적이라면 DB 에서 제공하는 리플리케이션 기능을 최대한 활용하는 것이 좋다. CDC 는 여러 여건상 그것이 도저히 
안될 때만 최후의 수단으로 사용하는 것이 좋다.

## 예제 맛보기

먼저 미리 만들어둔 예제 코드를 깃헙에서 내려받아 실행해보도록 하자.

```shell
git clone https://github.com/m0rph2us/docker-debezium.git
```

README 의 지침대로 실행한 다음 DB #1 에서 CUD 를 실행해보면 DB #2 에 그대로 적용되는 것을 알 수 있다.

## 데이터 전달 흐름

데이터 전달 흐름은 다음과 같이 이어진다.

```
#######        ############       #########       ##############       #######
# DB1 # ---->  # Debezium # ----> # Kafka # ----> # Sample App # ----> # DB2 #    
#######        ############       #########       ##############       #######
```

## MySQL 사전 요구사항

먼저 CDC 원본 DB 서버가 binlog 를 기록할 수 있도록 구성해야 한다.

```shell
log-bin=mysql-bin
binlog_format=row
binlog_row_image=full
```

만약에 슬레이브 노드를 CDC 원본 DB 서버로 사용한다면 위의 설정과 함께 다음 설정을 추가해야 한다.

```shell
log_slave_updates=1
```

디비지움은 리플리케이션 클라이언트처럼 동작하기 때문에 적합한 권한을 가진 계정이 있어야 한다. 다음과 같이 계정을 생성하면 된다.

```shell
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO `[your username]`@`[allowed host]` IDENTIFIED BY '[password]';
```

그리고, 생성된 계정으로 서버에 접속한 다음 `show binary logs` 를 실행해 보면 binlog 파일을 확인할 수 있다.

공식적으로는 MySQL 5.7 이후 버전부터 지원한다. 하지만 약간의 패치와 권한 부여를 통해 5.6 이하 버전에서도 사용이 가능하도록 만들 수 있다.

## 디비지움 커넥터 설정

다음은 커넥터 설정 샘플이다. 

```shell
{
        "name": "mysql-connector-db1",
        "config": {
                "connector.class": "io.debezium.connector.mysql.MySqlConnector",
                "database.hostname": "mysql-master-db1",
                "database.port": "3306",
                "database.user": "cdc_user",
                "database.password": "1234",
                "database.server.id": "3141592",
                "database.serverTimezone": "Asia/Seoul",
                "database.server.name": "cdc_db1",
                "database.history.kafka.bootstrap.servers": "kafka-1:9092",
                "database.history.kafka.topic": "dbhistory.db1",
                "include.schema.changes": "true",
                "table.whitelist": "sample.tb_user"
        }
}
```

여기에서 중요한 부분은 다음과 같다.

* `database.server.id`
    * 디비지움 커넥터의 ID 이다. 같은 데이터베이스를 바라보는 커넥터(슬레이브) 사이에서 이 아이디는 고유해야 하며, 그렇지 않을 경우 예상치 않게 동작할 수 
    있다.
* `database.server.name`
    * 디비지움 커넥터의 논리명이다. 이 논리명도 고유해야 한다. 이 이름은 커넥터에서 사용하는 모든 토픽의 접두사로 사용된다. 영숫자, 문자, 밑줄만 사용할 수 
    있다.
    * `include.schema.changes` 가 true(기본)인 경우 이 토픽에 스키마 변경 이벤트가 기록된다. 변경 이력을 저장하는 것과는 별개이다.
* `database.history.kafka.topic`
    * 디비지움은 원본 DB 의 스키마 변경을 항상 추적하여 이 토픽에 이력을 저장해 놓는다.
    
위와 같이 설정하는 경우 디비지움은 처음에 다음과 같이 수행하게 된다. 

1. 다른 클라이언트가 쓰기를 수행하지 못하도록 전역 읽기 잠금을 설정한다.
2. 후속 쿼리에서 일관적인 읽기를 보장하기 위해 REPEATABLE_READ 트랜잭션 격리 수준으로 트랜잭션을 시작한다.
3. 현재 binlog 포지션을 읽는다.
4. 데이터베이스와 테이블 스키마를 읽는다.
5. 전역 읽기 잠금을 해제한다.
6. DDL 변경을 스키마 변경 토픽에 기록한다. 전역 읽기 잠금이 DDL 까지 막지 못하므로, 잠금을 해제한 다음 마지막 포지션부터 변경된 DDL 을 적용하는 것으로 
이해하면 된다.
7. 데이터베이스 테이블을 스캔하여 테이블용 카프카 토픽에 각 행마다 CREATE 이벤트를 생성한다. 즉, 데이터 사본을 카프카에 행 단위로 프로듀싱한다고 생각하면 
된다. 이 과정이 스냅삿 과정인데, `snapshot.mode` 를 설정하여(기본은 `initial`) 스냅샷을 생략하도록 만들 수 있다.
8. 트랜잭션을 커밋한다.
9. 커넥터 오프셋에 현재 포지션을 기록한다.

이후에는 binlog 포지션을 추적하면서, 변경 데이터와 스키마 변경을 캡처하고 이벤트로 프로듀싱하게 된다.

## 언제 사용하는 것이 좋을까?

주로 다음과 같은 상황에서 유용할 것이라 생각된다.

* 이종 데이터베이스 동기화
    * 예를 들어, MySQL 에서 발생한 변경 데이터를 MongoDB 나 일래스틱 서치로 실시간 적용하는 것처럼 말이다. 
* 실시간 데이터 마이그레이션 및 동기화
    * 레거시 서비스를 개선하여 새로운 서비스를 만들었다고 가정해보자. 새로운 스키마가 만들어졌고, 서비스 중단 없이 기존 데이터를 마이그레이션하면서 레거시 
    서비스를 새로운 서비스로 전환해야 한다면 좋은 수단이 될 수 있다.  
* 데이터의 상태 변이 추적
    * 상태 변이를 준실시간으로 추적할 수 있다. 예를 들자면 배송상태 변경을 추적하는 것이 이에 해당할 수 있다.
* 데이터 정규화 및 역정규화
    * 데이터 변형과정에서 단순 변환 뿐만이 아니라 정규화 또는 역정규화 역시 수행할 수 있다.

## 마무리

간략한 예제를 통해 디비지움을 어떻게 사용하는지 살펴보았다. 하지만, 말 그대로 이해를 돕기 위한 간략한 예제일 뿐이다. 실제 프로덕션에서 사용하기 위해서는 
고려할 요소가 더 존재한다. 다음 포스팅에서는 프로덕션에서 고려할 사항을 살펴볼 것이다.

## 참고

1. [샘플 클러스터 + 애플리케이션](https://github.com/m0rph2us/docker-debezium)