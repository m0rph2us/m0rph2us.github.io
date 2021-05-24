---
layout: post
excerpt_separator: <!--more-->
title: 'Essence of kotlin'
categories: kotlin
---

# oAuth 이해하기
## 개요

이 문서는 oAuth 에 대한 개념을 정리한다.

## 구성 요소들

oAuth 를 구성하는 기본적인 요소들은 다음과 같다.

* `Resource Owner`
    * 사용자(회원)
* `User Agent`
    * `Resource Owner` 가 `Client` 와 상호작용하기 위해 사용하는 에이전트(예를 들어 웹브라우저)
* `Authorization Server`
    * 사용자 인증, 권한 인가 및 토큰 발급(`아이디 토큰`, `액세스 토큰`, `리프레시 토큰`)
* `Resource Server`
    * 일반적인 API 서버
        * 통합회원 API 서버
        * 상품, 구매후기 API 서버
        * 등등
* `Client`
    * `Authorization Server` 를 통해 발급받은 토큰으로 `Resource Owner` 를 대신해 
    `Resource Server` 에 액세스하는 애플리케이션.
        * 통합회원
        * 쇼핑몰
        * 등등

## 인증 흐름

인증 흐름은 다음과 같이 4가지 정도로 나눌 수 있다.

* `Authorization Code Flow`
    * 리다이렉션 기반
    * 웹 애플리케이션인 경우 가장 안전한 선택
* `Implicit Flow`
    * Javascript 로만 구현된 애플리케이션인 경우
    * 액세스 토큰을 안전하게 보관할 수단이 존재하지 않으므로 권장하지 않음
* `Resource Owner Password Flow`
    * 리다이렉션 기반 인증 흐름을 사용할 수 없는 경우
    * `Resource Owner` 가 제공한 id/password 기반으로 인증
    * `Client` 를 완벽하게 신뢰할 수 있는 경우
* `Client Credential Flow`
    * `Client` 자체가 `Resource Owner` 인 경우 
    * 예를 들어 machine to machine 인가에 해당
    * `Client` -> `Resource Server`

### Authorization Code Flow

{:refdef: style="text-align: center;"}
![athorization-code-flow](/assets/authorization-code-flow)
{:refdef}

## 토큰

토큰은 다음과 같이 3가지로 나눌 수 있다.

* 아이디 토큰
    * `Resource Owner` 에 대한 부분적인 정보가 담긴다
    * 이 토큰으로는 `Resource Server` 에 접근할 수 없다
* 액세스 토큰
    * `Resource Server` 에 접근하기 위해 필요한 토큰
    * 일반적으로 리프레시 토큰보다 생명 주기가 짧다
    * `Resource Owner` 에 대한 정보가 없음(아이디 외에 이름이나 이메일 등의 정보가 없다)
* 리프레시 토큰
    * 액세스 토큰을 재발급 받기 위해 사용하는 토큰
    * 일반적으로 액세스 토큰보다 생명 주기가 길다

## Authorization Server

먼저, `Spring Security` 의 `Authorization Server` 지원에 대한 연혁은 다음과 같다.

* 2019/11/14
    * 지원 중단을 공표함. 이미 `Spring Security` 에서 제공하는 솔루션보다 좋으면서 표준을 지원하는 
    오픈/상용 제품들이 많이 존재한다는 이유.
    * https://spring.io/blog/2019/11/14/spring-security-oauth-2-0-roadmap-update
* 2020/04/15
    * 하지만, 이후 커뮤니티에서 지속적으로 `Spring Security` 에서 지원할 필요가 있다고 말이 나온 모양이다. 그래서 
    실험적 영역에서 커뮤니티 프로젝트로 지속하기로 한 듯.
    * https://spring.io/blog/2020/04/15/announcing-the-spring-authorization-server
    
