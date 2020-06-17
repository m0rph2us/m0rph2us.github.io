---
layout: post
excerpt_separator: <!--more-->
title: Using MySQL Docker For Development
categories: mysql docker
---

# MySQL 도커를 활용하여 개발하기
## 개요

이 글에서는 MySQL 도커를 활용하여 개발하는 방식에 대해 알아볼 것이다. 데이터베이스를 활용한 개발에서 컨테이너는 그 진가를 크게 발휘한다. 프로덕션에서 발생하는 장애 요인 중 데이터베이스 관련한 장애가 꽤 큰 비중을 차지하기 때문에 개발 환경을 프로덕션 환경과 최대한 동일한 환경으로 구성하는 것이 좋다.
<!--more-->

## 데이터베이스 환경이 중요한 이유

경험적으로 보면 개발, 스테이징, 프로덕션 환경마다의 차이로 인해 발생하는 프로그램 오류가 적지 않다. 이런 상황에서는 흔히 *"내 컴퓨터에서는 잘 되는데?"* 라는 말을 듣거나(하곤) 한다. 이러한 환경 불일치 중에서 제법 높은 비중을 차지하는 것이 바로 데이터베이스 환경 불일치일 것이다.

보통, 프로덕션 데이터베이스는(MySQL의 경우) 마스터 서버 한 대와 여러 대의 슬레이브 서버들로 구성되어 있는 경우가 많다. 이러한 구성에서 마스터는 쓰기에 좀 더 집중된 용도로 사용하고, 슬레이브는 읽기 전용으로 사용한다. 하지만 우리는 개발, 스테이징이 마스터 한 대만으로 구성된 경우를 종종 경험할 수 있다. 이러한 환경 불일치는 개발, 스테이징까지는 잘 되다가 프로덕션에 배포했을 때, 읽혀야 할 값이 읽히지 않거나, 쓰여야 할 값이 쓰이지 않거나 등의 문제를 일으키게 된다.

그리고, 유닛테스트의 경우 데이터베이스와 연계테스트(integration test)가 반드시 필요하다. 동일한 MySQL 버전으로 연계테스트를 하되, H2DB의 MySQL 시뮬레이션 기능으로 연계테스트를 해서는 안된다. 왜냐하면 말 그대로 시뮬레이션일 뿐 MySQL 특유의 특성과 버전별로 가지고 있는 미묘한 특성까지는 시뮬레이션하지 못하기 때문이다. 이러한 특성들을 조기에 발견하지 못하면 프로덕션에서 장애를 맞이할 수도 있다.

## 요즘은 컨테이너가 대세

요즘은 말 그대로 컨테이너가 대세다. 앞서 얘기한 환경 불일치를 아주 깔끔하게 해소할 수 있으며, 거기에 보너스 효과까지 기대할 수 있다. 

일단 컨테이너로 개발 환경의 데이터베이스를 구성하면 다음과 같은 효과를 기대할 수 있다.

* 앞서 얘기한 마스터, 슬레이브 구조의 데이터베이스를 비교적 비용 효율적으로 쉽게 구성할 수 있다.
* 개발 데이터베이스는 중앙 위치에 있는 서버가 아니라 각 개발자의 컴퓨터에서 컨테이너로 띄워진다. 따라서 MySQL 데이터베이스의 통제권을 개발자가 가지게 된다.
  * 다양한 버전의 MySQL을 쉽게 스위칭할 수 있다.
  * 특정 테이블을 제거하거나, 칼럼 제거, 이름을 변경하기가 굉장히 용이하다.
  * 별의별 시도를 해볼 수 있다.
* 데이터를 깨끗하게 지워도 어떤 잠재적인 문제를 야기하지 않기 때문에 유닛테스트를 통한 연계테스트가 수월하다.
* VCS를 통해 모든 변경 내용을 버전 관리할 수 있다.

본격적으로 살펴보기 전에 오해 요소를 한 가지 짚고 넘어가려 한다. 대부분의 경우 프로덕션 환경은 굉장히 좋은 하드웨어를 사용하고, 성능을 최대로 뽑아내기 위해 VM, 컨테이너를 잘 사용하지 않는다. 즉, 우리가 지금 살펴보려고 하는 내용은 프로덕션과 동일한 환경을 갖춰 프로그램 오류 발생을 줄이기 위함이 목적이지 데이터베이스 환경 자체를 배포하기 위함이 아니다. 이 점을 확실히 인지하고 다음으로 넘어가도록 하자.

