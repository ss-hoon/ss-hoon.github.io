---
title: 골드스푼 동영상 업로드 기능 1 - presigned URL
categories: [Goldspoon]
tags: ['upload', '동영상 업로드', '업로드', 'presigned', 'presigned url', 'presigned-url', 'S3']
toc: true

date: 2025-04-24
last_modified_at: 2025-05-28
---

## 1. 서론

골드스푼 4.15 스프린트에서는 프로필 동영상 기능을 출시했습니다.

프로필 동영상 기능을 출시하면서 크게 두 가지를 적용했는데 새로운 파일 업로드 방식 도입과 private 버킷 데이터를 조회하기 위한 AWS 인프라 환경 구축입니다.

이번 포스트에서 다뤄볼 주제는 새로운 파일 업로드 방식인 Presigned URL에 대해 다뤄보겠습니다.

## 2. Presigned URL을 도입하게 된 이유

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url12.png" width="80%" height="40%" /><br>
  <p>기존 파일 업로드 동작 방식</p> 
</div>

기존 파일 업로드 기능에서는 이미지의 저장공간으로 활용되던 S3 버킷에 파일 업로드하기 위한 절차로 서버에서 액세스 키와 시크릿 키를 검증을 진행한 후 파일을 업로드 했습니다.

하지만 큰 용량의 파일을 업로드하기 위해서는 클라이언트에서 받은 이미지를 서버 메모리에 한번 업로드를 해야 하기에 서버에 큰 부담이 될 수 있습니다.

그래서 백엔드 리더이신 관효님께서 Presigned URL 업로드 방식을 제안주셨고 하나씩 공부해보며 적용하기로 했습니다.

Presigned URL은 말 그대로 미리 서명된 URL이라는 의미로 서버에서 파일을 업로드하는 방식이 아니라 서버에서 서명된 URL을 클라이언트에 넘겨주고 클라이언트가 직접 해당 URL을 통해 파일을 업로드하는 방식입니다.

즉, Presigned URL을 적용하게 되면 서버의 자원을 절약함과 동시에 사용자 경험을 향상시킬 수 있는 장점이 있습니다.

그래서 서버의 부담을 줄이고 유저의 사용성을 개선하고자 이번 4.15 프로필 동영상 기능에 적용하게 되었습니다.

## 3. Presigned URL을 통한 업로드 동작 방식

이번에는 Presigned URL을 통해 어떻게 S3 버킷에 업로드할 수 있는지 알아보겠습니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url1.webp" width="80%" height="40%" /><br>
  <p>Presigned URL 업로드 동작 방식 <a href="https://opstree.com/blog/2024/08/06/uploading-files-using-pre-signed-urls-to-a-specific-storage-class" target="blank">[출처]</a></p> 
</div>

> 1. 클라이언트는 서버에 파일 업로드 request를 보냅니다. 이때 파일을 보내지 않습니다.
>
> 2. 클라이언트의 요청을 받은 서버는 AWS S3 서비스에 인증 정보와 함께 서명 요청합니다.
>
> 3. AWS S3 서비스는 인증 정보가 유효한지 판단하고 유효하다면 서명된 URL을 반환합니다.
>
> 4. AWS S3 서비스로부터 받은 서명된 URL을 클라이언트에 반환합니다.
>
> 5. 클라이언트는 해당 URL로 AWS S3 버킷에 파일을 업로드합니다.

이때, 한 가지 우려되는 점이 있는데 클라이언트가 인증된 URL을 가지고 있다는 것입니다.

보안상 위험할 수 있는 부분인데 그렇기에 무한정 URL을 통해 업로드할 수 없도록 유효 기간을 지정합니다.

## 4. 공식 문서 코드를 통해 Presigned URL을 사용하여 파일 업로드 해보기

이번에는 공식 문서 코드를 통해 Presigned URL을 사용한 파일 업로드를 해보겠습니다.

준비물은 Spring Boot 환경과 AWS 계정, 테스트 이미지 그리고 API 테스트 도구(Postman) 입니다.

먼저, 파일을 업로드할 S3 버킷을 생성합니다. 

이때 테스트를 위해 버킷 타입을 public으로 지정하겠습니다. 

![S3 버킷 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url2.png)

그리고 계정의 Key를 생성해야 하는데 IAM 사용자 페이지의 보안 자격 증명 탭에서 생성할 수 있습니다.

생성할 때 나오는 사용 사례는 `AWS 외부에서 실행되는 애플리케이션`을 선택하여 생성합니다.

Key를 생성하면 csv 파일을 다운 받을 수 있는데 페이지를 벗어나면 다시 볼 수 없으니 다운로드 해놓습니다.

![AWS Access Key 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url4.png)

그럼 AWS 세팅은 끝입니다! 참 쉽죠? 😀

이제 Spring Boot 환경을 세팅해보겠습니다.

초기 세팅은 간단하게 `Spring Web` 만 지정합니다.

![Spring Boot 환경 세팅](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url3.png)

생성한 프로젝트에 있는 `build.gradle` 파일에서 AWS SDK dependency를 추가합니다.

```groovy
  implementation "software.amazon.awssdk:s3:2.20.0"
```

마지막으로 Presigned URL을 받을 수 있도록 컨트롤러와 서비스를 간단하게 생성합니다.

```java
  @RestController
  @RequestMapping("/upload")
  public class UploadController {
    private final UploadService uploadService;

    public UploadController(UploadService uploadService) {
      this.uploadService = uploadService;
    }

    @GetMapping("")
    public ResponseEntity<String> getPresignedUrl() {
      return ResponseEntity.ok(uploadService.getPresignedUrl());
    }
  }
```