메이븐 리포지토리를 보면 [spring-security-oauth2-authorization-server](https://mvnrepository.com/artifact/org.springframework.security.experimental/spring-security-oauth2-authorization-server)
패키지가 `experimental` 로 **`실험`** 단계인 것을 확인할 수 있다.

`Resource Server`, `Client` 같은 경우는 지원이 지속적으로 잘 되고 있기 때문에 `Spring Security` 제공 프레임워크로 
처리할 수 있지만, `Authorization Server` 의 경우는 필요한 경우 다른 서드파티 대안을 찾아봐야 될 수도 있겠다.

검색을 해보면 알려진 `Authorization Server` 로 자주 거론되는 솔루션을 다음 3가지로 압축할 수 있다.

* `Keycloak`
    * 오픈소스
    * SPI(Service Provider Interface) 제공으로 커스터마이징 프렌들리 함
    * 레드햇 스폰싱
    * 문서와 레퍼런스가 좋은 편
* `Okta`
    * Auth0 를 인수함
    * 15000 MAU 까지 무료이며 차등 유료 정책
* `Auth0`
    * 7000 AU 까지 무료이며 차등 유료 정책
    
`KeyCloak` 의 릴리스가 꾸준하고, 레드햇의 스폰싱을 받고 있는점, 문서 및 레퍼런스가 괜찮은 점, 커스터마이징 프렌들리 하다는 
점에서 고려할만하다고 판단된다.

## Keycloak

`Keycloak` 은 스탠드얼론이나 스프링 임베딩 방식으로도 구동할 수 있다.

다음은 `Keycloak` 으로 인증 및 인가 구현시 참고할만한 내용들이다.

* 로그인 UI 커스터마이징
    * 지역화, 템플릿, CSS 수정 등 거의 원하는 형태로 커스터마이징이 가능하다.
    * https://www.baeldung.com/keycloak-custom-login-page
* 사용자 데이터베이스 연동
    * 기존 사용자 데이터베이스를 연동할 수 있는 SPI 제공 
    * https://www.baeldung.com/java-keycloak-custom-user-providers
* 자바스크립트 어댑터 지원
    * https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter
* 소셜 로그인
    * Facebook, Google: Built-in
    * Apple: https://github.com/BenjaminFavre/keycloak-apple-social-identity-provider
    * Kakao: https://github.com/origandrew/keycloak-kakao

## 참고

* https://www.baeldung.com/sso-spring-security-oauth2
* https://dzone.com/articles/open-id-connect-authentication-with-oauth20-author
* https://docs.spring.io/spring-security-oauth2-boot/docs/2.2.x-SNAPSHOT/reference/html/boot-features-security-oauth2-authorization-server.html
* https://mvnrepository.com/artifact/org.springframework.security.experimental/spring-security-oauth2-authorization-server
* https://kimseungjae.tistory.com/15
* https://www.baeldung.com/spring-security-oauth-resource-server
* https://coding-start.tistory.com/153?category=869723
* https://godekdls.github.io/Spring%20Security/oauth2webflux/
* https://tech.socarcorp.kr/security/2019/07/31/keycloak-sso.html
* https://league-cat.tistory.com/397
* https://sultanov.dev/blog/migrate-from-spring-security-oauth-to-keycloak/
* https://auth0.com/docs/authorization/which-oauth-2-0-flow-should-i-use
* https://darutk.medium.com/diagrams-of-all-the-openid-connect-flows-6968e3990660
* https://cloud.ibm.com/docs/appid?topic=appid-tokens&locale=ko
* https://oingdaddy.tistory.com/196
* https://www.baeldung.com/spring-boot-keycloak
* https://www.baeldung.com/keycloak-embedded-in-spring-boot-app
* https://www.baeldung.com/keycloak-custom-login-page
* https://www.baeldung.com/java-keycloak-custom-user-providers
* https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter
* https://github.com/BenjaminFavre/keycloak-apple-social-identity-provider
* https://github.com/origandrew/keycloak-kakao