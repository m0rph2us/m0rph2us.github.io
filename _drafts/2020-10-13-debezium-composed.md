---
layout: post
excerpt_separator: <!--more-->
title: 'MySQL CDC With Debezium'
categories: mysql cdc debezium
---

# 디비지움으로 MySQL CDC 하기
## 개요

MySQL 테이블의 특정 칼럼 값이 변경될 때 해당 로우(row)의 데이터 또는 그와 연관된 정보가 준실시간으로 필요해졌다. 몇 가지 방식을 생각해 볼 수 있는데, 서로 장단점을 
따져보니 CDC(Change Data Capture)가 가지는 장점이 더 많았다. 
<!--more-->

이 문서는 CDC 가 어떤 장점을 가지는지, 기술 대중성이 있는지를 파악함에 목적이 있다.

## 고민

{:refdef: style="text-align: center;"}
![traditional api call process of MSA](/assets/traditional-api-call.png)
{:refdef}

다음의 고민들이 있다.

* B 서비스에서는 A 서비스가 소유한 데이터베이스의 데이터가 필요하다.
    * API 호출로 A 서버스로의 데이터 조회가 가능하지만, B 서비스는 전체적으로 이러한 정보 조회가 굉장히 빈번하기 때문에 대량의 API 호출을 유발한다.
* B 서비스에서는 A 서비스가 소유한 데이터베이스에서 데이터를 가져와야 하지만, A 서비스의 데이터베이스가 소유한 데이터를 제외하고 가져와야 한다.
    * 하나의 물리적 데이터베이스라면 LEFT 조인을 하면 되는 상황이지만, 불가능하다.
    * 이러한 연산을 API 에서 처리하기에는 너무 느리다.
* A 서비스가 어느 정도의 호출을 감당할 수 있는지 장담할 수 없다.
    
결론을 얘기하자면, 마이크로서비스 아키텍처에서 각자의 도메인이 소유한 데이터외에 타 도메인의 데이터도 필요한 상황이 생긴다. 하지만, 서비스들 사이의 데이터 조회를 
위해 API 를 호출하는 방식은 너무 느리고(모든 API 가 캐싱된 정보를 돌려준다면 모르겠지만), 성능적인 영향이 있는지 없는지 확인해야 할 지점이 생기게 된다. 
이 과정이 실제 업무에서는 굉장한 스트레스이다. 다른 서비스에 미칠 성능 영향을 유관부서 담장자들과 커뮤니케이션해야 하고, 개선해야 될 것이 있다면 그에 대응해야 하고, 
성능 문제가 없을 것이라 예상했지만 프로덕션 서비스를 시작했을 때 문제가 터지면 그때서야 부랴부랴 서버를 더 투입한다. 그렇게 해서 성능이 안정되면 다행이지만,
또 다른 서비스가 동일한 문제를 만들지 않으리란 법은 없다. 

{:refdef: style="text-align: center;"}
![actual need](/assets/actual-need.png)
{:refdef}

B 서비스에서 필요한 정보는 A 서비스가 가지는 정보의 극히 일부이다. 그게 아니라면 도메인의 대부분이 겹친다는 것이므로 B 서비스와 A 서비스는 합치는 걸 고려해 
봐야 할 것이다. 극히 일부의 필요한 데이터를 B 서비스가 소유한 데이터베이스에 실시간 반영할 수 있다면, 모든 데이터를 SQL 쿼리로 조회할 수 있기 때문에 고민 거리가 
많이 줄어들게 된다. 데이터는 알아서 배달이 될 것이고, 오로지 서비스 개발자의 오너쉽 도메인만 신경쓰면 된다.

## 전통적인 방식

몇 가지 전통적인 방식중 가장 많이 사용하는 방식은 데이터 변경 행위가 일어날 때마다 이벤트를 발생시켜주는 것이다.

{:refdef: style="text-align: center;"}
![traditional event driven](/assets/tranditonal-event-driven.png)
{:refdef}

하지만 이 경우에는 다음과 같은 잠재적인 문제가 존재한다.

* 운영, 버그 등의 이슈로 데이터베이스 데이터를 직접 변경해야 하는 경우, 이벤트가 누락될 수 있다. 물론, 이벤트를 수동으로 발생시킬 수도 있겠지만, 변경이 일어난 
대상을 특정하기 어렵다면 얘기는 달라진다.
* 데이터 변경 행위에 대한 이벤트를 발생시키기 위해 추가적인 코드가 필요하다. 데이터 변경이 단일 애플리케이션이나 서비스에서만 일어난다면 코드를 추가하는 것이 
어렵지 않겠지만, 그렇지 않다면 코드 추가는 다소 부담스러워진다. 레거시 시스템들은 대부분 이런 부분에 있어 파악이 어렵거나 코드 추가가 어려운 경우가 많다.
* 애플리케이션 버그 및 이벤트 스토어 장애에 의해 이벤트가 누락될 수 있다.

그 외에 배치, 리플리케이션 데이터베이스를 사용하는 방법도 있겠으나, 배치는 실시간 처리가 힘들고, 서로 다른 서비스가 서로 다른 도메인의 전체 데이터베이스를 
참조하는 것은 좋은 디자인이라고 볼 수 없다. 서비스는 각자 독립적으로 성장할 수 있어야 하고 최대한 느슨한 결합을 가지는 것이 좋다. 

