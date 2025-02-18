---
title: 골드스푼 파일 업로드 마이그레이션
categories: [Goldspoon]
tags: ['file upload', 'upload', '파일 업로드', '업로드']
toc: true

date: 2025-02-18
last_modified_at: 2025-02-18
---

## 1. 서론

골드스푼 4.13 스프린트에서 Stored Procedure 기반의 구 백엔드 서버에서 Spring Boot 기반의 신 백엔드 서버로 마이그레이션을 진행했습니다.

그 과정에서 제가 겪었던 시행착오들을 기록하여 파일 업로드를 개발하시는 다른 개발자들이 조금이라도 도움이 되었으면 하는 바램에 이 글을 남깁니다.

## 2. 골드스푼 서비스의 파일 종류

골드스푼에서 사용하는 파일은 크게 3가지로 일반 파일, Rive 파일, 서류 인증 파일입니다.

각 파일에 대해 간략히 설명하면 다음과 같습니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/goldspoon/file_upload_1.jpg" width="25%" height="1%" />
  <br><span class="span-img-comment">(프로필 파일 업로드)</span>
</div>

일반 파일은 유저들이 가장 많이 사용하는데 프로필을 포함하여 골드스푼의 라운지, 파티 등 대부분의 서비스에서 사용하는 이미지 파일입니다.
현재 모든 일반 파일은 public 버킷에 보관되어 있지만, 추후 프로필 파일은 개인정보 보호를 위해 private 버킷으로 이관할 예정입니다.

<div class="div-post-img">
  <video width="40%" height="5%" autoplay muted loop playsinline>
    <source src="{{ site.url }}/assets/media/goldspoon/file_upload_1.mp4" type="video/mp4">
  </video>
  <br><span class="span-img-comment">(Rive 파일 업로드)</span>
</div>

Rive 파일은 유저들이 직접 사용하지는 않고 주로 디자이너 분들이 작업해주신 움직이는 배너 파일로 이 또한 public 버킷에 관리하고 있습니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/goldspoon/file_upload_2.jpg" width="25%" height="1%" />
  <br><span class="span-img-comment">(서류 인증 파일 업로드)</span>
</div>

서류 인증 파일은 골드스푼의 가입 서류 파일로 연봉, 직업 정보 등 민감 정보를 가진 파일이므로 private 버킷에 관리하고 있습니다.

## 3. 파일 업로드 기능 개선 목표

기존 코드를 기반으로 그대로 마이그레이션을 진행하려 했는데 개선이 필요한 부분이 보였습니다.

그 부분은 바로 각 파일 업로드 내부 중복 프로세스를 줄이는 것이였습니다.

```java
  /** 일반 파일 업로드 */
  public void fileUpload() {
    // 1. 유효성 검증
    // 2. 전처리
    // 3. 원본 업로드
    // 4. 리사이징 업로드
    // 5. DB 저장
  }

  /** Rive 파일 업로드 */
  public void riveUpload() {
    // 1. 유효성 검증
    // 2. 전처리
    // 3. 원본 업로드
    // 4. DB 저장
  }

  /** 서류 인증 파일 업로드 */
  public void certificateUpload() {
    // 1. 유효성 검증
    // 2. 전처리
    // 3. 원본 업로드
    // 4. DB 저장
  }
```

이러한 구조는 한 프로세스를 수정할 때 모든 업로드 기능을 수정해야 한다는 단점이 있고 다른 종류의 파일 업로드가 추가되었을 때의 확장성도 좋지 않았습니다.

따라서 다른 업로드 기능이 추가되었을 때 중복 프로세스를 줄이고 확장성 있는 구조로의 변경을 목표로 삼았습니다.

## 4. 파일 업로드 구조 설계

파일 업로드 기능을 어떤 구조로 만들어야 프로세스의 중복 제거, 확장성 향상 이 두 마리의 토끼를 잡을 수 있을까 고민했습니다.

