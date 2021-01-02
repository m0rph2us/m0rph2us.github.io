---
layout: post
excerpt_separator: <!--more-->
title: Drogon - Get Started
categories: drogon
---

# 드로곤(Drogon) - 시작하기
## 개요

몇 달 전에 프레임워크 벤치마크에서 최상위를 차지했던 프레임워크라서 한번 테스트해보고 싶었다. 이 문서는 드로곤을 빌드하고 웹 애플리케이션을 작성하여 띄우는 
방법에 대해 살펴본다.

{:refdef: style="text-align: center;"}
![drogon benchmark](/assets/drogon-benchmark-2020.png)
{:refdef}
<!--more-->

# 드로곤 빌드하기

이 프로젝트의 소스코드는 [https://github.com/an-tao/drogon](https://github.com/an-tao/drogon) 에 있으면, 다음과 같이 클론 받을 수 있다.

```shell
git clone https://github.com/an-tao/drogon.git
```

클론받은 디렉터리 내부로 이동하면, `Dockerfile` 이 있는 것을 알 수 있다. 이 도커파일을 빌드하면 드로곤이 빌드되어 설치된 개발환경 이미지가 만들어진다.

```shell
docker build -t drogon/commpile-env .
```

# 첫 프로젝트 만들기

드로곤은 프로젝트의 초기 뼈대를 만들어 주는 유틸리티를 제공한다. 다음은 `test-app` 이라는 프로젝트 디렉터리를 생성한다.

```shell
drogon_ctl create project test-app
```

도커를 사용하지 않은 날것의 명령은 위와 같은데 도커 이미지를 사용하면 아래와 같이 할 수 있다.

```shell
docker run --rm --volume="$PWD:/drogon-project" -w="/drogon-project" drogon/compile-env drogon_ctl create project test-app
```

# 프로젝트 빌드하기

`test-app` 디렉터리 내부에 있다고 가정하면, 다음과 같이 수행할 수 있다.

```shell
docker run --rm --volume="$PWD:/drogon-project" -w="/drogon-project/build" drogon/compile-env sh -c "cmake .. && make" 
```

# 애플리케이션 실행하기

먼저 도커이미지로 만들기 위해 프로젝트 디렉터리 바로 밑에 `Dockerfile` 을 다음과 같이 작성하도록 한다. 실행파일은 `build` 디렉터리 밑에 위치한다.

```shell
FROM ubuntu:18.04

RUN apt-get update -yqq \
    && apt-get install -yqq --no-install-recommends software-properties-common \
    sudo curl wget cmake pkg-config locales git gcc-8 g++-8 \
    openssl libssl-dev libjsoncpp-dev uuid-dev zlib1g-dev libc-ares-dev\
    postgresql-server-dev-all libmariadbclient-dev libsqlite3-dev \
    && rm -rf /var/lib/apt/lists/* \
    && locale-gen en_US.UTF-8

ADD build/test-app /app/test-app

WORKDIR /app

CMD ./test-app
```

그리고 이미지를 빌드하고

```shell
docker build -t test/drogon-app .
```

다음과 같이 실행하면 된다. 

```shell
docker run --rm -d -p 8080:80 test/drogon-app 
```

검증은 다음과 같이 할 수 있다. 404 페이지가 표시되면 정상이다.

```shell
curl http://localhost:8080
```

# 참고

* [Dockerzing Drogon Web Application](https://medium.com/@srdbranding/how-to-dockerize-a-c-drogon-web-application-web-app-in-an-existing-project-for-the-web-7778ec7ddcd7)
