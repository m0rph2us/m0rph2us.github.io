---
layout: post
excerpt_separator: <!--more-->
title: Development Using Docker
categories: mysql cdc debezium
---

# 도커(Docker)를 활용한 개발
## 개요

요즘의 개발, 인프라 구성은 컨테이너 기술을 활용하는 것이 대세가 되어가는 추세다. 이 글에서는 컨테이너의 개념을 간략히 설명하고, 개발 및 인프라 구성을 할 때 어떻게 활용할 수 있는지를 설명하려고 한다.
<!--more-->

## 컨테이너 vs VM

먼저 컨테이너와 VM의 차이를 짚고 넘어갈 필요가 있다. 이 두 가지가 동작하는 방식은 서로 완전히 다르다.

{:refdef: style="text-align: center;"}
![architecture](/assets/containers-101-2x.png)
{:refdef}

위 그림에서는 왼쪽이 VM, 오른쪽이 컨테이너를 보여주고 있다.

VM은 Virtual Machine, 즉, 가상의 컴퓨터다. 이렇게 가상의 컴퓨터를 구동해 주는 것을 하이퍼바이저(hypervisor)라고 하며, 이러한 하이퍼바이저에는 여러분도 익숙한 VMWare, Oracle VirtualBox, QEMU, KVM 등이 있다. 가상의 컴퓨터이기 때문에 OS를 직접 만들어 올릴 수도 있고, 윈도우, 리눅스, DOS 등 가리지 않고 실제 컴퓨터로 할 수 있는 거의 모든 것을 할 수 있다. 호스트 OS가 무엇이든 상관없이 하이퍼바이저가 지원만 하면 되기 때문에 활용성이 굉장히 높다. 하지만, OS 위에 또 다른 OS가 올라가는 형태이기 때문에 호스트 OS의 리소스를 많이 사용할 수 밖에 없고, 실제 하드웨어 액세스는 게스트 OS <-> 하이퍼바이저 <-> 호스트 OS의 형태로 이루어지기 때문에 매우 느리다.

컨테이너를 보여주는 오른쪽 그림을 유심히 보면 게스트OS와 하이퍼바이저 자리에 컨테이너 런타임이 위치한다. 이러한 컨테이너 런타임은 호스트 OS 커널에서 컨테이너를 직접 구동하기 때문에 VM에 비해 훨씬 빠르고, 메모리를 적게 차지한다. 널리 사용되는 컨테이너 런타임에는 도커, rkt 등이 있다. VM만큼의 리소스 격리성은 아니지만, 여전히 높은 수준의 리소스 격리성을 제공한다. 도커는 LXC(Linux Container)기반이기 때문에 호스트 OS가 리눅스 커널이어야만 한다. 맥이나 윈도우는 작은 Linux VM이 설치되고 그 위에서 컨테이너를 구동한다.

컨테이너가 제공하는 높은 수준의 격리성, 빠른 애플리케이션 패키징 및 구동 속도는 최근의 소프트웨어 개발 및 배포 라이프사이클에 큰 영향을 주고 있다.

## 도커

도커는 가장 대중화된 컨테이너 런타임이다. 이것 저것 알아야 할 용어들이 많지만, 핵심적인 것을 나열해 보면 다음과 같다.

* 도커 이미지
  * 도커 컨테이너 런타임으로 구동할 수 있는 이미지이다. 이러한 이미지는 바닥부터 직접 만들 수도 있고, 누군가 이미 만들어둔 이미지를 이용하여 확장할 수도 있다.