여러 자료들을 찾아보던 중 [한 블로그](https://lktprogrammer.tistory.com/35){:target="_blank"}에서 [브릿지 패턴](https://ko.wikipedia.org/wiki/%EB%B8%8C%EB%A6%AC%EC%A7%80_%ED%8C%A8%ED%84%B4){:target="_blank"}을 활용한 확장성 있는 구조를 발견했습니다. 

블로그에서는 로컬 스토리지, S3 등 다양한 저장소에 업로드할 수 있도록 `FileService`라는 공통 인터페이스를 정의하고, 각 저장소별로 하위 클래스를 구현하는 방식이었습니다.

하지만 골드스푼은 하나의 저장소만 사용하고 있고 업로드 과정에서 공통되는 로직이 많아 브릿지 패턴을 적용하면 오히려 중복 코드가 발생할 가능성이 크다고 판단했습니다.

이에 디자인 패턴에서 다른 대안을 찾던 중 **[책임 연쇄 패턴(Chain of Responsibility)](https://ko.wikipedia.org/wiki/%EC%B1%85%EC%9E%84_%EC%97%B0%EC%87%84_%ED%8C%A8%ED%84%B4){:target="_blank"}**을 발견했습니다.

책임 연쇄 패턴은 중앙 집권된 각 프로세스를 핸들러 객체로 분리하여 책임을 분산시키는 패턴으로 해당 패턴을 사용함으로써 유연하고 가독성이 좋은 장점을 얻을 수 있습니다.

```java
/** FileHandler 추상 클래스 */
  public abstract class FileHandler {
    protected ThreadLocal<FileHandler> nextHandler = new ThreadLocal<>();

    public FileHandler setNext(FileHandler nextHandler) {
        this.nextHandler.set(nextHandler);
        return nextHandler;
    }

    protected abstract void process(FileHandlerData fileData);

    public void exec(FileHandlerData fileData) {
      process(fileData);

      if (nextHandler.get() != null) {
        nextHandler.get().exec(fileData);
      }
    }
  }
```

위와 같은 추상 클래스를 상속 받아 각 프로세스에서 사용하고자 하는 핸들러를 상세하게 구현합니다.

저의 경우에는 `유효성 검증`, `전처리`, `원본 업로드`, `리사이징 업로드`, `DB 저장` 핸들러로 분리했습니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/goldspoon/file_upload_3.png" width="80%" height="50%" />
  <br><span class="span-img-comment">(FileHandler 구조)</span>
</div>

그리고 `ChainExecutor`를 두어 각 업로드 메서드에서 사용할 수 있도록 구현했습니다.

```java
/** 일반 파일 업로드 체인 실행자 */
  public FileHandler createFileUploadExecutor() {
    fileValidationHandler.setNext(filePreProcessingHandler)
                         .setNext(fileUploadHandler)
                         .setNext(fileResizingHandler)
                         .setNext(fileSaveHandler)
                         .setNext(null);
  }

/** Rive 파일 업로드 체인 실행자 */
  public FileHandler createRiveFileUploadExecutor() {
    fileValidationHandler.setNext(filePreProcessingHandler)
                         .setNext(fileUploadHandler)
                         .setNext(fileSaveHandler)
                         .setNext(null);
  }

/** 서류 인증 파일 업로드 체인 실행자 */
  public FileHandler createCertificateFileUploadExecutor() {
    fileValidationHandler.setNext(filePreProcessingHandler)
                         .setNext(fileUploadHandler)
                         .setNext(fileSaveHandler)
                         .setNext(null);
  }
```

`ChainExecutor` 사용은 다음과 같이 사용할 수 있습니다.

```java
/** 일반 파일 업로드 메서드 */
  public FileUploadResponse fileUpload() {
    chainExecutor.createFileUploadExecutor().exec();
  }

/** Rive 파일 업로드 메서드 */
  public FileUploadResponse riveUpload() {
    chainExecutor.createRiveFileUploadExecutor().exec();
  }

/** 서류 인증 파일 업로드 메서드 */
  public FileUploadResponse certificateUpload() {
    chainExecutor.createCertificateFileUploadExecutor().exec();
  }
```

이렇게 되면 각 프로세스를 재사용할 수 있고 만약 새로운 업로드 형식이 출시된다면 필요한 기능을 체인으로 연결하여 사용하면 확장성을 개선시킬 수 있습니다.

하지만 이렇게만 작성하면 문제점이 있습니다.

골드스푼은 각 업로드 메서드에서 허용하는 확장자나 파일 크기가 다른데 이렇게 작성하는 경우, 각 업로드 메서드에서 필요한 유효성 체크를 유연하게 처리하지 못한다는 점입니다.

따라서 각 업로드 메서드에서 유연하게 유효성 체크를 할 수 있도록 추가 기능이 필요한 상황입니다.

그래서 저는 유효성 체크 핸들러에서 [팩토리 메서드 패턴](https://ko.wikipedia.org/wiki/%ED%8C%A9%ED%86%A0%EB%A6%AC_%EB%A9%94%EC%84%9C%EB%93%9C_%ED%8C%A8%ED%84%B4){:target="_blank"}을 사용했습니다.

팩토리 메서드 패턴은 객체 생성을 캡슐화 처리하여 대신 생성해주는 패턴으로 해당 패턴을 사용함으로써 유연하게 객체를 생성할 수 있는 장점이 있습니다.

자, 그럼 유효성 체크 부분을 팩토리 메서드 패턴을 사용해 분리해보겠습니다.

먼저 원래 구체 클래스였던 `FileValidationHandler`를 추상 클래스로 변경합니다.

그리고 `FileValidationHandler`를 상속 받는 각 파일 유효성 검사 핸들러를 생성하고 유효성 체크 메서드를 구분할 추상 메서드도 하나 추가합니다.

```java
/** 파일 유효성 체크 핸들러 */
  @Component
  @RequiredArgsConstructor
  public abstract class FileValidationHandler extends FileHandler {
    public abstract FileAction fileAction();
  }
```

```java
/** 파일 업로드 Action 구분 */
  public enum FileAction {
    FILE_UPLOAD,        // 일반 파일 업로드 (프로필, 라운지, 파티, 케미, 모먼트)
    RIVE_UPLOAD,        // Rive 파일 업로드
    CERTIFICATE_UPLOAD  // 서류 인증 파일 업로드
  }
```

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/goldspoon/file_upload_4.png" width="80%" height="50%" />
  <br><span class="span-img-comment">(FileValidationHandler 구조)</span>
</div>

다음은 체인에서 각 파일 유효성 검사 핸들러를 연결해 줄 Factory 메서드를 생성합니다.

```java
/** 파일 유효성 체크 핸들러 팩토리 메서드 */
  @Component
  @RequiredArgsConstructor
  public class FileValidationHandlerFactory {
    private final List<FileValidationHandler> fileValidationHandlers;

    public FileValidationHandler create(FileAction action) {
      return fileValidationHandlers.stream()
                                   .filter(handler -> handler.fileAction() == action)
                                   .findFirst()
                                   .orElseThrow(() -> ApiException.of(NOT_FOUND_FILE_VALIDATOR));
    }
  }
```

마지막으로 체인 실행자의 유효성 체크 핸들러를 변경합니다.

```java
/** 일반 파일 업로드 체인 실행자 */
  public FileHandler createFileUploadExecutor() {
    FileValidationHandler fileValidationHandler = FileValidationHandlerFactory.create(FILE_UPLOAD);

    fileValidationHandler.setNext(filePreProcessingHandler)
                         .setNext(fileUploadHandler)
                         .setNext(fileResizingHandler)
                         .setNext(fileSaveHandler)
                         .setNext(null);
  }

/** Rive 파일 업로드 체인 실행자 */
  public FileHandler createRiveFileUploadExecutor() {
    FileValidationHandler fileValidationHandler = FileValidationHandlerFactory.create(RIVE_UPLOAD);

    fileValidationHandler.setNext(filePreProcessingHandler)
                         .setNext(fileUploadHandler)
                         .setNext(fileSaveHandler)
                         .setNext(null);
  }

/** 서류 인증 파일 업로드 체인 실행자 */
  public FileHandler createCertificateFileUploadExecutor() {
    FileValidationHandler fileValidationHandler = FileValidationHandlerFactory.create(CERTIFICATE_UPLOAD);

    fileValidationHandler.setNext(filePreProcessingHandler)
                         .setNext(fileUploadHandler)
                         .setNext(fileSaveHandler)
                         .setNext(null);
  }
```

이렇게 되면 각 파일 업로드에 맞게 유효성 체크를 하면서 동시에 확장성을 가진 구조를 가지게 됩니다.

## 5. 파일 업로드 예외 발생 시 회복

이번에는 파일 업로드 처리 도중 실패했을 때의 상황을 살펴보겠습니다.

예를 들어 트랜잭션이 묶여있는 환경에서 다음과 같이 두 번째 파일 정보를 DB에 저장할 때 예외가 발생한다면 어떻게 될까요?

```java
  fileValidationHandler.setNext(filePreProcessingHandler)
                       .setNext(fileSaveHandler1)         // 첫 번째 파일 정보 저장
                       .setNext(fileUploadHandler)        // 원본 파일 S3 업로드
                       .setNext(fileResizeUploadHandler)  // 리사이징 파일 S3 업로드
                       .setNext(fileSaveHandler2)         // 두 번째 파일 정보 저장 (예외 발생)
                       .setNext(null);
```

당연히 모든 일련의 과정은 트랜잭션으로 묶여있기에 첫 번째, 두 번째 파일 정보 모두 rollback이 되어 DB에 저장되지 않을 것입니다.

하지만 S3에 업로드한 파일들은 어떨까요?

아쉽게도 S3에 이미 업로드를 한 파일은 삭제되지 않고 그대로 남겨져있습니다.

그 이유는 S3가 트랜잭션에 영향을 받지 않는 서드파티이기 때문입니다.

그래서 저는 어떤 핸들러에서 예외가 발생했을 때의 회복 처리에 대해 고민했고 다음과 같이 처리했습니다.

먼저, FileHandler 추상 클래스에 회복 처리 default 메서드를 추가하고 각 핸들러에서 발생하는 예외를 잡기 위해 try-catch 문을 사용합니다.

```java
/** FileHandler 추상 클래스 */
  public abstract class FileHandler {
    ...

    void recovery(FileHandlerData fileData) {} // 회복 처리 default 메서드 추가

    public void exec(FileHandlerData fileData) { // try-catch 문 추가
        try {
          process(fileData);

          if (nextHandler.get() != null) {
              nextHandler.get().exec(fileData);
          }
        } catch (Exception exception) {
          try {
            recovery(fileData); // 예외 발생 시 회복 처리
          } catch (Exception recoveryException) {
            ...
          } finally {
            ...
          }
        }

    ...
  }
```

여기서 default 메서드의 사용에 대해 궁금할 수 있는데 default 메서드를 사용한 이유는 트랜잭션의 영향을 받는 특정 핸들러는 회복 처리가 불필요하므로
필요하지 않는 경우에는 따로 재정의하지 않기 위함입니다.

다시 돌아와서 예외 발생 시 회복 코드가 필요한 핸들러에서는 회복 메서드를 다음과 같이 재정의합니다.

```java
  public class FileUploadHandler extends FileHandler {
    ...

    @Override
    public void recovery(FileHandlerData fileData) {
      log.info("Recovery:::S3 객체(원본 데이터)를 삭제합니다.");
      boolean isNotDeleted = !s3Repository.deleteObject(...);

      if (isNotDeleted) {
        throw ApiException.of(FAIL_TO_RECOVERY_UPLOAD);
      }
    }

    ...
  }
```

이렇게 되면 파일정보 저장 핸들러에서 예외가 발생했을 때 다음과 같이 회복 코드가 동작합니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/goldspoon/file_upload_5.png" width="80%" height="50%" />
  <br><span class="span-img-comment">(회복 코드의 동작)</span>
</div>

## 6. 서버 관련 기타 이슈 사항

이번에는 서버 인프라 세팅을 하면서 발생했던 이슈 사항들을 공유하겠습니다.

### 🚩 [1MB 이상의 파일 업로드 시 413 에러 발생]

  MultipartFile을 업로드할 때 다음과 같이 413 에러가 발생했습니다.

  <div class="div-post-img">
    <img src="{{ site.url }}/assets/img/goldspoon/file_upload_6.png" width="80%" height="50%" />
    <br><span class="span-img-comment">(1MB 이상의 파일 업로드 시 413 에러 발생)</span>
  </div>

  해당 증상은 nginx에서 발생하는데 nginx의 기본 업로드 사이즈가 1MB이므로 발생합니다.

  > [관련 링크](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size){:target="_blank"}

  해결 방법은 `nginx.conf`에 다음과 같이 작성합니다.

  ```conf
  # nginx.conf
    http {
      ...
      
      # 클라이언트 요청에 허용된 최대 크기 조정
      client_max_body_size 100M;
    }
  ```

<br>

### 🚩 [body 데이터가 임시 파일로 생성되는 warning 발생]

  서버 로그를 보다 보니 임시 파일이 생성되었다는 warning이 발생했습니다.

  <div class="div-post-img">
    <img src="{{ site.url }}/assets/img/goldspoon/file_upload_7.png" width="80%" height="100%" />
    <br><span class="span-img-comment">(body 데이터가 임시 파일로 생성되는 warning 발생)</span>
  </div>

  해당 증상도 nginx에서 발생하는데 body 데이터가 버퍼 크기보다 큰 경우, 임시 파일에 기록되기 때문에 발생합니다.

  > [관련 링크](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size){:target="_blank"}

  해결 방법은 `nginx.conf`에 다음과 같이 작성합니다.

  ```conf
  # nginx.conf
    http {
      ...
      
      # 클라이언트 버퍼 크기 조정
      client_body_buffer_size 100M;
    }
  ```

<br>

### 🚩 [nginx.conf 파일 원상복귀 이슈 발생]

  위의 두 이슈를 처리하기 위해 `nginx.conf` 파일을 수정했습니다. 
  
  하지만, 재배포를 하는 경우 `nginx.conf` 파일이 원상복귀 되는 이슈가 발생했습니다.

  원인은 ElasticBeanstalk 배포 시 자동으로 기본 `nginx.conf`로 덮어쓰여지기 때문입니다.

  이러한 경우, ElasticBeanstalk의 플랫폼 훅을 추가하여 배포 시 nginx의 `conf.d` 디렉토리에 위의 두 세팅을 추가하여 include 함으로써 해결할 수 있습니다.

  <div class="div-post-img">
    <img src="{{ site.url }}/assets/img/goldspoon/file_upload_8.png" width="40%" height="20%" />
    <br><span class="span-img-comment">(ElasticBeanstalk 플랫폼 훅 적용)</span>
  </div>

  ```conf
  # nginx-custom.conf
    http {
      ...

      # 클라이언트 요청에 허용된 최대 크기 조정
      client_max_body_size 100M;

      # 클라이언트 버퍼 크기 조정
      client_body_buffer_size 100M;
    }
  ```

  > [관련 링크 1](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/platforms-linux-extend.example.html){:target="_blank"}
  >
  > [관련 링크 2](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/platforms-linux-extend.hooks.html){:target="_blank"}

## 7. 마지막

이번 골드스푼 파일 업로드 마이그레이션 작업을 하면서 몰랐던 새로운 지식들을 많이 알게 되었습니다.

그 과정에서 많은 코드 리뷰와 인사이트를 주신 백엔드 리더 관효님께 감사의 마음을 전합니다.