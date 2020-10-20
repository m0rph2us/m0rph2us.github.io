---
layout: post
excerpt_separator: <!--more-->
title: 'Brief view of Openfire'
categories: mysql cdc debezium
---

# 오픈파이어 간략히 살펴보기
## 개요

이번 포스팅에서는 XMPP 에 대해서 간략히 알아보고, 채팅 서비스를 구현해야 되는 상황에서 오픈파이어의 채팅 기능이 얼마만큼 구현되어 있는지 살펴보았다.
<!--more-->

## XMPP

표준 메시지 전송 프로토콜이다. 이 프로토콜을 자바로 구현한 구현체중의 하나가 오픈파이어다.

{:refdef: style="text-align: center;"}
![xmpp-client-server-arch](/assets/xmpp-client-server-arch.png)
{:refdef}

표준을 충실히 구현하는 경우 이상적인 클라이언트 서버 아키텍처는 위와 같다.

역사가 깊은? 프로토콜이다보니 서버 구현체, 클라이언트 구현체, 라이브러리 등이 풍부한 편이다.

## 스탠자

기본적인 메시지 스트림의 구조는 다음과 같다. `C` 가 클라이언트가 보내는 메시지이고, `S` 가 서버가 보내는 메시지이다.

```
C: <stream:stream>

C:   <presence/>

C:   <iq type="get">
       <query xmlns="jabber:iq:roster"/>
     </iq>

S:   <iq type="result">
       <query xmlns="jabber:iq:roster">
         <item jid="alice@wonderland.lit"/>
         <item jid="madhatter@wonderland.lit"/>
         <item jid="whiterabbit@wonderland.lit"/>
       </query>
     </iq>

C:   <message from="queen@wonderland.lit" 
              to="madhatter@wonderland.lit">
       <body>Off with his head!</body>
     </message>

S:   <message from="king@wonderland.lit"
              to="party@conference.wonderland.lit">
       <body>You are all pardoned.</body>
     </message>

C:   <presence type="unavailable"/>

C: </stream:stream>
```

iq, message, presence 를 일컬어 스탠자(stanza)라고 부른다. XMPP 프로토콜에서 통신의 기본 단위가 된다.

* iq
    * 화면에 노출되지 않는 정보를 요청하고 받을 때 사용한다.
* message
    * 화면에 노출되는 정보를 주고 받을 때 사용한다.
* presence
    * 사용자의 온라인 상태 유무등을 주고 받을 때 사용한다.

모든 스탠자가 확장 가능한 구조이지만, presence 스탠자는 확장할 경우 성능 보틀넥이 될 수 있으므로 확장을 지양하는 것이 좋다(다른 스탠자보다 전송 빈도가 
굉장히 높다고 한다).

## 살펴볼 기본 기능

기본적으로 아래와 같은 기능이 기본 제공되어야 살을 붙여가면서 개발을 할 수 있을 것 같았기 때문에 해당 기능들이 기본적으로 지원되는지 확인했다. 

* 오프라인 유저에게 채팅 메시지가 전달되어야 한다
* 최근에 읽은 채팅 메시지를 구분할 수 있어야 한다
* 오프라인 상태에서 전달된 채팅 메시지를 구분할 수 있어야 한다
* 사용자가 입력 진행상태인지 알 수 있어야 한다
* DM이 가능해야 한다

온/오프라인 상태 확인이나 채팅 메시지를 주고 받는 등의 기능은 기본 기능이기 때문에 제외하고, 되는지 여부가 불투명한 부분으로 확인했다.

테스트 구성은 다음과 같다. 

{:refdef: style="text-align: center;"}
![openfire](/assets/openfire.png)
{:refdef} 

서버는 오픈파이어, IM 은 스파크(Spark)를 사용했다.

## 오프라인 유저에게 채팅 메시지가 전달될까?

서로 친구로 추가하지 않은 상태에서 채팅을 시도했고(아마 DM에 해당할 것 같다), jenny 가 오프라인된 상태에서 michael 쪽에서 채팅을 시도한 모습이다.

{:refdef: style="text-align: center;"}
![openfire-001](/assets/openfire-001.png)
{:refdef}

jenny 가 나중에 온라인이 되었을 때 오프라인 상태일 때 도착한 메시지의 확인이 가능했다. 하지만 오프라인 상태에서 온 메시지에 대해서 ***사용자에게 
푸시나 이메일을 통해서 알림을 줘야 한다면*** 이 부분에 대한 고민은 필요할 것 같다.

## 최근에 읽은 채팅 메시지를 구분할 수 있을까?

jenny 의 채팅 창을 닫고, michael 이 채팅 메시지를 보내면 새로 도착한 메시지가 아래와 같이 단락이 구분되는 것을 알 수 있다.

{:refdef: style="text-align: center;"}
![openfire-002](/assets/openfire-002.png)
{:refdef}

하지만 창을 비활성 해둔 상태에서는 마지막으로 읽은 메시지와 구분되지 않는 모습이었는데, 이 부분은 가능 여부를 더 확인해 봐야 할 것 같다.

## 사용자가 입력 진행상태인지 알 수 있을까?

아래의 입력상자 바로 위를 보면 `jenny has stopped typing` 이라고 되어 있는데, 입력이 시작되면 상태 메시지가 바뀐다. 따라서 상대방이 입력 진행상태인지 
알 수 있었다.

{:refdef: style="text-align: center;"}
![openfire-002](/assets/openfire-002.png)
{:refdef}

## DM이 가능할까?

결론적으로는 가능한 것으로 보인다. 왜냐하면 jenny 와 michael 모두 서로 친구가 아닌 상태에서 그리고 특정 방에 들어가 있지 않은 상태에서 대화가 가능했기 
때문이다. 그리고 둘 사이의 대화는 둘 다 메시지 창을 닫아도 계속해서 유지가 되어, 후속 대화에서 이전 대화를 계속 확인할 수 있었다. 하지만, DM 의 경우 
온라인 오프라인 유무가 확인되지 않는데, 이는 로스터(친구 목록)에 포함된 경우에만 가능한 것이 아닌가 싶다.

## 마무리

앞서 정의했던 최소한의 채팅 기능들은 무리없이 되는 것으로 보인다. 여기에서 막을 것들은 막고, 확장할 것들은 확장하면 무리없이 채팅 서비스의 틀을 갖출 수 있을 
것으로 보인다.

## 참고

* XMPP: The Definitive Guide
* [Docker Openfire](https://github.com/m0rph2us/docker-openfire)