* 레지스트리
  * 도커 이미지 저장소 정도로 생각하면 된다. 퍼블릭 레지스트리로는 도커허브([Docker Hub](https://hub.docker.com/))가 있다.
* 도커 커맨드
  * 도커 이미지를 레지스트리에서 가져오거나 푸시하고, 구동 및 관리하는 커맨드이다.
* 도커 컴포즈
  * 도커 커맨드는 옵션이 굉장히 많다. 일일이 입력하여 사용하기에는 어려움이 있을 수 있기 때문에 관리 편의를 위해 yaml로 정의하여 사용하는 것이 좋다. 스웜이라는 컨테이너 오케스트레이션 도구가 존재하지만 두 포맷의 호환이 매끄럽게 되지 않는 문제가 있어 둘 중 하나를 선택해야 하는 문제가 생긴다. 본인은 도커 컴포즈를 더 선호한다.

이정도면 됐고, 기본 사용법을 알아보도록 하자.

## 기본 사용법

도커가 설치되어 있지 않다면 먼저 설치해야 한다. 설치 방법은 [여기](https://docs.docker.com/engine/install/)를 참고하기 바란다. 굉장히 다양한 도커 커맨드가 존재하지만, 다음 다섯 가지가 가장 기본적으로 많이 사용되는 커맨드라고 보면 된다.

### ps

실행중인 컨테이너의 목록을 조회한다.

```
docker ps
```

중단된 컨테이너 목록까지 보고 싶은 경우 `-a` 옵션을 함께 준다.

### stop

실행중인 하나 혹은 여러 컨테이너를 중단한다.

```
"docker stop" requires at least 1 argument.
See 'docker stop --help'.

Usage:  docker stop [OPTIONS] CONTAINER [CONTAINER...]

Stop one or more running containers
```

### rm

하나 혹은 여러 컨테이너를 제거한다.

```
"docker rm" requires at least 1 argument.
See 'docker rm --help'.

Usage:  docker rm [OPTIONS] CONTAINER [CONTAINER...]

Remove one or more containers
``` 

### run

run은 새로운 컨테이너를 생성하고 [COMMAND]를 실행한다는 것을 알 수 있다.

```
"docker run" requires at least 1 argument.
See 'docker run --help'.

Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container
```

예를 들어 다음은 mysql 커맨드를 사용하여 원격 서버에 접속한다. [COMMAND]가 종료되면 컨테이너도 함께 종료된다.

```
docker run -it --rm mysql mysql -h192.168.1.11 -P3306 -uroot  -p
```

데몬으로 실행하고 싶다면 `-d` 옵션을 함께 줄 수 있다.

### exec

exec는 이미 실행중인 컨테이너에서 COMMAND를 실행한다.

```
"docker exec" requires at least 2 arguments.
See 'docker exec --help'.

Usage:  docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

Run a command in a running container
``` 

예를 들어 다음은 이미 실행중인 컨테이너에 /bin/bash 커맨드를 수행한다(이는 결과적으로 컨테이너의 쉘을 습득한다). COMMAND의 종료는 실행중인 컨테이너에 영향을 주지 않는다.

```
docker exec -it f72ca548214e /bin/bash
```

## 도커를 이용한 개발

도커는 데이터 퍼시스턴스를 명시적으로 지정하지 않는 이상, 컨테이너를 제거하면 변경된 데이터 또한 소실된다. 이러한 불변성을 이용하면 매번 동일한 환경을 유지하면서 개발을 진행할 수 있다. 아마도 비즈니스 서비스를 만들면서 데이터베이스를 사용하지 않는 서비스는 거의 없을 것이다. 따라서 이 글에서는 MySQL 컨테이너를 개발에서 사용할 때 어떻게 사용할 수 있는지 그 예를 보여주려고 한다.

### MySQL 컨테이너 사용하기 

왜 굳이 컨테이너일까? 그냥 공용 물리서버에 MySQL 올려서 사용하면 되는 것 아닌가? 하지만 다음을 생각해 보자.

* 다른 버전의 MySQL을 테스트해보고 싶은 경우
* MySQL 설정을 바꿔가면서 테스트해보고 싶은 경우
* 슬로우 쿼리나 제너럴 로그를 남겨서 확인하고 싶은 경우
* 특정 테이블을 제거하거나, 칼럼 제거, 이름을 변경하면서 테스트해보고 싶은 경우
* 아무튼 별의별 짓을 해보고 싶은 경우

과연 공용 물리서버에 구성된 MySQL이라면 타인에게 불편함을 주지않고, 위와 같은 작업을 할 수 있을까? 물론 VM으로도 가능한 부분들이 있지만, 컨테이너에 비하면 너무나도 비효율적이다.

### GitOps

## 리소스 조정

## 참고

1. [컨테이너 vs VM](https://cloud.google.com/containers?hl=ko)
