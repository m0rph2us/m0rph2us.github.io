---
layout: post
excerpt_separator: <!--more-->
title: 'Testing Spring Application'
categories: spring test
---

# 스프링 애플리케이션 테스트
## 개요

이번 포스팅에서는 스프링 애플리케이션을 작성하고 테스트를 작성할 때 흔히 접근하는 방식을 설명하려고 한다.

<!--more-->

## 테스트 코드는...

테스트 코드는 ...

* 프로덕션에 배포되는 애플리케이션이라면 필수이다.
* 테스트를 수행하기 위한 준비 작업이 간결해야 한다.
* 어느 개발자의 PC 에서 수행되더라도 큰 노력없이 테스트 코드를 수행할 수 있어야 한다.
* 외부 연동 지점과 격리된 상태에서도 수행할 수 있어야 한다.
* 테스트 코드는 애플리케이션 작성을 느리게 하는 것이 아니라 더 빨리 만들 수 있도록 도와준다.
* 수행 속도가 너무 느려서는 안된다.
* 자동으로 테스트를 수행할 수 있는 환경을 구성할 수 있다면 좋다.
* 베스트하지 않더라도 기능을 충분히 테스트하고 검증 및 보장할 수 있는 코드라면 어떤식으로든 작성하는 것이 아예 작성하지 않는 것보다 낫다.

다음부터 설명하는 내용만 알아도 스프링 애플리케이션을 작성하면서 테스트 코드를 작성하는 데 무리가 없으니 꼭 알아두도록 하자.

## 내가 만든 API 엔드포인트를 어떻게 테스트 하나요?

`MockMvc` 를 사용하면 된다. 사용법은 아주 쉬우며 기본적인 틀은 아래와 같다. 이와 함께 `RestDocs` 를 함께 활용하면 테스트 작성과 
API 문서 작성을 한꺼번에 할 수 있다는 장점이 있다.

```java
@SpringBootTest
@AutoConfigureMockMvc
public class MyTest {

    @Autowired
    private MockMvc mvc;

    @Test
    public void getRobots() throws Exception {
    
        // prepare test data
    
        mvc.perform(get("/api/robots")
          .contentType(MediaType.APPLICATION_JSON))
          .andExpect(status().isOk())
          .andExpect(content()
          .contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
          .andExpect(jsonPath("$[0].name", is("beepy")));
    }
}
```

## @MockBean, @SpyBean 을 적절히 사용하자

이 애너테이션들은 스프링 컨텍스트의 특정 빈 인스턴스의 작동을 목킹하고 싶을 때 사용한다. 이는 실제로 어떤 빈 인스턴스의 메서드가 
특정한 값을 반환하도록 하고 싶을 때 유용하게 사용할 수 있다. 둘 사이의 차이점은 목킹되는 방식에 있다. `@MockBean` 은 목킹된 프록시 인스턴스가
주입되는데 실제 구현체 인스턴스를 프록싱하지 않기 때문에 텅텅 비어있다고 생각하면 된다. 반면 `@SpyBean` 은 실제 구현체 인스턴스를 프록싱한다고 
생각하면 된다.

다음은 `@MockBean` 사용 예제이다.

```java
public interface HelloService {
  String getMessage();
}

@Component
public class MyHelloProcessor {
  @Autowired
  private HelloService helloService;

  public String getMessage() {
      return helloService.getMessage();
  }
}

@SpringBootTest
public class MyHelloProcessorTest {
  @Autowired
  private MyHelloProcessor myHelloProcessor;

  @MockBean
  private HelloService helloService;

  @Test
  public void testSayHi() {
      Mockito.when(helloService.getMessage()).thenReturn("test message.");
      String message = myHelloProcessor.getMessage();
      Assert.assertEquals(message, "test message.");
  }
}
```

다음은 `@SpyBean` 사용 예제이다.