## CDC(CHANGE DATA CAPTURE)

CDC 는 변경 데이터 포착(Change Data Capture)이다. 주로 데이터베이스같은 데이터 스토어의 데이터 변경을 포착하여 ETL, 감사(audit), 캐싱과 같은 다양한 
후속 처리를 하는 데 사용된다.

본인은 MySQL 데이터베이스를 주로 사용하기 때문에 MySQL CDC 솔루션들을 찾아봤다.

[https://github.com/wushujames/mysql-cdc-projects/wiki](https://github.com/wushujames/mysql-cdc-projects/wiki)

대충 훑어보면 플립카트, 링크드인, 알리바바등이 내부적으로 CDC를 사용하고 있는 것으로 보인다.

플립카트의 경우:
>Data change propagation from source to consumers is a fairly common requirement in systems that automate business processes. 
>An example is inventory updates on a warehousing system reflecting on product pages of an eCommerce portal.
> 
>원천에서 컨슈머로의 데이터 변경 전파는 비지니스 프로세스를 자동화하는 시스템에서 상당히 일반적인 요구사항이다. 이커머스 포탈의 프로덕트 페이지에 반영되는 웨어하우징 
>시스템의 재고 업데이트가 그 예이다.

링크드인의 경우:
>We have built Databus, a source-agnostic distributed change data capture system, which is an integral part of LinkedIn's 
>data processing pipeline.
>
>우리는 소스에 구애받지 않는 분산 변경 데이터 포착 시스템인 Databus를 만들었으며, 이는 LinkedIn의 데이터 처리 파이프 라인에 없어서는 안될 부분이다.

국내의 경우 카카오뱅크, 카카오커머스, 라인커머스가 CDC 를 사용하고 있다.

여러 솔루션들 중 Kafka와 가장 궁합이 잘 맞을 것으로 판단되는 디비지움(Debezium)을 선택했다.

## DEBEZIUM

디비지움은 CDC 를 수행하기 위한 오픈소스 분산 플랫폼이다.

여기 블로그에 가보면 왜 CDC 인가? 라는 내용이 다음과 같이 설명되어 있다.

>Finally, CDC can also play a vital role in microservices architectures; exchanging data between services and keeping 
>local views of data owned by other services achieves a higher independence, without having to rely on synchronous API calls.
>
>마지막으로 CDC는 마이크로 서비스 아키텍처에서 중요한 역할을 수행할 수 있다. 서비스 사이에서 데이터를 교환하고 다른 서비스가 소유한 데이터의 로컬 뷰를 유지하면 
>동기식 API 호출에 의존하지 않고도 독립성이 높아진다.

중요한 내용이다. 마이크로서비스 아키텍처에서는 서비스들 사이의 데이터 조회가 이러한 동기식 API에 의존하는 경우가 많은데, 이 부분에서 챌린지가 빈번하게 발생하고, 
장애로 이어지는 경우가 많다.

디비지움을 사용하는 일반적인 아키텍처 구성은 다음과 같다.

{:refdef: style="text-align: center;"}
![genral archtecture of debezum use](/assets/A475FB9A-ED60-4354-9222-20ED8FC25768.png)
{:refdef}

위 그림에서 DBZ XXX Kafka Connect에 해당하는 부분이 디비지움에 해당하는 부분으로, 데이터베이스의 데이터 변경을 포착하여 Kafka 에 이벤트를 푸시하고, 
특정 이벤트에 관심을 가지는 서비스들이 소비하는 구조이다.

특징 및 제약조건은 다음과 같다.

* MySQL 외에도 PostgreSql, Mongo 등을 지원한다.
* 스키마 변경, INSERT, UPDATE, DELETE 모두 변경 포착이 가능하다.
* MySQL Replication Slave 처럼 작동한다.
	* 디비지움을 재시작하면 마지막에 중단된 로그 포지션부터 데이터 변경 포착을 시작한다. 즉, 유실되지 않는다.
* MySQL 관련
	* log-bin 이 설정되어 있어야 한다.
	* MySQL Slave 에 log-bin 을 설정해놨을 경우 해당 Slave 에 붙어서 CDC 를 할 수 있다.
		* 이런 구성으로 CDC 를 할 경우 MySQL 서버 설정에 log_slave_update=1 설정이 반드시 필요하다. 그렇지 않을 경우 Replication 으로 인한 
		변경분은 binlog 에 남지 않게 된다.
	* binlog_format=row
	* binlog_row_image=full
		* MySQL 5.6 이상부터 지원한다.
		* MySQL 5.5 는 명시적인 내용이 없으나, full 로 봐도 무방하다. 오래된 오피셜 문서를 통해 확인한 내용이다.

디비지움은 기본적으로 5.5를 지원하지 않지만, 몇 줄의 소스코드 수정을 통해 5.5를 사용할 수는 있다.

이러한 구성이 가지는 장점은 명확하다.

* 이벤트 발생을 위한 추가적인 코드가 애플리케이션 서비스에 필요하지 않다. 즉, 능동적으로 데이터 변경을 포착할 수 있다.
* 리플리케이션 로그 포지션을 잃어버리지 않는 한 이벤트 누락이 발생하는 일은 없다.
* 다른 서비스에 대한 데이터베이스 의존성을 완화할 수 있다.

## MySQL 리플리케이션

우선 MySQL 리플리케이션을 간략히 살펴보도록 하자.

{:refdef: style="text-align: center;"}
![mysql-replication](/assets/mysql-replication.png)
{:refdef}

위의 구성에서는 1대의 마스터와, 3대의 슬레이브(리플리케이션 DB)로 구성되어 있다. 마스터에서 일어난 변경 사항이 binlog 에 기록이 되고 슬레이브는 
binlog 를 읽어 슬레이브에 적용하는 구조이다. 따라서 보통의 경우 말단에 위치한 슬레이브는 binlog 를 생성하지 않도록 설정하는 편이다.

{:refdef: style="text-align: center;"}
![mysql-replication](/assets/mysql-replication-with-debezium.png)
{:refdef}

그렇다면 디비지움 커넥터는 마스터를 바라보도록 하는 것이 좋을까? binlog 를 남기도록 한 슬레이브를 바라보도록 하는 것이 좋을까? 지금 설명하고 있는 CDC 시스템
에서는 쓰기 작업이 전혀 필요하지 않기 때문에, binlog 설정을 한 슬레이브 구성을 활성 1대, 예비 1대 정도로 추가하는 것이 좋다.  

## 카프카

카프카는 메시징 큐이다. 디스크에 메시지를 저장하며, 컨슘한 메시지는 삭제되지 않고 보존 기한만큼 보존된다. 프로듀싱한 메시지는 클러스터 내에 지정한 개수만큼의 
사본으로 저장되어, 서버 장애에도 고가용성을 달성할 수가 있다. 또한, 확장성과 처리 성능이 좋다. 모던 아키텍처에서는 리얼타임 이벤트 처리를 위해 카프카와 같은 
메시징 큐를 아키텍처에 포함하는 경우가 많다.

## 기본 개념들
 
카프카를 사용하면서 주로 알고 있어야 하는 개념은 다음과 같다. 복잡한 설명없이 핵심만 추려서 나열했다.

* 브로커
    * 하나의 카프카 서버를 의미한다.
    * 이런 브로커들을 묶어 카프카 클러스터를 만들게 된다.
    * 최소 3 대 이상이 무난하다.
    * 브로커를 쉽게 스케일아웃할 수 있다.
* 토픽
    * 토픽은 프로듀싱한 메시지가 담기는 통 쯤으로 생각하면 된다.
* 메시지
    * 메시지는 키와 값, 헤더로 구성된다.
* 파티션
    * 토픽 내부에서 메시지가 담기는 구획 구간이다.
    * 메시지가 어느 구획에 담길지는 메시지 키에 의해 결정되며, 동일한 메시지 키를 가진 메시지는 항상 같은 파티션으로 담긴다.
    * 키가 `NULL` 이라면 랜덤하게 배분된다.
    * 파티션의 수는 운영중에 늘릴 수 있지만 줄일 수는 없다. 초기에 파티션 수를 신중하게 결정하고, 운영중에 파티션 수를 늘려야 한다면 신중하게 결정해야 한다.
    * 읽기와 쓰기 작업은 리더 파티션에만 수행되며, 팔로워 파티션은 리더 파티션으로부터 복제만 하게 된다.
* 리플리케이션 팩터(RF)
    * 복제량으로, 클러스터 내에 몇 개의 사본이 존재하는지이다.
    * 이 값을 3 으로 한다면 카프카 클러스터내에 원본 메시지 포함 총 3 개의 메시지 사본이 존재하게 된다.
    * 이 값은 브로커 수 이하여야 한다. 2 ~ 3 정도로 설정하는 것이 무난하다. 
* 최소 사본
    * 메시지를 프로듀싱할 때 최소 몇 개의 사본이 존재할 때 프로듀싱을 성공한 것으로 보는지이다.
    * 프로듀서의 `acks` 설정이 -1(all) 인 경우에 유효하다.
    * 이 값은 RF 값 이하로 설정해야 프로듀싱 오류가 발생하지 않는다. 예상치 못한 브로커의 셧다운을 고려해서 RF + (-1 ~ -2) 정도로 설정하는 것이 좋다.
    * 브로커 설정 지정자는 `min.insync.replicas` 이며, 기본값은 1 이다.
* 프로듀서
    * 토픽에 메시지를 프로듀싱하는 주체이다. 
    * 펍섭(Pub-Sub) 모델에서 퍼블리셔(Publisher)에 해당한다.
    * `acks` 는 메시지를 프로듀싱할 때 메시지가 전달된 여부를 확인하는 수준이다. 비즈니스 요구사항에 맞게 선택하면 된다.
        * `0`: 리더가 제대로 메시지를 전달받았는지 확인하지 않는다. 그만큼 빠르지만 유실 가능성이 가장 크다.
        * `1`: 리더가 제대로 메시지를 전달받았는지 확인한다. `0` 보다는 느리지만 유실 가능성은 거의 없다. 하지만 리더가 메시지를 받은 다음 장애로 복구 
        불가한 상태가 되면 여전히 유실될 가능성이 존재한다.
        * `-1`: 리더 뿐만 아니라 팔로워에도 복제되었는지 확인한다. 가장 느리지만 유실 가능성은 없다.
* 컨슈머
    * 토픽에서 메시지를 컨슘하는 주체이다.
    * 하나의 컨슈머는 서로 다른 토픽 하나씩을 컨슘할 수 있다.
    * 펍섭(Pub-Sub) 모델에서 섭스크라이버(Subscriber)에 해당한다.
* 컨슈머 그룹
    * 컨슈머는 컨슈머 그룹으로 묶을 수 있다.
    * 같은 컨슈머 그룹에 속한 컨슈머는 코디네이터에 의해 각기 다른 파티션을 컨슘하게 된다. 즉, 하나의 파티션은 동일한 컨슈머 그룹에서 하나의 컨슈머에 의해서만 
    컨슘된다.

다음은 토픽과 파티션에 대한 이해를 돕기 위한 그림이다.

{:refdef: style="text-align: center;"}
![architecture](/assets/topic-partition.png)
{: refdef}

다음은 컨슈머 그룹의 컨슈머가 토픽내에 있는 파티션을 어떻게 컨슘하는지 이해를 돕기 위한 그림이다.

{:refdef: style="text-align: center;"}
![architecture](/assets/consumer-group.png)
{: refdef}

위의 그림에서 컨슈머 1 이 없어진다면 어떻게 될까? 그렇게 되면 컨슈머 0 이나 2 가 컨슈머 1 을 대신해서 파티션을 컨슘하게 된다. 다시 컨슈머 1 이 살아난다면 
컨슈머 0 과 2 가 컨슘하던 파티션중 일부를 컨슘하게 된다.

## 아키텍처

우리가 예제로 만들 작은 시스템의 아키텍처는 다음과 같다. 실제 프로덕션에서 사용하는 아키텍처도 이와 크게 다르지 않다. 기본 뼈대는 같고 단지 HA 구성을 위해 
Zookeeper, Kafka 가 더 추가되는 정도이다. 프로덕션 고려사항에 대해서는 다음 포스팅에서 살펴보도록 할 것이다.

{:refdef: style="text-align: center;"}
![architecture](/assets/simple-debezium-usage-architecture.png)
{:refdef}

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
* 카프카 클러스터
  * 단일 혹은 여러 브로커로 구성된 클러스터이다.
* Zookeeper
  * 분산 코디네이션 시스템으로 카프카 클러스터의 정보 관리, 리더 선출, 잠금 및 동기화를 위해 사용된다.
  * 카프카는 주키퍼 의존성을 조금씩 줄여나가고 있다.
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

## 주요 토픽

앞의 샘플 클러스터가 동작하는 과정에서 사용하는 중요 토픽들을 나열하면 다음과 같다.

* _schemas
    * 스카마 레지스트리에서 스키마 저장 용도로 사용한다.
* cdc_db1
    * 디비지움 스키마 변경 이벤트가 기록되는 토픽이다. 스키마 변경 이벤트는 이 토픽을 컨슘하면 된다.
* cdc_db1.sample.tb_user
    * 테이블에 대한 변경 이벤트가 기록되는 토픽이며, 행 단위로 이전 행 전체 데이터와 바뀐 행 전체 데이터가 함께 담긴다.
    * 테이블마다 이런 네이밍 규칙으로 생성된다.
* dbhistory.db1
    * 스키마 변경 이력을 기록하는 토픽이다.
* debezium_configs
    * 커넥터의 설정이 기록되는 토픽이다. 이 토픽의 이름은 설정으로 변경할 수 있다.
* debezium_offsets
    * 커넥터의 binlog 오프셋이 기록된다. 이 토픽의 이름은 설정으로 변경할 수 있다.
* debezium_statuses
    * 커넥터의 상태가 기록된다. 이 토픽의 이름은 설정으로 변경할 수 있다.

## 스키마 레지스트리를 사용하는 이유

아래와 같은 이유로 토픽에 데이터를 프로듀싱할 때 JSON 이 그렇게 적합하진 않다.

* JSON 은 타입 정보가 없으므로 정확한 크기의 데이터 처리가 힘든 면이 있다. 즉, 어떤 속성이 1 이라는 값을 가진다면 해당 속성이 Int 형을 담는지 Long 형을 
담는지는 JSON 을 역직렬화 하는 부분에서 결정해야만 한다.  
* 모든 데이터가 문자열로 표현되므로 직렬화한 데이터 크기가 큰 편이다.

스키마 레지스트리는 이러한 단점들을 보완한다. 스키마에 필드와 타입 정보를 기록하고, 직렬화나 역직렬화를 수행할 때 해당 스키마 정보를 참조하여 수행하게 된다. 이때
직렬화 포맷은 AVRO 를 사용하게 되며, 데이터 타입에 대한 정확한 처리가 가능하고, 바이너리 데이터 포맷이므로 직렬화된 데이터 크기를 크게 줄일 수 있다.

그리고, 스키마 버저닝이 되기 때문에, 데이터의 호환성 모드를 설정할 수 있다. 즉, A 라는 속성이 모델에서 어느 순간 사라진다면, 이 호환성 모드에 따라서 오류가 
발생할 수도 있고, 또는 오류가 발생하지 않도록 기본값을 채울 수도 있다는 의미이다.

## 언제 사용하는 것이 좋을까?

주로 다음과 같은 상황에서 유용할 것이라 생각된다.

* 실시간 이종 데이터베이스 동기화
    * 예를 들어, MySQL 에서 발생한 변경 데이터를 MongoDB 나 일래스틱 서치로 실시간 적용하는 것처럼 말이다. 
* 실시간 데이터 마이그레이션 및 동기화
    * 레거시 서비스를 개선하여 새로운 서비스를 만들었다고 가정해보자. 새로운 스키마가 만들어졌고, 서비스 중단 없이 기존 데이터를 마이그레이션하면서 레거시 
    서비스를 새로운 서비스로 전환해야 한다면 좋은 수단이 될 수 있다.  
* 실시간 데이터의 상태 변이 추적
    * 상태 변이를 준실시간으로 추적할 수 있다. 예를 들자면 배송상태 변경을 추적하는 것이 이에 해당할 수 있다.
* 실시간 데이터 정규화 및 역정규화
    * 데이터 변형과정에서 단순 변환 뿐만이 아니라 정규화 또는 역정규화 역시 수행할 수 있다.
    
물론, 배칭 처리로 작업을 유사하게 수행할 수 있지만, 배칭 처리는 다음과 같은 단점이 있다.

* 삭제에 대한 처리가 불가능하다.
* 변경을 포착하기 위해 기준 시간이 행 데이터에 존재해야만 

## 프로덕션 고려사항

프로덕션 환경에서는 장애가 발생했을 경우 어느 수준까지 장애가 용인이 되는지, 장애가 발생했을 경우 얼마나 빠르게 운영자(개발자)에게 전달이 되는지, 그리고 
얼마나 빠르게 복구할 수 있는지가 중요하다. 또한, 처리 성능 역시 잘 나와야 한다.

따라서, 시스템을 구축할 때 다음 사항을 고려해야 한다.

* 가용성
    * 카프카 브로커는 최소 3 대로 구성하는 것을 권장한다.
    * 스키마 레지스트리의 경우 앞단에 LB 가 필요하다면 넣고 스케일아웃 하도록 한다.
    * Zookeeper 는 최소 3 대로 구성하는 것을 권장한다.
* 성능
    * 모두 JVM 기반 환경이므로 서버 하드웨어 선택과 GC 설정등 JVM 성능 관련 파라미터들을 잘 조정해야 한다.
    * 필요하다면 서버의 커널 파라미터 설정도 할 필요가 있다.
    * 카프카는 꽤 좋은 성능의 하드웨어를 사용해야 한다.
* 경보
    * 모니터링 시스템이 있어야 하며, 로그나 지표에서 오류를 검출하여 담당자에게 이상 징후를 알릴 수 있어야 한다.

이 모든 것을 구성 및 관리하는 것은 쉬운 일이 아니다. 변경사항이 추적되지 않고, 빠르게 환경을 구성해 테스트해볼 수 없다면 장애 상황에서 검증이 어려워진다. 
따라서, 도커와 같은 컨테이너 환경을 적극적으로 사용하길 권장한다. 컨테이너는 한번 구성된 환경이 불변성을 가지기 때문에 항상 같은 상태가 유지된다는 장점이 있다.  
영속화가 필요한 데이터는 볼륨을 마운트해서 영속화하면 된다. 그리고, 개발, 스테이징, 프로덕션 환경 모두를 거의 동일하게 구성할 수 있으며, 쉘 스크립트, yaml 과 
같이 텍스트 기반 설정으로 환경을 구성하기 때문에 git 과 같은 VCS 로 이력을 추적하기가 용이하다.

프로덕션 수준의 카프카 클러스터와 모니터링 시스템을 구성하는 내용은 별도의 포스팅을 통해 살펴 볼 것이다.

## 스냅샷

이전 포스팅에서도 설명했지만, 디비지움은 최초에 한번 스냅샷이라는 작업을 수행한다. 이 스냅샷이라는 작업은 원본 테이블의 데이터를 모두 읽어 토픽에 행 단위로 프로듀싱
하는 작업이다. 수행속도는 나쁘지 않다. 스냅샷 로직 내부에서 데이터 건수에 따라 건수가 적으면 페이징 방식으로 읽어 처리하고, 많을 경우 커서 방식으로 읽어 처리한다.
이처럼 기본으로 제공하는 기능을 사용할 수도 있고, ETL 애플리케이션을 별도로 만들어 스냅샷을 수동으로 수행할 수도 있다.

ETL 애플리케이션을 통해 수동으로 스냅샷을 수행하는 경우라면 디비지움의 테이블 토픽에 CDC 이벤트가 계속해서 기록되므로, ETL 애플리케이션을 모두 수행한 다음에 
이러한 CDC 이벤트를 컨슘하는 것이 좋다. 이벤츄얼 컨시스턴시(Eventual consistency) 개념으로, 변경 데이터가 순차적으로 반영되면 어느 시점이 되어 
데이터가 원본과 같아지게 된다. 컨슘을 하면서 ETL 을 함께 수행한다면 데이터 일관성을 보장할 수 없다.

어느 방식이 더 낫다고 단정 짓기 힘들지만, 유연성을 따져보면 ETL 애플리케이션을 별도 작성하는 것이 더 나아보인다. 즉, 별도로 작성하면 중간에 어떤 문제가 발생해도
다른 애플리케이션에 영향을 주지 않고 개별 대처가 훨씬 쉽다는 것이다. 이렇게 해서 다른 MySQL 데이터베이스로 3시간 동안 1억 4천만 건을 스냅샷 수행한 경험이 있다.
이 1억 4천만건은 단순히 한 테이블에서 다른 테이블로의 동일한 데이터 이동이 아니라 역정규화가 이뤄진 형태의 스냅샷이었다. 물론 이렇게 하면 컨슈머 애플리케이션과
ETL 애플리케이션 사이의 코드 중복이 어느정도 발생하게 되는건 사실이다.

{:refdef: style="text-align: center;"}
![architecture](/assets/snapshot-with-etl.png)
{:refdef}

스냅샷 과정은 초기에 한번 수행하는 프로세스이지만, 원본과 대상의 데이터 일관성을 맞추는 굉장히 중요한 과정이기도 하다. 테스트해보고, 고민해보고 운영 방식에 
걸맞는 방식을 선택하는 것이 좋다.

## 원본 데이터베이스 변경이 필요한 경우

운영을 하다보면 바라보는 원본 데이터베이스를 변경해야 하는 경우가 종종 생기게 된다. 보통 사유는 다음과 같다.

1. 하드웨어 증설
2. 대량의 데이터가 존재하는 테이블에 컬럼을 추가하거나 변경하는 스키마 변경
3. 복구 불가능한 데이터베이스 장애

보통 위와 같은 상황에서는 대상이 되는 데이터베이스 서버를 서비스에서 제외한 다음 대응하게 된다. 데이터베이스를 오랫동안 사용할 수 없는 상황이 되기 때문에 
디비지움 커넥터를 예비 데이터베이스로 연결을 해줘야 한다. 문제는 여기에서 생기는데 단순히 커넥터 설정에 있는 데이터베이스 설정만 바꿔서는 안된다. 각각의 
데이터베이스가 만들어내는 binlog 는 이름도 다르고 포지션도 다르기 때문에 문제가 발생하게 된다.

1번과 2번의 경우에는 활성 장비와 예비 장비가 모두 살아 있는 상태에서 사전에 대응하므로 커넥터를 아예 새로 만들고, 기존 커넥터를 폐기하는 방식으로 대응하면 된다. 

커넥터 구성을 새로 구성하는 방법은 다음과 같다. 다음과 같은 원본 커넥터 설정 `mysql-connect-db1.json` 이 있다고 가정해보자.

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

다음과 같이 변경한 다음 설정파일을 저장한다.

```shell
{
        "name": "mysql-connector-db1-2020100700",
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
                "database.history.kafka.topic": "dbhistory.db1.2020100700",
                "include.schema.changes": "true",
                "table.whitelist": "sample.tb_user"
        }
}
```

그런다음 다음과 같이 새로운 디비지움 커넥터를 생성한다.

```shell
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" \
  http://127.0.0.1:8083/connectors/ -d @mysql-connect-db1.json
```

그리고, 데이터가 잘 인입되는지 확인 후 기존 커넥터를 삭제하면 된다. 

```shell
curl -i -X DELETE -H "Accept:application/json" -H  "Content-Type:application/json" \
  http://127.0.0.1:8083/connectors/mysql-connector-db1
```

이렇게 하게 될 경우 문제점은 중복 프로듀싱이다. 따라서 메시지가 중복 프로듀싱 되더라도 문제없이 처리되도록 설계하는 것이 좋다.

반면, 3번의 경우에는 복구 불가능한 장애 상황에서 예비로 변경하는 것이기 때문에 활성 장비에서 기록된 binlog 포지션이 예비 장비의 binlog 포지션 어느 부분에 
해당하는지 확인한다음 binlog 포지션을 변경해줘야 한다. 다소 번거러운 작업이다. 하지만 binlog 포지션만 정확히 찾을 수 있다면 데이터 유실은 걱정할 
필요가 없다.

빈로그 포지션을 변경하는 방법은 다음과 같다. 먼저 디비지움 커넥터의 마지막 포지션이 담긴 파티션의 내용을 확인해야 한다.

```
sudo docker run -it --rm edenhill/kafkacat:1.5.0 -b {Server list. e.g. kafka-1:9092} -C -t \
    {Name of the offset topic} -f 'Partition(%p) %k %s\n'
```

커넥터 이름을 확인한 다음 제일 마지막 오프셋 내용을 통째로 복사한다. 내용은 다음과 유사할 것이다.

```shell
Partition(10) ["mysql-connector-test-db",{"server":"cdc_test_db"}] {"ts_sec":1600149140,"file":"test-bin.000638","pos":8579523,"row":1,"server_id":1001241,"event":2}
```

그런 다음 `mysqlbinlog` 를 이용해 원격 서버의 binlog 포지션을 조사해야 한다.

```shell
sudo docker run -it --rm mysql mysqlbinlog -h{Server host} -u{User ID} -p --read-from-remote-server \
    --start-position=8579523 --stop-position=8679523 test-bin.000638
```

변경할 적당한 포지션을 찾았다면 해당 포지션으로 건너뛸 수 있도록 다음과 같이 토픽에 내용을 프로듀싱해주면 된다. 만약 858952 포지션으로 강제 적용해야 한다면 다음과
같이 할 수 있다.

```shell
echo '["mysql-connector-test-db",{"server":"cdc_test_db"}]|{"ts_sec":1600149140,"file":"test-bin.000638","pos":858952,"row":1,"server_id":1001241,"event":2}' | \
docker run -i -a stdin --rm edenhill/kafkacat:1.5.0 -P -b {Server list. e.g. kafka-1:9092} -t {Name of the offset topic} \
-K \| -p {Partition number. 10 in this case.}
```

파이프를 사용하기 때문에 sudo 권한이 필요하다면 `sudo su` 로 권한을 상승시켜 실행하거나 스크립트로 만들어 실행하기 바란다.

프로덕션용 데이터베이스는 아예 활성 장비와 예비 장비 두 대를 한 번에 준비하는 것이 좋다. 준비라고 해봤자 권한 추가하는 것과 이력을 관리하는 것 밖에 없으므로 
큰 노력이 들지 않는다.

## 디비지움 SQL 파서 오류

디비지움은 앤틀러(Antlr)를 이용해 SQL 파서를 직접 만들었다고 한다. 기존의 파서가 MySQL 버전마다의 다양한 DDL 을 제대로 파싱하지 못한 것이 주요 이유였는데, 
현재의 파서도 그런 문제가 있는 것으로 보인다. 하지만 이 부분은 확실치 않다. 어쩌면 5.5 버전의 DDL 을 파싱하지 못하는 것이 아닌가 추측이 들기도 한다. 
디비지움에서 공식으로 지원하는 버전은 5.7 부터이기 때문이다. 본인은 필요에 의해 5.5 버전을 지원할 수 있도록 살짝 패치를 해서 사용하고 있는데, 공교롭게도 
5.5 버전의 데이터베이스에서 수행한 DDL 이 장애를 만들었다.

아무튼 이런 상황이면, 커넥터에서 오류가 발생한 데이터베이스 스레드만 중단된다. 아무리 재시작을 해도 정상화되지 않으므로, 마지막 binlog 포지션을 기점으로 
오류가 발생하는 지점의 포지션을 확인해야 한다. 그런 다음 필요하다면 해당 포지션을 건너뛸 수 있도록 만들어 줘야 한다. 단순히 포지션을 변경하는 수준이라면 아주
쉽게 처리할 수 있다.

## 도커 컨테이너 로깅

로깅 드라이버가 설정되지 않은 경우 기본적으로 `json-file` 로깅 드라이버를 사용한다. 호스트 파일 시스템에 계속해서 기록되며, `docker logs` 커맨드를 통해
로그를 조회할 수 있다.

아래와 같이 몇 가지 기본 드라이버가 제공되며, 직접 구현할 수도 있다.

* syslog
* gelf
* fluentd
* awslogs
* splunk
* etwlogs
* gcplogs
* Logentries
* local
* json-file
* journald

이 중에서 도커 커뮤니티 에디션에서 `docker logs` 커맨드로 로그를 조회할 수 있는 드라이버는 `local`, `json-file`, `journald` 뿐이다. 커뮤니티 에디션을 
사용하면 로그를 확인하는 부분이 불편할 것 같지만 사실 전혀 불편하지 않다. 컨테이너를 구동하는 핵심 기능과 성능 면에서 커뮤니티 에디션과 엔터프라이즈 에디션은 큰 차이가 
없으니 참고하기 바란다.

여러 호스트 서버에 접속해서 일일이 로그를 확인하기는 번거롭기 때문에 로그를 중앙으로 수집할 필요가 있다. 이때 일반적으로 사용할 수 있는 드라이버가 `fluentd`,
 `splunk` 가 될 수 있다. 이 포스팅에서는 여러 책에서 많이 언급된 `fluentd` 로깅 드라이버를 사용해서 살펴보기로 한다.
 
## 로그 수집

fluentd 데몬을 호스트 머신에 각각 구성하고, 일래스틱서치로 로그를 전달하는 방식을 사용했다.

{:refdef: style="text-align: center;"}
![collect-log](/assets/fluentd-es-kibana.png)
{:refdef}

로그를 수집하길 원하느 서비스 컨테이너의 docker-compose.yml 파일에는 서비스 속성에 다음과 추가하면 된다. 그리고 로깅드라이버 설정과는 별도로 애플리케이션 
로그는 볼륨 마운트를 통해 호스트 볼륨에 퍼시스턴스하기 바란다.

```shell
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: log.zk-1
```

일래스틱서치로 로그를 전달하려면 fluentd 컨테이너에 fluentd 플러그인을 설치해야한다. 다음과 같이 `Dockerfile` 을 만들어 도커 이미지를 만들어 놓을 수 있다.

```shell
# fluentd/Dockerfile
FROM fluent/fluentd:v1.6-debian-1
USER root
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-document", "--version", "3.5.2"]
USER fluent
```

사설 컨테이너 레지스트리가 있다면 그것을 사용하도록 한다. 다음은 레지스트리에 푸시된 `test/fluentd-es-plug:v1.6-debian-1` 이라는 이미지를 사용하여 
`docker-compose.yml` 컴포즈 파일을 구성한 모습이다. 이 `fuentd` 컨테이너는 호스트당 하나만 띄우면 된다.

```shell
version: '3'
services:

  fluentd:
    image: test/fluentd-es-plug:v1.6-debian-1
    ports:
      - 24224:24224
      - 24224:24224/udp
    environment:
      - TZ=Asia/Seoul
    extra_hosts:
      - 'elasticsearch:172.26.0.100'
    volumes:
      - ./fluentd/conf:/fluentd/etc:ro
```

`extra_hosts` 속성에 있는 일래스틱서치 호스트 IP 를 적절하게 변경하기 바란다. 그리고 `fluentd.conf` 설정파일을 다음과 같이 생성한다.

```shell
# fluentd/conf/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    user elastic
    password 1234
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name container_log
    tag_key @log_name
    reconnect_on_error true
    reload_on_failure true
    reload_connections false
    flush_interval 1s
  </store>
  <store>
    @type stdout
  </store>
</match>
```

일래스틱서치의 `docker-compose.yml` 구성은 다음과 같다.

```shell
version: '3'
services:

  elasticsearch:
    image: elasticsearch:7.6.1
    ports:
      - 9200:9200
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xmx1G
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=1234
      - TZ=Asia/Seoul
    ulimits:
      memlock:
        soft: -1
        hard: -1
#    volumes:
#      - ./volume/es/data:/usr/share/elasticsearch/data

  kibana:
    image: kibana:7.2.0
    environment:
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=1234
      - TZ=Asia/Seoul
```

키바나를 통해 수집된 로그를 확인할 수 있다.

이로써 로그 수집 부분은 완료되었으며, 지표 수집 부분을 살펴보도록 하자.

## 지표 수집

지표 수집은 다음과 같이 두 가지로 나눌 수 있다. 이 지표는 CPU 사용량, 메모리 사용량, 네트워크 RX 및 TX 등의 중요 지표를 포함한다.

* 컨테이너 지표 수집
* 호스트 지표 수집

컨테이너 지표 수집에는 `cAdvisor` 를 사용하고, 호스트 지표 수집은 `node-exporter` 를 사용한다. 대략적인 구성도는 다음과 같다.

{:refdef: style="text-align: center;"}
![collect-metric](/assets/cadvisor-node-exporter-prometheus-grafana.png)
{:refdef}

`fluentd` 와 마찬가지로 호스트당 컨테이너 하나를 띄우면 된다.

```shell
version: '3'
services:

  cadvisor:
    image: gcr.io/google-containers/cadvisor:v0.36.0
    ports:
      - 8180:8080
    user: 0:0
    privileged: true
    environment:
      - TZ=Asia/Seoul
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro

  node-exporter:
    image: prom/node-exporter:v0.18.1
    ports:
      - 9100:9100
```

프로메테우스와 그라파나가 구성된 `docker-compose.yml` 파일은 다음과 같이 구성하면 된다.

```shell
version: '3'
services:

  grafana:
    image: grafana/grafana:6.7.1
    ports:
      - 3000:3000
    environment:
      - TZ=Asia/Seoul
#    volumes:
#      - ./volume/grafana/data:/var/lib/grafana
#      - ./volume/grafana/logs:/var/log/grafana
#      - ./volume/grafana/conf:/etc/grafana

  prometheus:
    image: prom/prometheus:v2.16.0
    command:
      - --config.file=/etc/prometheus/prometheus.yaml
    environment:
      - TZ=Asia/Seoul
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yaml:ro
```

`prometheus.yaml` 설정파일은 다음과 같이 생성하기 바란다. `targets` 에 지표수집 대상이 되는 호스트 목록을 추가하면 된다.

```shell
scrape_configs:
  - job_name: monitor
    scrape_interval: 5s
    static_configs:
      - targets:
          # service
          - 172.26.0.100:8180
          - 172.26.0.101:9100
```

모두 정상적으로 구성되었다면, 그라파나를 통해 지표 모니터링을 시작할 수 있다.

## 마무리

MySQL 데이터베이스를 사용하고 위에서 언급한 내용들에 대해 고민해본적이 있다면 추천하는 설루션이다. 실제로 프로덕션에 적용해서 서비스의 비효율적인 부분들을 
많이 개선한 경험이 있다. 디비지움 프로젝트도 현재 MySQL에 대해서는 성숙도가 높다.

## 참고

* [이벤트 소싱](https://www.confluent.io/blog/event-sourcing-vs-derivative-event-sourcing-explained/)
* [CDC 솔루션들](https://github.com/wushujames/mysql-cdc-projects/wiki)
* [if kakao 2019](https://mk.kakaocdn.net/dn/if-kakao/conf2019/%EB%B0%9C%ED%91%9C%EC%9E%90%EB%A3%8C_2019/T03-S01.pdf)
* [Debezium Connector for MySQL](https://debezium.io/documentation/reference/1.0/connectors/mysql.html)
* [MySQL Replication Options](https://dev.mysql.com/doc/mysql-replication-excerpt/5.5/en/replication-options-slave.html)
* [샘플 클러스터 + 애플리케이션](https://github.com/m0rph2us/docker-debezium)
* [공식 튜토리얼](https://debezium.io/documentation/reference/1.2/tutorial.html)
* [MySQL 커넥터 공식 문서](https://debezium.io/documentation/reference/1.2/connectors/mysql.html)
