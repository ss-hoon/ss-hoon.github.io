---
title: 스프링 배치(Spring Batch) - 1
categories: [Spring]
tags: ['Batch', 'Scheduler']
toc: true

date: 2022-12-26
last_modified_at: 2022-12-29
---

## 1. 들어가기

이번에 회사 프로젝트에서 Batch 작업을 맡게 되었습니다.

정보처리기사 시험을 준비할 때 Batch에 대해 간략하게 공부한 적이 있지만 정확히 기억나지 않기 때문에

Batch를 공부할 겸, 회사 프로젝트의 업무를 돕고자 이번 포스트를 작성합니다.

## 2. Batch

Batch란 무엇일까요? 🤔

Batch를 한국어로 해석하면 **"일괄"**이라는 의미입니다.

일괄은 **"개별적인 여러 가지의 것을 한데 묶음"**이라는 뜻으로 한 번에 처리한다는 의미를 가지고 있습니다.

즉, Batch는 데이터를 실시간으로 처리하는 것이 아닌 일괄적으로 모아서 처리하는 작업입니다.

일괄적으로 모아서 처리할 수 있는 Batch는 다음과 같은 특징이 있습니다.

* 대량의 데이터를 처리한다.

* 특정 시간에 프로그램을 실행한다.

* 일괄적으로 처리한다.

그리고 Batch는 특정 시간에 대량의 자료들을 모아 한 번에 처리하기 때문에

보통 다음과 같은 시스템에서 사용합니다.

* 성적 처리 시스템

* 급여 처리 시스템

* 통계 자료 처리 시스템

## 3. Batch의 장점

그러면 이런 의문이 들 수 있습니다.

> 실시간으로 처리하면 되는데 왜 굳이 일괄로 처리를 하는걸까?

일괄 처리 작업은 실시간 처리 작업에 비해 어떤 장점이 있을까요?

1. 처리 비용을 절감할 수 있다.

   데이터를 모아서 일괄로 처리하기 때문에 처리 비용을 절감할 수 있습니다.

   <br>

2. 시스템 이용 효율을 증대시킨다.

   특정 시간에 시스템을 일괄로 처리하기 때문에 컴퓨터 자원이 덜 사용되는 시간대에 동작하게 함으로써

   전체적으로 자원의 유휴(idle)를 피할 수 있습니다.

## 4. Batch의 단점

장점이 있다면 단점도 있겠죠?

이번에는 Batch의 단점을 알아보겠습니다.

1. 즉시 결과를 얻을 수 없다.

   Batch는 실시간으로 처리하는 것이 아닌 특정 시간에 한 번에 처리하기 때문에

   즉시 결과를 얻을 수 없습니다.

   <br>

2. 준비 작업이 필요하다.

   많은 양의 데이터를 수집 후 정리, 분류 작업이 필요하므로 준비 작업이 필요합니다.

## 5. Scheduler

앞에서 알아보았던 Batch는 데이터를 모아서 일괄로 처리합니다.

하지만, Batch는 대량의 데이터를 처리만 할 뿐 자동으로 정해진 시간에 동작하도록 설정해주진 않습니다.

그렇기 때문에 Batch는 특정 시간에 시스템을 수행하도록 설정하는 Scheduler와 함께 사용합니다.

우리는 Spring Batch와 Quartz라는 Batch Scheduler를 사용해 실습해보도록 하겠습니다.

→ [2편에서 계속]()

## 6. 참조

* [https://en.wikipedia.org/wiki/Batch_processing](https://en.wikipedia.org/wiki/Batch_processing){:target="_blank"}

* [https://gunggum2.tistory.com/135](https://gunggum2.tistory.com/135){:target="_blank"}

* [https://kkh0977.tistory.com/2909](https://kkh0977.tistory.com/2909){:target="_blank"}