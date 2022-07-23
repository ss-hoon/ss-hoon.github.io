---
title: Exception Handling
categories: [Spring]
tags: ['ExceptionHandler', 'ControllerAdvice']
toc: true

date: 2022-07-19
last_modified_at: 2022-07-23
---

## 1. 들어가기

프로젝트를 하다 보니 자연스럽게 예외를 처리해야 할 경우가 많아졌습니다.

그러다보니 어떻게 하면 유지 보수와 클린 코드 두 가지 토끼를 잡을 수 있을까 고민하던 중,

Spring 어노테이션인 ExceptionHandler와 ControllerAdvice를 알게 되어 이번 포스트를 작성합니다.

## 2. ExceptionHandler

[ExceptionHandler](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html){:target="_blank"}는 컨트롤러 어노테이션이 붙어있는 Bean에서 발생하는 예외를 처리할 수 있습니다.

그렇기 때문에 Controller의 메서드나 ControllerAdvice 클래스의 메서드에서만 사용할 수 있습니다.

그럼 ExceptionHandler를 어떻게 사용할 수 있을까요?

예시를 통해 알아보겠습니다.

```java
  @Controller
  public class TestController {
    ...

    @ExceptionHandler
    public ResponseEntity handle(Exception e) {
      ...
    }
  }
```

다음과 같이 @Controller(@RestController)로 선언된 클래스에서 발생한 예외를

@ExceptionHandler에서 받아 처리할 수 있습니다.

주의해야 할 점은 ExceptionHandler가 작성된 컨트롤러에서 발생한 예외만 처리가능하다는 점입니다.

즉, 다른 컨트롤러에서 발생한 예외는 해당 메서드로 처리할 수 없습니다.

## 3. ExceptionHandler의 속성

때에 따라 특정 예외만 처리하고 싶은 경우가 있습니다.

그런 경우에는 어떻게 하면 될까요?

다행히도 ExceptionHandler는 value 속성을 가지고 있습니다.

이번에도 예시를 통해 알아보겠습니다.

```java
  @Controller
  public class TestController {
    ...

    @ExceptionHandler(value = NullPointerException.class) // value는 생략 가능합니다.
    public ResponseEntity handle(Exception e) {
      ...
    }
  }
```

다음과 같이 작성하게 되면 TestController에서 발생한 NullPointerException 예외만 처리할 수 있습니다.

즉, ArithmeticException과 같이 NullPointerException이 아닌 예외는 처리할 수 없습니다.

만약, 두 가지 이상의 예외를 한 ExceptionHandler에서 처리하고 싶을 경우는 다음과 같이 작성하면 됩니다.

```java
  @Controller
  public class TestController {
    ...

    @ExceptionHandler(value = { NullPointerException.class, ArithmeticException.class })
    public ResponseEntity handle(Exception e) {
      ...
    }
  }
```

추가로, 예외를 분리해서 처리하고 싶다면 내부에 ExceptionHandler 메서드를 여러개 생성하면 됩니다.

```java
  @Controller
  public class TestController {
    ...

    @ExceptionHandler(NullPointerException.class)
    public ResponseEntity handleNullPointerException(NullPointerException ne) {
      ...
    }

    @ExceptionHandler(ArithmeticException.class)
    public ResponseEntity handleArithmeticException(ArithmeticException ae) {
      ...
    }
  }
```

## 4. ControllerAdvice

위에서 본 예시의 문제점이 무엇일까요?

바로, 컨트롤러에 예외처리 로직까지 더해져 코드가 지저분하게 된다는 것입니다.

그래서 보통 컨트롤러에서 예외처리 로직을 분리시키는데 이때 [ControllerAdvice](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html){:target="_blank"} 어노테이션을 사용합니다.

사용법도 함께 알아볼까요?

```java
  @ControllerAdvice
  public class TestControllerAdvice {
    ...
  }
```

다음과 같이 클래스 레벨에서 ControllerAdvice 어노테이션을 붙여주기만 하면 됩니다.

그러나 이렇게 한다면 모든 컨트롤러에서 발생한 예외가 해당 클래스로 들어와 유지 보수가 어려워집니다.

그럼 어떻게 해야 예외처리 로직을 분리하면서 유지 보수도 쉽게 할 수 있을까요?

## 5. ControllerAdvice의 속성

이를 해결하기 위해 ControllerAdvice는 속성을 제공합니다.

속성은 지정하고 싶은 패키지, 클래스, 어노테이션에 따라 다르게 사용합니다.

* 패키지를 지정하고 싶은 경우

```java
  @ControllerAdvice(value = "test.package")
  public class TestControllerAdvice {
    ...
  }
```

```java
  @ControllerAdvice(basePackages = "test.package")
  public class TestControllerAdvice {
    ...
  }
```

* 특정 클래스를 기준으로 패키지를 지정하고 싶은 경우

```java
  @ControllerAdvice(basePackageClasses = TestController.class)
  public class TestControllerAdvice {
    ...
  }
```

* 특정 클래스를 지정하고 싶은 경우

```java
  @ControllerAdvice(assignableTypes = TestController.class)
  public class TestControllerAdvice {
    ...
  }
```

* 어노테이션을 지정하고 싶은 경우

```java
  @ControllerAdvice(annotations = Controller.class)
  public class TestControllerAdvice {
    ...
  }
```

## 6. RestControllerAdvice

@RestControllerAdvice는 @ControllerAdvice와 @ResponseBody를 합친 어노테이션입니다.

그렇기 때문에 ControllerAdvice와 동일한 역할을 수행하면서 예외를 body에 담아 반환할 수 있습니다.

* [ControllerAdvice] vs [RestControllerAdvice]

```java
/* ControllerAdvice 예시 */
  @ControllerAdvice
  public class TestControllerAdvice {
      @ExceptionHandler(NullPointerException.class)
      public String handleNullPointerException(NullPointerException ex) {
          return "Body Data";
      }
  }
```

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/spring/exception-handler/controlleradvice-error.png" width="90%" height="50%" />
</div>

```java
/* RestControllerAdvice 예시 */
  @RestControllerAdvice
  public class TestControllerAdvice {
      @ExceptionHandler(NullPointerException.class)
      public String handleNullPointerException(NullPointerException ne) {
          return "Body Data";
      }
  }
```

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/spring/exception-handler/controlleradvice-ok.png" width="90%" height="50%" />
</div>

## 7. 정리

이번 포스트는 ExceptionHandler와 ControllerAdvice를 통해 예외 처리하는 방법을 알아보았습니다.

기존에 사용하던 try-catch 문은 유지보수가 어렵고 코드를 어지럽게 만드는 문제점이 있었던 반면,

ExceptionHandler와 ControllerAdvice 어노테이션은 앞의 두 문제점을 해결할 수 있기 때문에

Spring에서 예외 처리를 할 때는 되도록 두 어노테이션을 사용해 예외를 처리해봅시다.

만약, 예외를 반환하고 싶다면 ControllerAdvice 대신 RestControllerAdvice를 사용하면 됩니다.