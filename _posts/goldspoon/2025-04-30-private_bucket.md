---
title: 골드스푼 동영상 업로드 기능 2 - private 버킷 조회
categories: [Goldspoon]
tags: ['upload', '동영상 업로드', '업로드', 'lambda@edge', 'S3', 'cloudfront']
toc: true

date: 2025-04-30
last_modified_at: 2025-04-30
---

## 1. 서론

골드스푼 4.15 스프린트에서는 프로필 동영상 기능을 출시했습니다.

프로필 동영상 기능을 출시하면서 크게 두 가지를 적용했는데 [새로운 파일 업로드 방식 도입](../presigned_url){:target="_blank"}과 private 버킷 데이터를 조회하기 위한 AWS 인프라 환경 구축입니다.

이번에는 private 버킷에 있는 파일을 조회하는 방법을 공유하면서 그와 함께 AWS 환경에서 인프라를 간단하게 구축해보겠습니다.

## 2. 어떻게 private 버킷의 파일을 조회할 수 있을까?

public 버킷에 있는 파일들은 손쉽게 URL을 통해 접근할 수 있습니다.

하지만 private 버킷에 있는 파일은 URL을 통해 접근하면 다음과 같이 `Access Denied` 메시지가 뜹니다.

![private 버킷 조회 시 권한 에러](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket1.png)

그럼 private 버킷의 파일은 영영 볼 수 없는 걸까요?

이 때 해결할 수 있는 방법은 CloudFront를 사용하는 것입니다.

즉, S3 버킷으로의 직접 접근을 막고 외부에서는 CloudFront를 통해서 S3 버킷에 접근할 수 있도록 인프라를 구축할 수 있습니다.

하지만 단순하게 CloudFront만 사용하게 되면 경로만 하나 더 추가된 것이지 사실상 public 버킷과 다름없어 모든 사용자가 CloudFront를 통해 S3 버킷에 접근할 수 있습니다.

그래서 인증된 특정 사용자만 접근할 수 있도록 인증 시스템을 만들어야 합니다.

어떤 방법으로 만들 수 있을까요?

먼저 우리 서버들이 어떻게 사용자를 인증하는지 살펴봅시다.

대부분의 서버들은 인증(또는 인가까지) 절차를 통해 특정 사용자만 자원을 가져갈 수 있도록 합니다. 각 서버마다 인증 처리 방식이 다릅니다만, Spring Boot 환경에서는 보통 필터 기반으로 인증을 처리합니다.

AWS 환경에서도 비슷하게 처리할 수 있습니다. 바로 Lambda라는 서비스를 통해 말이죠

여기서 서버는 CloudFront이고 필터는 Lambda라고 생각하면 됩니다.

CloudFront에 Lambda 트리거를 설정하고 특정 request가 들어오면 CloudFront에 도달하기 전 Lambda가 가로채 인증된 사용자만 S3 버킷에 접근할 수 있도록 만들 수 있습니다.

이때 중요한 것은 일반 Lambda가 아닌 CloudFront의 엣지 로케이션(Edge Location)에서 사용할 수 있는 Lambda@Edge를 사용합니다.

그리고 보안 측면에서 가장 중요한 인증키는 Lambda 내부에서 가지고 있는 것이 아닌 AWS ParameterStore에서 관리하도록 합니다.

위 설명을 그림으로 간단하게 요약하면 다음과 같습니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket2.png" width="50%" height="30%" />
</div>

이번 포스트에서는 위 그림의 인프라를 구축해보겠습니다.

## 3. 외부에서 S3 버킷으로 바로 접근할 수 없도록 private 버킷으로 만들기

먼저 첫 번째로 해야할 일은 이전에 만들었던 S3 버킷을 private 버킷으로 전환해야 합니다.

private 버킷으로 만드는 방법은 아래와 같이 "모든 퍼블릭 엑세스 설정"을 선택하여 설정할 수 있습니다.

![private 버킷 전환1](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket3.png)

![private 버킷 전환2](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket4.png)

## 4. CloudFront를 생성해 private 버킷과 연결하기

그다음 해야할 일은 CloudFront를 생성해 해당 S3와 연결합니다.

AWS 페이지 상단 검색칸에 CloudFront를 검색하고 배포 생성을 누르면 다음과 같이 나옵니다.

![CloudFront 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket5.png)

Origin domain으로 S3 버킷을 선택하고 Origin path는 S3 내부 URL과 동일하게 사용할 것이므로 비워둡니다.

그리고 가장 중요한 OAI(Origin Access Identity) 또는 OAC(Origin Access Control)를 설정합니다.

OAI와 OAC에 대해 간략하게 설명하면 둘 다 CloudFront에서 S3 버킷에 대한 접근을 제어하는 보안 방식인데 OAC는 OAI보다 조금 더 세부적인 보안 설정이 가능합니다.

OAC에 대한 자세한 내용은 [공식문서](https://aws.amazon.com/ko/blogs/korea/amazon-cloudfront-introduces-origin-access-control-oac/){:target="_blank"}를 참조하세요!

여기서 OAI를 설정하려면 `Legacy access identities`, OAC를 설정하려면 `원본 액세스 제어 설정(권장)`을 선택하면 되는데 저희는 권장 방법인 `원본 액세스 제어 설정(권장)`을 선택하겠습니다.

Create new OAC를 선택하여 OAC를 생성합니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket9.png" width="50%" height="30%" />
</div>

![CloudFront 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket6.png)

## 5. JWT를 복호화할 public key를 parameterStore에 저장하기