## MySQL을 띄우는 방법
### docker

다소 번거롭지만 다음처럼 순수 docker 커맨드를 사용하여 띄울 수 있다.

```
docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag
```

커스텀 네트워크, 환경 변수, 볼륨이 추가되는 등 여러 옵션들이 추가되면 사용성 및 가독성이 현저히 저하된다. 컨테이너를 기동하고 종료하는 스크립트를 만들어 사용할 수도 있겠지만, docker-compose 라는 커맨드라인 도구가 있기 때문에 굳이 그럴 필요가 없다.

### docker-compose

yaml로 컨테이너를 정의하여 띄우는 방식이기 때문에 docker 커맨드로 옵션을 일일이 지정하는 방식보다 그나마 사용성과 가독성이 뛰어나다. 그리고, 하나의 yaml에 여러 서비스를 함께 정의할 수 있으며, 이때 해당 서비스들이 같은 네트워크에 자동으로 묶이기 때문에 편리한 측면이 있다. 또한 기동과 중지가 간편하다. 개발을 하다보면 docker 커맨드는 주로 유틸리티 목적으로 사용하고, docker-compose는 서비스를 정의하고 실행하는 데 사용하는 편이다.

```
docker-compose up
```

## docker-compose를 이용한 구성 살펴보기 

단순히 MySQL 서버만 띄워서 개발 목적으로 사용하는 것은 무리가 있다. 왜냐하면 프로덕션 수준의 중요한 서버 설정들(e.g. 트랜잭션 격리 수준, 리플리케이션 등)의 설정 적용이 필요하고, 필요한 데이터베이스 스키마도 구성해야 하며, 때에 따라서는 기본 데이터가 필요할 수도 있기 때문이다.

