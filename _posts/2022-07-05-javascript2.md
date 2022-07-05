---
title: 호이스팅(Hoisting)
categories: [JavaScript]
tags: [Hoisting]
toc: true
secret: true

date: 2022-07-05
last_modified_at: 2022-07-05
---

## 1. 서론

사내에서 JavaScript 교육 받았던 내용을 리마인드합니다.

이번 포스트는 호이스팅(Hoisting)에 대해 다루겠습니다.

## 2. Hoisting

Hoisting이란 무엇일까요?

먼저, Hoist의 사전적 의미부터 알아보겠습니다.

> to lift something heavy, sometimes using ropes or a machine:

이를 해석하면, '줄이나 기계로 무거운 어떤 것을 끌어올리다.' 라는 의미입니다.

우리는 '끌어올리다' 라는 의미에 초점을 맞춰봅시다.

어떤 것을 끌어올리는 걸까요?

자바스크립트는 실행되기 전, 변수나 함수를 모두 모아서 유효 범위의 최상단에 선언합니다.