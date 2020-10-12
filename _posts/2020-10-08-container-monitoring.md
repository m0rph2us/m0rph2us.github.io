---
layout: post
excerpt_separator: <!--more-->
title: 'Container Monitoring'
categories: container monitoring fluentd cadvisor prometheus elasticsearch elastalert CMAK grafana
---

# 컨테이너 모니터링
## 개요

이번 포스팅에서는 컨테이너 모니터링에 대해서 살펴보려고 한다. 모니터링 환경은 시스템을 운영하는 데 있어 없어서는 안될 부분이다. 모니터링은 로그, 지표 수집, 경보를 
모두 고려해야 한다. 이 포스팅에서는 docker-compose 로 구성된 컨테이너 환경을 기반으로 로깅, 지표 수집을 어떻게 하는지 살펴볼 것이다. 테스트 목적의 구성이므로 
프로덕션에서 사용하려면 하드웨어 구성에 따라 옵션을 조정해가면서 적합 여부를 테스트해보기 바란다.

{:refdef: style="text-align: center;"}
![fluentd](/assets/fluentd.png)
{:refdef}
<!--more-->

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

## 경보

기본적으로 일래스틱서치는 경보 옵션을 제공하지 않는다. 유료 옵션을 구매해야 하는데, 그러지 않고도 로그에 기반하여 경보를 받을 수 있는 수단이 존재한다. 
`elastalert` 이라는 도구를 사용하면 되며, 슬랙, 팀즈 같이 대중적으로 사용되는 채널들을 폭넓게 지원하고, 경보 폭탄을 받지 않도록 설정을 조절할 수 있는 장점이
있다.

## 마무리

비즈니스에서 장애는 치명적이고, 빠르게 복구되어야 한다. 따라서 문제를 빠르게 추적하고 현상을 이해하기 위해 로그 모니터링 시스템을 잘 구성하고 유지해야 한다.