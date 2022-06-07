---
title: 실행 컨텍스트 (Execution Context) - 작성중
categories: [JavaScript]
tags: [Execution Context]
toc: true

date: 2022-06-07
last_modified_at: 2022-06-07
---

## 1. 서론

사내에서 JavaScript 교육 받았던 내용을 리마인드합니다.

이번 포스트는 실행 컨텍스트(Execution Context)에 대해 다루겠습니다.

## 2. 실행 컨텍스트 (Execution Context)

실행 컨텍스트가 무엇일까요?

쉽게 말해서 실행 환경으로 실행 가능한 코드가 실행되기 위해 필요한 환경이라고 생각하면 됩니다.

이 실행 컨텍스트는 크게 두가지로 나눌 수 있습니다.

* 전역 실행 컨텍스트 (Global Execution Context)

  전역 실행 컨텍스트는 무조건 하나만 존재하고 애플리케이션이 종료될 때까지 유지됩니다.

  코드가 실행되기 전에 생성되고, 함수 내에 없는 코드는 모두 전역 실행 컨텍스트에 존재하게 됩니다.

* 함수 실행 컨텍스트 (Functional Execution Context)

  전역 실행 컨텍스트가 생성된 후, 각 함수가 실행될 때마다 생성되는 실행 컨텍스트를 말합니다.

## 3. 실행 컨텍스트가 형성되는 경우

이 실행 컨텍스트가 형성되는 경우는 세 가지로 규정합니다.

1. 전역 코드 (전역 영역에 존재하는 코드)
   
2. eval 코드 (eval 함수로 실행되는 코드)

3. 함수 코드 (함수 내에 존재하는 코드)

## 4. 실행 컨텍스트 스택

실행 컨텍스트들은 생성과 소멸될 때 Stack으로 관리되는데 이를 실행 컨텍스트 스택이라고 합니다.

```javascript
  function outer() {
    function inner() {
      ...
    }
    inner();
  }
  outer();
```

위 코드를 실행하면 실행 컨텍스트 스택의 상태는 다음과 같습니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/javascript/execution_context.png" width="80%" height="30%" />
</div>

1. 처음 자바스크립트가 실행하게 되면 전역 실행 컨텍스트가 스택에 담깁니다.

   해당 전역 실행 컨텍스트는 애플리케이션이 종료될 때까지 유지됩니다.

2. outer 함수가 호출되면 outer 함수에 대한 환경을 수집해 새로운 실행 컨텍스트를 생성하고

   그 실행 컨텍스트를 스택에 쌓습니다.

3. outer 함수가 실행되다가 inner 함수를 호출하면 inner 함수에 대한 환경을 수집하고

   새로운 실행 컨텍스트를 생성해 스택에 쌓습니다.

4. inner 함수가 실행이 종료되었을 경우 inner 함수 실행 컨텍스트는 스택에서 제거됩니다.

5. outer 함수가 실행이 종료되었을 경우 outer 함수 실행 컨텍스트는 스택에서 제거됩니다.

> 스택의 최상단에 있는 실행 컨텍스트가 현재 실행할 코드다.

## 5. 실행 컨텍스트 구성

<!-- 실행 컨텍스트의 내부는 어떻게 구성되어 있을까요?

실행 컨텍스트는 두 가지의 property를 소유합니다.

* Lexical Environment
  
  해당 컨텍스트에서 선언된 변수나 함수들의 Reference 값을 저장합니다.

  Lexical Environment는 두 가지의 요소를 가지고 있습니다.

  * Environment Record

    Lexical Environment 안에 함수와 변수 선언을 저장하는 곳입니다.

    * Declarative Environment Record

      변수와 함수 선언을 저장합니다.

    * Object Environment Record


  * Outer Environment Reference

    Lexical Scope를 기준으로 상위 Scope의 Lexical Environment를 참조합니다.

    전역 실행 컨텍스트의 경우는 null 값을 가집니다.

* Variable Environment -->