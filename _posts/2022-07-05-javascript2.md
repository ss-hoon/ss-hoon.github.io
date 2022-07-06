---
title: 호이스팅(Hoisting) (작성중)
categories: [JavaScript]
tags: [Hoisting]
toc: true
secret: true

date: 2022-07-05
last_modified_at: 2022-07-06
---

## 1. 서론

사내에서 JavaScript 교육 받았던 내용을 리마인드합니다.

이번 포스트는 호이스팅(Hoisting)에 대해 다루겠습니다.

## 2. Hoisting

Hoisting이란 무엇일까요?

먼저, Hoist의 사전적 의미부터 알아보겠습니다.

> to lift something heavy, sometimes using ropes or a machine:

이를 해석하면, '줄이나 기계로 무거운 어떤 것을 끌어올리다.' 라는 의미입니다.

우리는 '끌어올리다' 에 초점을 맞춰봅시다.

도대체 무엇을 끌어올리는 걸까요?

앞서 우리는 [실행 컨텍스트](/javascript/javascript1){:target="_blank"}에 대해 알아보았습니다.

자바스크립트의 실행 컨텍스트는 실행되기 전, 실행에 필요한 정보(변수, 함수 등)를 미리 세팅하는데

이러한 정보의 모든 선언을 스코프에 등록합니다.

그래서 실행하기 전 스코프에 등록하는 것이 마치 끌어올리는 것처럼 보이기 때문에 hoisting 이라고 합니다.

주의해야할 점은 실제로 코드가 끌어올려지는 것이 아니라 자바스크립트 내부적으로 처리하는 것이기에

실제 메모리에서는 변화가 없습니다.

## 3. 변수 Hoisting

...