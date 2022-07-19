---
title: Exception Handling (작성중)
categories: [Spring]
tags: ['ExceptionHandler', 'ControllerAdvice']
toc: true
secret: true

date: 2022-07-19
last_modified_at: 2022-07-19
---

## 1. 들어가기

프로젝트를 하다 보니 자연스럽게 예외를 처리해야 할 경우가 많아졌습니다.

그러다보니 어떻게 하면 유지 보수와 클린 코드 두 가지 토끼를 잡을 수 있을까 고민하던 중,

Spring 어노테이션인 ExceptionHandler와 ControllerAdvice를 알게 되어 이번 포스트를 작성합니다.

## 2. ExceptionHandler

[ExceptionHandler](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html){:target="_blank"}는 Controller 메서드나 ControllerAdvice 클래스의 메서드에서 사용할 수 있습니다.

