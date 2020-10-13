---
layout: post
excerpt_separator: <!--more-->
title: 'Exploring Open-source Chat'
categories: chat matter-most
---

# 오픈소스 챗 살펴보기
## 개요

회사 신규 시스템에서 채팅 기능이 필요하여 리서치한 내용을 남겨본다. 직접 만드는 게 제일 재밌지만 ㅎㅎ 회사는 나에게 그런 재밌는 시간을 허락하지 않으므로, 
오픈소스 설루션을 살펴보았다. 
<!--more-->

비교 대상 제품은 다음과 같다.

* XMPP
    * https://xmpp.org/
* Rocket Chat
    * https://rocket.chat/
* Mattermost
    * https://mattermost.com/
* Zulip
    * https://zulipchat.com/
    
비교 지표 항목은 다음과 같다.

* 구현 언어
* 오픈 소스 여부
* 고가용성 구축
* 문서 수준
* 가격 및 라이센싱 정책
* 레퍼런스
* 장점
* 단점

## XMPP

먼저, XMPP 이다. 가장 익숙하게 들어뵜을 법하다.

* 구현 언어
    * XMPP 는 프로토콜이기 때문에 언어마다 여러 구현체가 존재한다.
    * 구현체
        * Ejabberd
            * Erlang
        * Openfire
            * Java
* 오픈 소스 여부
    * 여러 오픈소스 구현체가 존재한다.
* 고가용성 구축
    * Openfire
        * https://secureanycloud.com/openfire-on-cloud-technical-support-cloud-help-azure-aws-opensource-cognosys/
    * 
* 문서 수준
    * 프로토콜 문서가 곧 문서이다.
* 가격 및 라이센싱 정책
    * 오픈소스 구현체의 라이센스 정책에 따르지만, 대부분의 구현체가 무료다.
* 레퍼런스
    * https://xmpp.org/uses/gaming.html
    * 레퍼런스 자체는 훌륭해 보임
* 장점
    * 역사가 있는 프로토콜
    * 책도 있고, 레퍼런스도 있는 편
* 단점
    * 프로토콜이 XML 로 되어 있어 서버 부하를 유발하고 유지 비용이 올라가게 하지 않을까?
    * 너무 오래되었고, 최근 10년 이내에 출판된 책이 없다.

## Rocket chat

* 구현 언어
    * javascript
* 오픈 소스 여부
    * https://github.com/RocketChat/Rocket.Chat
* 고가용성 구축
    * https://rocket.chat/pricing/
    * 플랜을 보면 엔터프라이즈 에디션 제공하는 것으로 되어 있지만, 커뮤니티 에디션으로도 충분한 듯
        * https://docs.rocket.chat/installation/docker-containers/high-availability-install 
* 문서 수준
    * https://docs.rocket.chat/?gclid=undefined
    * 부족해 보임
        * 자바스크립트 사용 예제가 없다
* 가격 및 라이센싱 정책
    * https://rocket.chat/pricing/
    * MIT 라이센스
    * 클라우드 지원
    * 셀프 호스팅
        * 커뮤니티 제공하지만, 엔터프라이즈만의 피쳐는 빠져 있음

## Mattermost

* 구현 언어
    * go
* 오픈 소스 여부
    * https://github.com/mattermost/mattermost-server
* 고가용성 구축
    * 엔터프라이즈 에디션 구입이 필수이다
    * 코어만을 가지고 구축할 수 있을지는 확인이 필요하다
* 문서 수준
    * https://api.mattermost.com/
    * 부족해 보임
        * 자바스크립트 사용 예제가 없다
* 가격 및 라이센싱 정책
    * 클라우드 지원
    * 셀프 호스팅
        * 코어만 프리
        * 대부분의 기본 피쳐들은 엔터프라이즈 에디션 구입 필요
    * MIT 라이센스

## Zulip

* 구현 언어
    * python
* 오픈 소스 여부
    * https://github.com/zulip/zulip
* 고가용성 구축
    * 엔터프라이즈 에디션에 지원하는 것으로 나와 있지만, 커뮤니티 에디션으로 독자적인 HA 구성을 할 수 있는지 확인이 필요하다
    * https://zulip.readthedocs.io/en/stable/production/requirements.html#scalability
* 문서 수준
    * https://zulipchat.com/api/
    * 좋음
        * 자바스크립트 사용 예제가 있다
        * 자바스크립트 전용 클라이언트 라이브러리를 제공함
* 가격 및 라이센싱 정책
    * 클라우드 지원
    * 셀프 호스팅
        * 커뮤니티 에디션
            * 스탠다드 및 엔터프라이즈 피쳐 모두 제공
        * 엔터프라이즈 에디션
            * 고가용성 지원
    * 아파치 2.0 라이센스