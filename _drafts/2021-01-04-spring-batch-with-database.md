---
layout: post
excerpt_separator: <!--more-->
title: 'Spring Batch #1 - Utilizing Database'
categories: spring batch
---

# 스프링 배치 #2 - 데이터베이스 활용
## 개요

이번 포스팅에서는 데이터베이스를 활용하여 스프링 배치 애플리케이션을 작성하는 방법과 주의점들을 살펴보려고 한다.

{:refdef: style="text-align: center;"}
![spring batch](/assets/spring-batch.png)
{:refdef}
<!--more-->

## 데이터 읽기

데이터베이스에서 데이터를 읽는 방식에 따라서 두 가지 방식으로 나눌 수 있다.

* 페이징 방식
* 커서 방식

샘플코드를 통해 두 가지 방식의 차이점을 이해해 보도록 하자.

### 페이징 방식

### 커서 방식

## 트랜잭션의 범위

## 잘 짠 쿼리와 인덱스 생성은 중요하다

## 대상 데이터가 너무 많다면 페이징 방식을 사용하면 안된다

아무리 인덱스를 생성했다고 해도 페이징 방식을 사용하면 옵셋(offset)이 뒤로 갈수록 탐색이 느려진다. 

## Processor 에서 데이터베이스 접근은 지양하자

데이터베이스에 쿼리하는 것은 굉장히 비용이 많이 드는 작업이다. 되도록 지양하는 것이 좋고, 배치 처리할 때 데이터 변경이 없음이 보장되고, 데이터 크기가 
메모리에 충분히 담을 수 있는 크기라면 전체 데이터를 메모리에 올려 참조 데이터로 사용하는 것을 추천한다. 

## 벌크

insert, update 시 단건씩 여러건을 처리하기 보다는 여러건을 모아 한번에 처리하는 것이 좋다. 단건씩 처리할 경우 라운드트립 비용이 증가하는데 처리 
속도와 직결된다. 따라서 라운드트립 비용을 줄이는 것이 좋다. 그리고 MySQL 은 벌크 insert 와 update 를 지원한다. MySQL 같은 경우에는 insert 를 
단건으로 처리할 경우 필드 정보가 불필요하게 중복으로 남아 벌크 처리에 비해서 생성된 bin log 의 사이즈가 더 크다. 따라서 대량의 insert, update 를 
처리하는 경우 레플리카(슬레이브) 노드에서도 [라운드트립 + bin log 크기] 증가로 인해 bin log 가 밀리게 될 수 있다. 그러므로 대량의 데이터를 insert, 
update 해야 한다면 반드시 벌크로 처리하는 것이 좋다. 

## JPA 를 사용하는 경우



## 마무리

## 참고