```java
  @Service
  public class UploadService {
    public String getPresignedUrl() {
        return "";
    }
  }
```

이렇게 하면 모든 Spring Boot 환경 세팅은 끝입니다.

그럼 이제부터 AWS SDK를 통해 Presigned URL을 조회해보겠습니다.

코드는 [AWS 공식문서](https://docs.aws.amazon.com/AmazonS3/latest/API/s3_example_s3_Scenario_PresignedUrl_section.html){:target="_blank"}를 참조했습니다.

```java
  public String getPresignedUrl() {
    String bucketName = "bucket-name";  // S3 버킷 이름
    String keyName = "key-name";        // 버킷 내 경로 및 파일명

    try (S3Presigner presigner = S3Presigner.create()) {
      PutObjectRequest objectRequest = PutObjectRequest.builder()
                                                       .bucket(bucketName)
                                                       .key(keyName)
                                                       .build();

      PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                                                                      .signatureDuration(Duration.ofMinutes(10))
                                                                      .putObjectRequest(objectRequest)
                                                                      .build();


      PresignedPutObjectRequest presignedRequest = presigner.presignPutObject(presignRequest);

      return presignedRequest.url().toExternalForm();
    }
  }
```

코드를 한 부분씩 보겠습니다.

먼저, Presigned URL 조회를 위해 가장 중요한 `S3Presigner`입니다.

```java
  try (S3Presigner presigner = S3Presigner.create())
```

S3Presigner 클래스 내부로 진입하면 다음과 같이 작성되어 있습니다.

![S3Presigner 내부 코드](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url5.png)

해석하면 DefaultAwsRegionProviderChain에서 로드되고 자격 증명은 DefaultCredentialsProvider에서 로드된다면서
그와 함께 각 순서에 맞게 인증이 이루어진다고 되어 있습니다.

![DefaultCredentialsProvider](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url6.png)

저는 간단하게 2번의 환경 변수를 통해 설정해보겠습니다.

IntelliJ 기준 우측 상단에 있는 Edit Configurations에 진입하면 다음과 같이 나오는데 Environment variables 위치에 `AWS_ACCESS_KEY`와 `AWS_SECRET_ACCESS_KEY`를 입력합니다.

![Environment Variable](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url8.png)

다음은 presigned URL을 사용하여 파일 업로드할 때의 버킷의 이름과 버킷 내 경로 그리고 파일명을 설정하는 부분입니다.

```java
  PutObjectRequest objectRequest = PutObjectRequest.builder()
                                                 .bucket(bucketName)
                                                 .key(keyName)
                                                 .build();
```

추후에 파일 업로드를 하면 해당 경로로 저장되게 됩니다.

다음은 앞서 등록한 정보와 함께 서명된 URL의 유효 기간을 설정하는 부분입니다.

```java
  PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                                                                  .signatureDuration(Duration.ofMinutes(10))
                                                                  .putObjectRequest(objectRequest)
                                                                  .build();
```

유효 기간은 보안을 위해 오랜 기간으로 설정하지 않도록 합니다.

다음은 실제로 AWS S3 서비스 API 호출을 하는 부분입니다.

```java
  PresignedPutObjectRequest presignedRequest = presigner.presignPutObject(presignRequest);
```

해당 API의 결과로 우리가 원하는 presigned URL을 받을 수 있습니다.

자, 그럼 모든 준비는 완료되었습니다.

이제 API 테스트 도구로 파일을 업로드 해보겠습니다.

API 테스트 도구는 여러 가지가 있고 그 중 아무거나 사용하면 됩니다. 저는 postman을 사용하겠습니다.

앞서 생성했던 `UploadController` 경로로 호출합니다.

그럼 아래와 같이 Presigned URL을 받을 수 있습니다.

![Get presigned url](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url9.png)

URL을 복사하고 새로운 request 창을 열어 해당 URL을 붙여넣기 합니다.

이 때 가장 중요한 것은 "PUT 메서드"로 호출하는 것입니다. 다른 메서드로 호출하면 정상적으로 나오지 않으니 주의해주세요.

그리고 body 데이터에 binary 형태로 테스트 이미지를 업로드합니다.

정상적으로 업로드가 된 경우 아래와 같이 200 코드에 빈 값으로 나옵니다.

![File upload](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url10.png)

만약, 시그니처 관련 오류가 나오는 경우, AWS에서 발급한 액세스 키나 시크릿 키를 확인해주세요.

그럼 이제 실제로 S3에 업로드 되었는지 확인해봅시다.

![Confirm Image](https://d36u0n6bmvvikl.cloudfront.net/blog/presigned-url/presigned_url11.png)

우리가 원하는 버킷과 경로에 설정한 파일명으로 업로드 된 것을 볼 수 있습니다.

## 5. 퍼블릭 액세스가 차단된 버킷의 접근

이렇게 하면 퍼블릭 액세스가 열려있는 버킷에 저장한 이미지를 외부에서 손쉽게 접근할 수 있습니다.

하지만 프로필 이미지나 동영상의 경우는 개인정보이므로 외부에서의 접근을 막기 위해 퍼블릭 액세스를 차단합니다.

그럼 이런 경우에 어떤 방법으로 버킷 파일을 조회할 수 있을까요?

그 방법은 [다음](../private_bucket){:target="_blank"} 포스트에서 다룰 예정입니다.