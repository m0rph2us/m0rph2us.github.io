---
layout: post
excerpt_separator: <!--more-->
title: Understanding  MySQL Transaction Isolation Level
categories: mysql transaction
---

# MySQL 의 트랜잭션 격리 수준 이해하기
## 개요

이 글에서는 MySQL 데이터베이스의 트랜잭션 격리 수준에 대해 살펴본다. 트랜잭션 격리 수준은 트랜잭션 내에서 어느 정도의 조회 일관성을 보장하느냐를 의미한다. 이러한 트랜잭션 격리 수준을 제대로 이해하고 있어야 현재 애플리케이션이 사용하는 트랜잭션 격리 수준에서 잠재적으로 어떤 현상이 발생할 수 있는지 알 수 있다.
<!--more-->

# MySQL 

MySQL은 SQL:1992 표준에 기술된 다음의 격리 수준을 제공한다.

* READ_UNCOMMITED
* READ_COMMITED
* REPEATABLE_READ
* SERIALIZABLE

별도로 설정하지 않았다면 기본 격리 수준은 REPEATABLE_READ 이다.

# READ_UNCOMMITED

격리 수준 중에서 제일 낮은 격리 수준이다. 이 트랜잭션 격리 수준에서는 A 트랜잭션에서 변경만 하고 커밋하지 않는 내용을 B 트랜잭션에서 조회할 수가 있다. B 트랜잭션은 이후 A 트랜잭션이 롤백 또는 커밋이 되었는지 알 수 없다. 흔히 이러한 읽기를 더티 리드(Dirty Read)라고 한다. 이 단계에서는 넌리피터블 리드(Non-Repeatable Read)와 팬텀 리드(Phantom Read) 발생 가능성도 함께 공존한다.

# READ_COMMITED

A 트랜잭션에서 변경하고 커밋한 내용을 B 트랜잭션에서 조회할 수 있다. 커밋하지 않은 내용은 B 트랜잭션에서 조회할 수 없다. 더티 리드가 허용되는 READ_UNCOMMITED 보다 확실히 한 단계 수준이 높다는 것을 알 수 있다. 더티 리드를 허용하지 않지만, 다른 트랜잭션에서 커밋한 데이터를 조회할 수 있으므로 이 역시 트랜잭션 내에서 일관된 조회를 보장하지는 않는다. 이를 넌리피터블 리드(Non-Repeatable Read)라고 한다. 이 단계에서는 팬텀 리드 발생 가능성이 함께 공존한다.

# REPEATABLE_READ

A 트랜잭션에서 어떤 변경이 일어나도 B 트랜잭션은 트랜잭션 내에서 항상 동일한 조회를 보장한다. 보통의 경우, B 트랜잭션에서의 조회가 범위 조회인 경우 A 트랜잭션에서의 INSERT 로 인해 B 트랜잭션에서는 팬텀 리드(Phantom Read)가 발생할 수 있다. 하지만 MySQL 의 경우 이 격리 수준에서는 A 트랜잭션에서 INSERT 가 발생해도 B 트랜잭션에서 팬텀 리드가 발생하지 않는다. 왜냐하면 이 격리 수준의 경우 MySQL 은 트랜잭션 내에서 처음 조회할 때 만들어진 스냅샷을 조회하기 때문이다.

# SERIALIZABLE

A 트랜잭션에서의 모든 조회가 SHARED LOCK 을 유발한다. SHARED LOCK 은 여러 트랜잭션에서 조회가 가능하지만 변경은 LOCK 이 해제될 때까지 대기를 타게 된다. 즉, 트랜잭션이 종료되기 전까지는 다른 트랜잭션에서 어떤 변경도 허용하지 않기 때문에 항상 일관성 있게 데이터를 조회할 수 있다.

## 마무리

격리 수준이 높아질수록 트랜잭션내에서 데이터 일관성은 높아지지만, 성능적으로는 느려진다. SHARED LOCK은 인덱스가 사용되지 않으면, 테이블 락이 걸리므로 유의해야 한다. 격리 수준은 말 그대로 트랜잭션내의 데이터 일관성에 관한 것이기 때문에 동시성을 처리하기 위해서는 LOCKING 메커니즘을 사용해야 한다.

## 참고

1. [MySQL 격리 수준](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