```java
public class HelloService {
  public String getMessage(){
      return null;
  }

  public String getFormattedMessage() {
      return "Hi there! " + getMessage();
  }
}

@Component
public class MyHelloProcessor {
  @Autowired
  private HelloService helloService;

  public String getMessage() {
      return helloService.getMessage();
  }

  public String getFormattedMessage() {
      return helloService.getFormattedMessage();
  }
}

@SpringBootTest
public class MyHelloProcessorTest {
  @Autowired
  private MyHelloProcessor myHelloProcessor;

  @SpyBean
  private HelloService helloService;

  @Test
  public void testSayHi() {
      Mockito.when(helloService.getMessage()).thenReturn("test message.");
      String message = myHelloProcessor.getMessage();
      Assert.assertEquals(message, "test message.");

      String formattedMessage = myHelloProcessor.getFormattedMessage();
      Assert.assertEquals(formattedMessage, "Hi there! test message.");
  }
}
```

## 도커를 적극 사용하자

도커를 활용하는 케이스는 mysql, redis 등과 같이 외부 연동 지점 시스템을 가진 경우이다. 도커를 이용해 이러한 환경을 
구성할 수 있는 경우라면 도커를 이용해 테스트를 수행하는 것도 한 가지 방법이 될 수 있다. 이 경우 장점은 마치 통합 
테스트를 하는 것처럼 최대한 유사한 환경으로 테스트를 수행할 수 있다는 것이고, 해당 시스템이 가지고 있는 고유의 특성들을 
테스팅 시에 함께 확인할 수 있다는 장점이 있다. 즉, mysql 고유의 특성을 예를 들면, mysql 만이 지원하는 전용 함수와 구문, 
master-slave 리플리케이션 구성 등을 예로 들 수 있다.

데이터베이스의 경우 리파지토리를 목킹하는 것을 생각해 볼 수 있겠지만, 이 경우 데이터베이스 특성으로 인한 부분을 전혀 확인할 수 없기 때문에
이에 대한 테스트를 별도로 진행하지 않고 프로덕션으로 배포했을 경우 데이터베이스 특성을 고려하지 못한 코드로 인해 장애가 발생할 가능성이 
높아지게 된다. H2 같은 인메모리 데이터베이스 역시 마찬가지이다. mysql 에뮬레이션을 지원하는 경우도 있지만 고유의 특성까지 재현하지는 못한다.

그 밖에도 도커는 VM에 비해 빠르게 띄울 수가 있고, 손쉽게 초기 상태로 되돌아 갈 수 있다는 장점이 있다.

## 외부 엔드포인트 연동

아마도 다른 서비스와의 http 통신이 이에 해당할 것이다. 이런 경우에는 `WireMock` 을 사용할 수 있다. 이렇게 하면 외부 서버없이도
엔드포인트를 목킹할 수 있다. 간략한 사용예는 다음과 같다.

```java
@Test
public void exactUrlOnly() {
    stubFor(get(urlEqualTo("/some/thing"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "text/plain")
                .withBody("Hello world!")));

    assertThat(testClient.get("/some/thing").statusCode(), is(200));
    assertThat(testClient.get("/some/thing/else").statusCode(), is(404));
}
```

## 특정 시간으로 테스트

테스트 코드를 작성하다 보면 특정 시간에 작동하는지 확인해야 될 때가 있는데, 이에 대한 테크닉으로는 `Clock` 을 목킹하는 방법이 있다.

```java
@Bean
fun appClock(): Clock {
    return Clock.systemDefaultZone()
}

// 애플리케이션에서는 다음과 같이 사용한다.
LocalDate.now(appClock);

// 그리고 테스트 코드에서는 다음과 같이 목킹한다.
@MockBean
Clock clock;

public static void setupFixedClock(Clock clock, String fixedDt) {
    LocalDateTime dt = DateUtil.fromDtFormat(fixedDt);
    Clock fixedClock = Clock.fixed(dt.atZone(ZoneId.systemDefault()).toInstant(), ZoneId.systemDefault());

    Mockito.when(clock.instant()).thenReturn(fixedClock.instant());
    Mockito.when(clock.getZone()).thenReturn(fixedClock.getZone());
}
```

## AWS?

요즘 가장 테스트하기 까다로운 환경일 것 같다. 이에 대한 대안으로는 현재로서는 `localstack` 이 가장 유력해 보인다.

## 참고

1. [localstack](https://github.com/localstack/localstack)
1. [WireMock](http://wiremock.org/)
1. [MockMvc](https://www.baeldung.com/integration-testing-in-spring)
1. [Overriding system time](https://www.baeldung.com/java-override-system-time)