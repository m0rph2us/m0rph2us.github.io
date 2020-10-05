---
layout: post
excerpt_separator: <!--more-->
title: Docker Basics
categories: docker
---

# 도커 시작하기
## 개요

요즘의 개발, 인프라 구성은 컨테이너 기술을 활용하는 것이 대세가 되어가는 추세다. 이 글에서는 컨테이너의 개념을 간략히 설명할 것이다.
<!--more-->

## 컨테이너 vs VM

먼저 컨테이너와 VM의 차이를 확실히 짚고 넘어갈 필요가 있다. 이 두 가지가 작동하는 방식은 완전히 다르다.

{:refdef: style="text-align: center;"}
![architecture](/assets/containers-101-2x.png)
{:refdef}

위 그림에서 왼쪽이 VM, 오른쪽이 컨테이너를 보여주고 있다.

VM은 Virtual Machine, 즉, 가상의 컴퓨터다. 이렇게 가상의 컴퓨터를 구동해 주는 것을 하이퍼바이저(hypervisor)라고 하며, 이러한 하이퍼바이저에는 여러분도 
익숙한 VMWare, Oracle VirtualBox, QEMU, KVM 등이 있다. 가상의 컴퓨터이기 때문에 실제 컴퓨터처럼 윈도우, 리눅스, DOS 등을 설치하여 사용할 수 있다. 
호스트 OS의 종류와 관계없이 하이퍼바이저가 지원만 하면 되기 때문에 활용성이 굉장히 높다. 하지만, 호스트 OS 위에 또 다른 OS가 올라가는 형태이기 때문에 호스트 
OS의 리소스를 많이 사용할 수 밖에 없고, 실제 하드웨어 액세스는 게스트 OS <-> 하이퍼바이저 <-> 호스트 OS의 형태로 이루어져 작동이 매우 느릴 수 밖에 없다.

오른쪽 그림을 유심히 보면 게스트 OS와 하이퍼바이저 자리에 컨테이너 런타임이 위치한다. 이러한 컨테이너 런타임은 호스트 OS 커널 내부에서 컨테이너를 직접 구동하기 
때문에 VM에 비해 훨씬 빠르고, 메모리를 적게 차지한다. 널리 사용되는 컨테이너 런타임에는 도커, rkt 등이 있다. VM만큼의 리소스 격리성은 아니지만, 여전히 높은 
수준의 리소스 격리성을 제공한다.

컨테이너가 제공하는 높은 수준의 격리성, 빠른 애플리케이션 패키징 및 구동 속도는 최근의 소프트웨어 개발 및 배포 라이프사이클에 큰 영향을 주고 있으며, 생태계가 
잘 갖추어져 있다.

## 도커

{:refdef: style="text-align: center;"}
![architecture](/assets/engine-components-flow.png)
{:refdef}

도커는 가장 대중적인 컨테이너 런타임이다. 이것 저것 알아야 할 용어들이 굉장히 많지만, 가장 기본적인 것을 나열해 보면 다음과 같다.

* 도커 이미지
  * 컨테이너 인스턴스를 생성하기 위해 사용되는 이미지다. 이러한 이미지는 개발자가 바닥부터 직접 만들 수도 있고, 이미 만들어진 이미지를 이용하여 확장할 수도 
  있다.
* 도커 데몬
  * 도커 이미지, 컨테이너, 네트워크, 볼륨을 관리하는 서버이며 상호작용을 위한 REST API를 제공한다.
* 도커 레지스트리
  * 도커 이미지 저장소 정도로 생각하면 된다. 퍼블릭 레지스트리로는 도커허브([Docker Hub](https://hub.docker.com/))가 있다.
* 도커 클라이언트
  * 도커 데몬과 상호작용하는 클라이언트이며 도커 커맨드라인 인터페이스(CLI)가 이에 해당한다.
* 도커 컴포즈
  * 도커 클라이언트는 옵션이 굉장히 많다. 일일이 입력하여 사용하기에는 어려움이 있을 수 있기 때문에 편의를 위해 yaml로 정의하여 사용하는 것이 좋다.

기본적인 내용은 이정도면 됐고, 사용법을 알아보도록 하자.

## 기본 사용법

도커가 설치되어 있지 않다면 먼저 설치를 해야 한다. 설치 방법은 [여기](https://docs.docker.com/engine/install/)를 참고하기 바란다. 굉장히 다양한 
도커 커맨드가 존재하지만, 다음 여섯 가지가 가장 기본적으로 많이 사용하는 커맨드라고 보면 된다.

### images

로컬 레지스트리에 저장된 이미지 목록을 보여준다.

```
docker images
```

### ps

실행중인 컨테이너의 목록을 조회한다.

```
docker ps
```

중단된 컨테이너 목록까지 보고 싶은 경우 `-a` 옵션을 함께 준다.

### stop

실행중인 하나 혹은 여러 컨테이너를 중단한다. 중단하면 애플리케이션은 중단되지만 파일시스템 변경 사항은 소실되지 않는다. 이 상태에서 `commit` 커맨드를 통해 
새로운 이미지로 변경 상태를 반영할 수 있다.

```
"docker stop" requires at least 1 argument.
See 'docker stop --help'.

Usage:  docker stop [OPTIONS] CONTAINER [CONTAINER...]

Stop one or more running containers
```

이렇게 중단된 컨테이너는 `start` 커맨드로 다시 시작할 수 있다. 

### rm

하나 혹은 여러 컨테이너를 제거한다. 제거하면 파일스시템 변경 사항이 완전히 소실되며, `docker ps`로도 확인할 수 없다.

```
"docker rm" requires at least 1 argument.
See 'docker rm --help'.

Usage:  docker rm [OPTIONS] CONTAINER [CONTAINER...]

Remove one or more containers
``` 

### run

run은 새로운 컨테이너 인스턴스를 생성하고 [COMMAND]를 실행한다.

```
"docker run" requires at least 1 argument.
See 'docker run --help'.

Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container
```

예를 들어 다음은 mysql 커맨드를 사용하여 원격 서버에 접속한다. [COMMAND]가 종료되면 컨테이너도 함께 종료된다. 여기에서 `-it`는 컨테이너와 상호작용하기 
위한 옵션으로 보면된다. `--rm`은 컨테이너가 종료될 때 자동으로 파일시스템 변경사항을 제거한다.

```
docker run -it --rm mysql mysql -h192.168.1.11 -P3306 -uroot  -p
```

백그라운드 모드로 실행하고 싶다면 `-d` 옵션을 함께 줄 수 있다.

### exec

exec는 이미 실행중인 컨테이너에서 COMMAND를 실행한다.

```
"docker exec" requires at least 2 arguments.
See 'docker exec --help'.

Usage:  docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

Run a command in a running container
``` 

예를 들어 다음은 이미 실행중인 컨테이너에 `/bin/bash` 커맨드를 수행한다(이는 결과적으로 컨테이너의 쉘을 습득한다). COMMAND의 종료는 실행중인 컨테이너에 
영향을 주지 않는다.

```
docker exec -it f72ca548214e /bin/bash
```

## 마무리

컨테이너의 개념을 아주 간략하게 알아봤다. 사실 이해할 개념이 훨씬 많지만, 해당 부분은 조금씩 심화 코너로 글을 올리려고 한다.