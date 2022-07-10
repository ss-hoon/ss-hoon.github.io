---
title: 실행 컨텍스트 (Execution Context)
categories: [JavaScript]
tags: [Execution Context]
toc: true

date: 2022-06-07
last_modified_at: 2022-06-08
---

## 1. 서론

사내에서 받은 JavaScript 교육 내용을 리마인드합니다.

이번 포스트는 실행 컨텍스트(Execution Context)에 대해 다루겠습니다.

## 2. 실행 컨텍스트 (Execution Context)

실행 컨텍스트가 무엇일까요?

ECMAScript 명세에서 실행 컨텍스트에 대한 내용을 찾아보면 다음과 같습니다.

> An execution context is a specification device that is used to track the runtime evaluation of code by an ECMAScript implementation.

이를 해석하면 다음과 같습니다.

> 실행 컨텍스트는 ECMA스크립트 런타임 구현 코드의 평가를 추적하는 데 사용되는 규격 장치이다.

여기서 코드 평가라는 것은 '실행에 필요한 정보를 세팅한다'라는 의미이고

실행 컨텍스트는 실행 가능한 코드가 실행되기 위해 필요한 환경이라고 생각할 수 있습니다.

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

실행 컨텍스트의 내부는 어떻게 구성되어 있을까요?

실행 컨텍스트는 세 가지의 property를 소유합니다.

* Variable Environment (VE)

  현재 컨텍스트 내의 식별자들에 대한 정보와 외부 환경 정보가 담기고 변경사항은 반영되지 않습니다.

  ES6에서는 변수 var만 저장됩니다.

* Lexical Environment (LE)

  Variable Environment와 동일하지만 변경사항이 실시간으로 반영됩니다.

  ES6에서는 변수 let과 const가 저장됩니다.

* ThisBinding

  this의 값 할당이 여기서 결정됩니다.

  전역 실행 컨텍스트에서 this는 전역 객체(window)입니다.

  함수 실행 컨텍스트에서 this는 함수가 객체 참조에 의해 호출될 경우, 해당 객체로 설정되지만

  그렇지 않은 경우, 전역 객체(window)를 가리키거나 strict mode에서는 undefined를 가리킵니다.

🎯[추가]
  
ES5와 ES6 이후부터는 ThisBinding의 위치가 달라졌습니다.

ES5에서의 실행 컨텍스트 내부에는 VE, LE, ThisBinding 세 가지의 property가 존재했지만,

ES6 이후부터 실행 컨텍스트 내부에는 VE, LE만 존재하고

각 VE와 LE 내부에 ThisBinding이 존재하게 되었습니다.

그 이유는 ES6부터 let과 const가 생겨나 기존의 var와 구별할 필요가 있기 때문입니다.

(함수 레벨 스코프 VS 블록 레벨 스코프)

## 6. 실행 컨텍스트 생성 과정

실행 컨텍스트는 Creation과 Execution 과정을 통해 생성됩니다.

* Creation Phase

  Variable Environment과 Lexical Environment, ThisBinding에 초기화가 이루어집니다.

* Execution Phase

  자바스크립트 엔진이 코드를 실행시키고, 식별자에 변수를 할당합니다.

## 7. 정리

이번 포스트는 실행 컨텍스트에 대해 알아보았습니다.

실행 컨텍스트는 실행 가능한 코드가 실행되기 위해 필요한 환경입니다.

자바스크립트의 모든 코드 실행은 실행 컨텍스트로부터 시작되기 때문에

좋은 자바스크립트 개발자가 되기 위해서는 반드시 알고 넘어갑시다.