이를 위해 본인이 미리 만들어둔 docker-compose 구성이 있으므로 해당 구성을 천천히 살펴보도록 하자. [여기](https://github.com/m0rph2us/docker-mysql)에서 클론 받을 수 있다.

먼저 디렉터리 구조는 다음과 같다.

```shell
.
├── README.md
├── docker-compose.yml
├── conf
│   ├── master
│   │   └── my-ext.cnf
│   └── slave
│       └── my-ext.cnf
├── init
│   ├── db
│   │   └── sample
│   │       ├── init-data.sql
│   │       └── init.sql
│   ├── mysql
│   │   ├── master
│   │   │   └── init.sql
│   │   └── slave
│   │       └── init.sql
│   └── seeding.sh
└── tool
    ├── tail-general-log.sh
    └── tail-slow-log.sh
```

각 파일 및 디렉터리는 다음과 같은 내용을 포함한다.

* docker-compose.yml
  * master-slave 구조의 MySQL 서비스 정의
* conf
  * master, slave 서버 각각의 서버 설정
* init/db
  * 데이터베이스 초기화 SQL 스크립트
* init/mysql
  * MySQL 초기화 SQL 스크립트
* init/seeding.sh
  * 데이터베이스 초기화 쉘 스크립트
* tool
  * 제너럴, 슬로우 로그 확인용 스크립트

docker-compose.yml 파일을 열어보자.

```yaml
version: '3'
services:
  mysql-master:
    image: mysql:5.7.22
    volumes:
      - ./conf/master:/etc/mysql/conf.d
      - ./init/mysql/master:/init-mysql
      - ./init/db:/init-db
      - ./init/seeding.sh:/docker-entrypoint-initdb.d/seeding.sh
      #- ./volume/mysql-master:/var/lib/mysql
    environment:
      #- TZ=Asia/Seoul
      - MYSQL_ROOT_PASSWORD=1234
    ports:
      - '33060:3306'

  mysql-slave:
    image: mysql:5.7.22
    volumes:
      - ./conf/slave:/etc/mysql/conf.d
      - ./init/mysql/slave:/init-mysql
      - ./init/db:/init-db
      - ./init/seeding.sh:/docker-entrypoint-initdb.d/seeding.sh
      #- ./volume/mysql-slave:/var/lib/mysql
    depends_on:
      - mysql-master
    environment:
      #- TZ=Asia/Seoul
      - MYSQL_ROOT_PASSWORD=1234
    ports:
      - '33070:3306
```

정의는 굉장히 직관적이기 때문에 따로 설명은 하지 않겠다. 단지 이렇게 정의하고 `docker-compose up` 을 수행하게 되면 작업 디렉터리에서 docker-compose.yaml 파일을 찾아 기동하게 된다. `-f` 옵션을 사용하면 `docker-compose -f some.yaml up` 처럼 yaml 파일을 별도로 지정할 수도 있다.

## 진실의 원천(Source of Truth)

사실 MySQL 도커를 사용하고자 했던 주된 이유 중의 하나가 바로 테이블 설계 때문이었다.

기존의 방식을 되짚어 보면 프로젝트가 시작되고 누군가 요구사항을 분석하여 초안을 워드나 위키 문서로 만든다. 대부분 표를 그려서 한땀한땀 적어넣는다. 그리고 정성스러운? 리뷰 과정을 거쳐서 문서가 완성되면 개발, 스테이징에 차례로 반영하게 된다. 그리고, 개발 과정에서 수정 사항이 나오면 기록하거나 기록하지 않은채 개발, 스테이징에 반영하는 사이클이 반복되는 구조다. 정리된 문서 기준으로 피드백을 받다보니 팀원들의 리뷰 의무와 전파력이 약한 문제가 있으며, 심지어 누가 언제 어떤 일감 때문에 추가하고 삭제했는지 이력 파악이 힘들다는 단점이 있다. 심지어 프로덕션 데이터베이스에 반영할 DDL이 누락되는 케이스도 종종 볼 수 있다. 

하지만, 앞에서와 같이 컨테이너로 데이터베이스 환경을 구성하고, git과 같은 VCS로 버전관리를 하게 되면, 새로운 기능을 개발할 때마다 브랜치를 만들어 작업을 진행할 수 있게 된다. git으로 관리할 수 있다는 것은 풀리퀘스트와 같은 프로세스를 태울 수 있음을 의미한다. 즉, 더 이상 위키나 워드 문서에 표를 만들어 정리할 필요가 없다는 얘기이고, 모든 변경이력 또한 추적할 수 있음을 의미한다.

## 유닛테스트

MySQL 컨테이너를 큰 힘 들이지 않고 항상 새것처럼 빠르게 띄울 수 있기 때문에 유닛테스트에 사용할 때도 이점이 있다. H2DB와 같은 데이터베이스는 MySQL 시뮬레이션을 지원하지만, 말 그대로 시뮬레이션이기 때문에, MySQL 만이 가지는 특성 및 버전마다 존재하는 특성까지 시뮬레이션하기를 기대하기는 어렵다. 즉, 제대로 연계테스트를 하려면 MySQL에 직접해야 된다는 것이다.

## 두 유 노우 flyway?

여담이지만, flyway 라는 데이터베이스 버전컨트롤 툴?을 사용하는 경우를 간혹 볼 수 있는데, 테이블 하나가 수백만 로우의 데이터를 가지는 수준의 서비스라면, 사용을 진지하게 고민해보라고 말하고 싶다. 사용하는 데이터베이스에 따라서 테이블 락이 걸리거나 리플리케이션 지연이 발생할 수도 있기 때문에, 이러한 마이그레이션 툴에 의해서 온라인 DDL이 수행되는 경우 굉장히 주의해야 한다. 특히 MySQL은 버전에 따라 온라인 DDL 수행시 고려할 사항이 더 많은 편이다. 

## 마무리

이 글에서는 개발을 진행할 때 MySQL 도커를 어떻게 활용할 수 있는지 살펴보았다. 비단 MySQL 뿐만이 아니다. Kafka, Redis 등도 마찬가지로 응용할 수가 있다. 데이터베이스라는 특성상 DDL이 존재하기 때문에 활용성이 더 클 뿐이다. 

## 참고

1. [도커 시작하기](https://m0rph2us.github.io/docker/2020/06/12/docker-basics.html)
1. [스프링에서 여러 데이터베이스 연결하기](https://github.com/m0rph2us/multiple-datasource-spring-boot)
1. [Docker For MySQL](https://github.com/m0rph2us/docker-mysql)
1. [Overview of Docker Compose](https://docs.docker.com/compose/)
