---
title: 호이스팅(Hoisting)
categories: [JavaScript]
tags: [Hoisting]
toc: true

date: 2022-07-05
last_modified_at: 2022-07-10
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

이러한 정보의 모든 "선언"을 스코프에 등록합니다.

그래서 실행하기 전 스코프에 등록하는 것이 마치 끌어올리는 것처럼 보이기 때문에 hoisting 이라고 합니다.

주의해야할 점은 실제로 코드가 끌어올려지는 것이 아니라 자바스크립트 내부적으로 처리하는 것이기에

실제 메모리에서는 변화가 없습니다.

## 3. 함수 Hoisting 과정

함수 Hoisting은 어떻게 이뤄지는 걸까요?

실행 컨텍스트 생성의 Creation Phase 단계에서 함수가 통째로 컨텍스트 최상단 스코프에 등록됩니다.

주의해야할 점으로 함수는 함수 표현식과 함수 선언식으로 사용할 수 있고,

이 중, 함수 표현식은 Hoisting의 영향을 받지 않고 함수 선언식만 Hoisting의 영향을 받습니다.

(함수 표현식으로 정의한 함수는 변수 Hoisting이 발생합니다.)

```javascript
  /* 함수 표현식 */
  var func1 = function() {}

  /* 함수 선언식 */
  function func2() {}
```

## 4. 변수 Hoisting 과정

그럼 변수 Hoisting은 어떻게 이뤄지는 걸까요?

변수 Hoisting은 총 3단계에 걸쳐 이뤄집니다.

1. 선언 단계
   
   Lexical Environment는 식별자(identifier)와 값(variable)이 매핑된 자료구조 형태이고,

   Lexical Environment 내부에는 식별자의 정보를 저장하는 Environment Record가 있습니다.

   Environment Record가 컨텍스트 내부를 처음부터 끝까지 쭉 훑어나가며 식별자 정보를 수집합니다.

   여기서 식별자 정보란, 변수의 선언을 뜻합니다.

2. 초기화 단계

   Lexical Environment에 담긴 변수를 위한 공간을 메모리에 확보합니다.

   이 단계에서 var 키워드의 변수는 undefined으로 초기화되지만,
   
   let과 const 키워드의 변수는 초기화되지 않습니다.

   (그렇다고 let과 const가 Hoisting 되지 않는다는 말은 아닙니다.)

   이 때의 let과 const는 Temporal Dead Zone(TDZ) 이라고 합니다.

3. 할당 단계

   Lexical Environment의 variable에 값이 할당됩니다.

## 5. Hoisting 예시

앞서 Hoisting의 이론을 알아보았으니 이번에는 예시를 통해 이론을 정리해봅시다.

* 함수 Hoisting (함수 선언식)

```javascript
  a(1, 2);

  function a(x, y) {
    return x + y;
  }
```

```
  3
```

* 함수 Hoisting (함수 표현식)

```javascript
  a(1, 2);

  var a = function(x, y) {
    return x + y;
  }
```

```
  a is not a function
```

* 변수 Hoisting (var 키워드)

```javascript
  console.log(a);
  var a = 10;
```

```
  undefined
```

* 변수 Hoisting (let 키워드)

```javascript
  console.log(a);
  let a = 10;
```

```
  a is not defined
```

## 6. 정리

이번 포스트는 호이스팅에 대해 알아보았습니다.

호이스팅은 자바스크립트 엔진이 실행 컨텍스트를 생성하는 과정에서

실행에 필요한 정보(변수, 함수 등)의 선언을 해당 스코프의 최상단으로 끌어올리는 것을 말합니다.

이러한 성질 때문에 초기화 전에 해당 식을 호출하거나 변수를 사용해도 에러는 나지 않습니다.

하지만 호이스팅으로 인해 의도했던 동작과 다른 결과로 이어질 수 있기 때문에

선언 이전에 어떤 값에 접근하려 할때는 명확한 기준 하에 접근해야 합니다.