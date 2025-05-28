---
title: 골드스푼 동영상 업로드 기능 2 - private 버킷 조회
categories: [Goldspoon]
tags: ['upload', '동영상 업로드', '업로드', 'lambda@edge', 'S3', 'cloudfront', 'private']
toc: true

date: 2025-05-28
last_modified_at: 2025-05-28
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

하지만 단순하게 CloudFront만 사용하게 되면 경로만 하나 더 추가된 것이지 사실상 public 버킷과 다름없어 모든 사용자가 CloudFront의 URL을 통해 S3 버킷에 접근할 수 있습니다.

그래서 인증된 특정 사용자만 접근할 수 있도록 인증 시스템을 만들어야 합니다.

어떤 방법으로 만들 수 있을까요?

먼저 우리 서버들이 어떻게 사용자를 인증하는지 생각해봅시다.

대부분의 서버들은 인증(또는 인가까지) 절차를 통해 특정 사용자만 자원을 가져갈 수 있도록 합니다. 각 서버마다 인증 처리 방식이 다릅니다만, 보통 서비스에 도달하기 전 요청을 가로채 인증을 처리합니다.

AWS 환경에서도 이와 비슷하게 처리할 수 있습니다. CloudFront에 트리거를 설정하여 CloudFront에 도달하기 전 요청을 가로채 인증된 사용자만 S3 버킷에 접근할 수 있도록 만들 수 있습니다.

여기서 CloudFront에 트리거를 설정하여 요청을 가로챌 수 있는 방법은 크게 두 가지인데 CloudFront Function과 Lambda@Edge입니다.

두 서비스의 차이는 다음과 같습니다.

![CloudFront Function vs Lambda@Edge](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket17.png)

실무에서 key를 코드 상에 저장하는 것은 위험한 행동입니다. 따라서 중요한 key는 AWS ParameterStore에 저장하여 사용하겠습니다.

그러므로 CloudFront Functions보다는 AWS ParameterStore를 사용할 수 있는 Lambda@Edge를 사용하겠습니다.

여기서 또 하나의 의문이 들 수 있습니다.

> Lambda@Edge가 일반 Lambda와 무엇이 다르지?

그래서 이 또한 표로 정리해보면 다음과 같습니다.

![Lambda vs Lambda@Edge](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket18.png)

그래서 우리는 CloudFront와 통합하여 편하게 사용할 것이므로 Lambda@Edge를 사용하겠습니다.

지금까지의 설명을 그림으로 간단하게 표현하면 다음과 같습니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket2.png" width="50%" height="30%" />
</div>

이번 포스트에서는 위 설명대로 인프라를 구축해보겠습니다.

## 3. 외부에서 S3 버킷으로 바로 접근할 수 없도록 private 버킷으로 만들기

먼저 첫 번째로 해야할 일은 이전에 만들었던 S3 버킷을 private 버킷으로 전환해야 합니다.

private 버킷으로 만드는 방법은 S3 버킷의 `권한` 탭에 진입하여 아래와 같이 `모든 퍼블릭 엑세스 설정`을 활성화하면 설정할 수 있습니다.

![private 버킷 전환1](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket3.png)

![private 버킷 전환2](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket4.png)

## 4. CloudFront를 생성해 private 버킷과 연결하기

그다음 해야할 일은 private 버킷을 연결할 CloudFront를 생성하는 것입니다.

AWS 페이지 상단 검색칸에 CloudFront를 검색하고 배포 생성을 누르면 다음과 같이 나옵니다.

![CloudFront 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket5.png)

Origin domain은 우리가 앞서 만들었던 S3 버킷을 선택하고, Origin path는 비워둡니다.

그리고 가장 중요한 OAI(Origin Access Identity) 또는 OAC(Origin Access Control)를 설정합니다.

OAI와 OAC에 대해 간략하게 설명하면 둘 다 CloudFront에서 S3 버킷에 대한 접근을 제어하는 보안 방식인데 OAC는 OAI보다 조금 더 세부적인 보안 설정이 가능합니다.

OAC에 대한 자세한 내용은 [공식문서](https://aws.amazon.com/ko/blogs/korea/amazon-cloudfront-introduces-origin-access-control-oac/){:target="_blank"}를 참조하세요!

여기서 OAI를 설정하려면 `Legacy access identities`, OAC를 설정하려면 `원본 액세스 제어 설정(권장)`을 선택하면 되는데 우리는 권장 방법인 `원본 액세스 제어 설정(권장)`을 선택하겠습니다.

Create new OAC를 선택하여 OAC를 생성합니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket9.png" width="50%" height="30%" />
</div>

그리고 Origin access control에 방금 전 만들었던 OAC를 선택합니다.

다음은 CloudFront의 캐시 동작을 설정하는 화면입니다.

저는 뷰어 프로토콜 정책만 `Redirect HTTP to HTTPS`로 바꾸고 나머지는 그대로 두겠습니다.

![기본 캐시 동작 설정](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket6.png)

아래로 내려가면 함수 연결을 할 수 있는 섹션이 있습니다. 추후 Lambda@Edge를 생성 후 연결하여 특정 사용자만 조회할 수 있도록 설정할 예정입니다.

WAF는 CloudFront의 접근을 제어하고 싶을 때 사용합니다. 실 서비스에서는 보안을 위해 연결하기도 하는데 지금은 테스트용 인프라이므로 비활성화로 두겠습니다.

![함수 연결과 WAF 설정](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket7.png)

다음은 대체 도메인을 세팅할 수 있는데 CloudFront 생성 시 기본으로 생성되는 도메인은 기억하기 어려우므로 도메인을 생성하여 대체 도메인으로 설정하면 설정한 도메인으로 CloudFront에 접근할 수 있습니다. 지금은 테스트이므로 따로 설정하지는 않겠습니다.

그리고 나머지는 그대로 두고 `배포 생성`을 선택합니다.

![대체 도메인 설정](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket8.png)

그러면 S3를 바라보고 있는 CloudFront가 생성됩니다.

하지만 여기서 끝난 것이 아닙니다. 가는 것이 있으면 오는 것도 있는 법! S3에도 생성한 CloudFront에서의 접근을 허용하도록 설정해야 합니다.

앞서 생성했던 S3로 이동하여 권한 탭의 버킷 정책 섹션에서 편집을 선택합니다.

![S3 버킷 정책 설정](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket10.png)

여기에서 우리가 만든 S3 버킷의 모든 정책을 관리할 수 있는데 앞서 생성한 CloudFront에서의 접근을 허용하도록 설정해보겠습니다.

```json
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipalReadOnly",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "{S3_ARN}/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "{CloudFront_ARN}"
                }
            }
        }
    ]
  }
```

`resource`는 S3 내부에서 허용할 자원을 말하는데 우리는 해당 버킷 내 모든 자원을 CloudFront를 통해 접근할 것이므로 S3 ARN 뒤에 모든 자원을 뜻하는 `/*`을 붙입니다.

S3 ARN은 해당 페이지 내에 존재합니다.

![S3 ARN 위치](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket11.png)

그리고 CloudFront ARN은 앞서 생성한 CloudFront 페이지 내에 존재합니다.

![CloudFront ARN 위치](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket12.png)

두 ARN을 세팅했으면 `변경 사항 저장`을 선택합니다.

그러면 CloudFront를 통해 앞서 생성한 private 버킷에 접근할 수 있게 됩니다.

위 사진의 가장 처음에 있는 `배포 도메인 이름`으로 S3 버킷에 업로드한 테스트 파일을 조회해봅시다.

> 📌 S3 버킷 내 경로와 동일하게 호출해야 합니다.

잘 나오죠? 😀

## 5. JWT를 복호화할 public key를 parameterStore에 저장하기

위의 설정까지만 하면 모든 사용자가 조회할 수 있어 사실상 public 버킷과 다름없습니다.

지금부터는 특정 사용자만 접근할 수 있도록 인증 시스템을 만들어보겠습니다.

인증 수단은 현재 널리 사용하고 있는 JWT를 사용하겠습니다.

JWT를 통해 인증 시스템을 만들기 위해서는 먼저 public key와 private key를 생성해야 합니다.

각 key는 OpenSSL을 통해 쉽게 생성할 수 있습니다.

> Mac 환경에서는 homebrew, Windows 환경에서는 아래 사이트에서 설치 가능합니다.
> 
> Mac: `brew install openssl`
>
> Windows: [OpenSSL 설치 사이트](https://slproweb.com/products/Win32OpenSSL.html){:target="_blank"}

```bash
  # 1) 2048비트 RSA 비공개키 생성
  openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048

  # 2) 비공개키에서 공개키 추출
  openssl rsa -pubout -in private.pem -out public.pem
```

해당 key를 이용해 JWT를 생성하는 방법은 [JWT 생성 사이트](https://jwt.io/){:target="_blank"}에 진입하여 알고리즘과 `PAYLOAD`를 수정한 후 `VERIFY SIGNATURE`에 방금 생성한 private key와 public key를 복사하여 붙여넣습니다.

![JWT 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket14.png)

생성된 JWT를 메모장과 같은 곳에 따로 저장해놓습니다. 해당 JWT는 테스트로 사용할 것입니다.

이제 이 JWT를 복호화할 public key를 저장해봅시다.

이때 key가 코드 내에 존재하는 것은 보안에 좋지 않으므로 AWS 환경 내 key-value 형태로 저장되는 parameterStore에 저장합니다.

parameterStore에 저장하기 전 가장 중요한 부분이 하나 있는데 바로 `us-east-1` 리전에 생성해야 한다는 것입니다.

그 이유는 우리가 곧 생성할 Lambda@Edge는 `us-east-1`에서만 생성 가능하기 때문입니다.

그래서 parameterStore에 진입하여 리전을 `us-east-1`으로 변경한 후 `파라미터 생성`을 선택합니다.

![parameterStore 생성1](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket13.png)

그러면 파라미터를 생성할 수 있는 페이지가 나옵니다.

여기서 외부에서 접근할 수 있는 이름을 임의로 넣고 우리가 저장할 내용이 보안 상 중요한 key이므로 보안 문자열로 선택합니다.

![parameterStore 생성2](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket15.png)

그리고 이전에 생성했던 public key를 해당 위치에 붙여넣습니다. 그 후, `변경 내용 저장`을 선택하여 최종 저장합니다.

![parameterStore 생성3](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket16.png)

이렇게 되면 AWS 환경 내 `us-east-1` 리전에서 public key를 접근할 수 있게 됩니다.

## 6. Lambda@Edge 실행 권한을 가진 IAM 역할 생성하기

이번에는 Lambda@Edge 실행을 위한 IAM 역할을 생성해보겠습니다.

먼저 IAM 서비스에 진입 후, 액세스 관리 내부의 역할 탭에 들어가서 `역할 생성`을 선택합니다.

![IAM 생성1](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket33.png)

생성 페이지의 가장 먼저 나오는 페이지에서는 유형과 사용 사례를 선택할 수 있습니다.

우리는 Lambda 실행에 필요한 역할을 생성할 것이므로 신뢰할 수 있는 엔터티 유형은 `AWS 서비스`, 사용 사례는 `Lambda`를 선택합니다.

![IAM 생성2](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket34.png)

다음 페이지는 신뢰 정책을 선택할 수 있는 페이지입니다.

아래 3개의 권한을 추가해주세요.

> * S3 읽기 권한 (`AmazonS3ReadOnlyAccess`)
>
> * ParameterStore 읽기 권한 (`AmazonSSMReadOnlyAccess`)
>
> * AWS Lambda 실행 권한 + CloudWatch 로그 권한 (`AWSLambdaBasicExecutionRole`)

이후에 나오는 페이지는 최종으로 검토하는 페이지인데 앞서 선택한 내용과 동일한지 확인하고 저장합니다.

그리고 방금 생성한 역할에 다시 들어가서 신뢰 관계에 `edgelambda.amazonaws.com`를 추가해주세요.

```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": [
              "lambda.amazonaws.com",
              "edgelambda.amazonaws.com"
          ]
        },
        "Action": "sts:AssumeRole",
      }
    ]
  }
```

그럼 Lambda@Edge를 생성 및 실행하기 위한 모든 준비가 완료되었습니다.

## 7. Lambda@Edge 생성하기

자, 이제 Lambda@Edge를 생성해보겠습니다.

리전을 `us-east-1`으로 변경한 후 Lambda 서비스의 홈에 진입하여 `함수 생성`을 선택합니다.

![Lambda 생성1](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket19.png)

`함수 생성`을 선택하면 다음과 같이 함수를 생성할 수 있는 페이지가 나오게 됩니다.

![Lambda 생성2](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket20.png)

Lambda@Edge는 일반 Lambda와 다르게 Node.js와 Python 언어만 사용할 수 있습니다.

저는 Node.js를 사용하겠습니다.

그리고 추가로 `기본 실행 역할 변경`을 통해 방금 생성한 역할을 선택합니다.

![Lambda 생성3](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket21.png)

그리고 최종으로 `함수 생성`을 선택해 Lambda 함수를 생성합니다.

## 8. 로컬에서 코드를 작성하여 Lambda 서비스에 코드 업로드하기

원래 Lambda 함수는 인프라 서버 구축없이 코드만 작성하여 마치 서버가 있는 것처럼 바로 사용 가능합니다.

하지만, 우리가 사용할 라이브러리의 용량이 커 100MB 제한이 있는 Lambda에서 바로 작성하는 것은 불가능합니다.

그 대안으로 라이브러리를 Layer로 두는 방법으로 시도해보았지만 Lambda@Edge에서는 불가능하더라구요.

그래서 또 다른 대안을 찾다가 [다음](https://stackoverflow.com/questions/77986827/cloudfront-lambdaedge-in-nodejs-layer-is-not-supported-for-cloudfront-how-to){:target="_blank"} 링크에서
로컬에서 코드와 라이브러리 묶음을 zip파일로 만들어 S3로 업로드하고 Lambda에서 해당 S3의 zip파일을 업로드하여 사용하는 방법을 찾게 되었습니다.

해당 방법을 통해 Lambda에 코드를 업로드해보겠습니다.

먼저 로컬에서 디렉토리를 하나 생성해서 해당 디렉토리에 진입합니다.

```bash
  # 디렉토리 생성
  mkdir lambda-edge-auth

  # 생성한 디렉토리 진입
  cd lambda-edge-auth
```

디렉토리에 진입하여 우리는 Node.js를 사용할 것이므로 자바스크립트 라이브러리를 설치합니다.

설치할 라이브러리는 AWS ParameterStore를 사용하기 위한 `@aws-sdk/client-ssm`과 JWT 인증을 위한 `jsonwebtoken`입니다.

```bash
  # SSM 패키지 설치 (AWS ParameterStore 사용)
  npm install "@aws-sdk/client-ssm"

  # JWT 인증 패키지 설치
  npm install "jsonwebtoken"
```

그러면 해당 위치에 `node_modules` 디렉토리와 `package-lock.json`, `package.json` 파일이 생성된 것을 볼 수 있습니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket22.png" width="50%" height="30%" />
</div>

추가로 Lambda가 실행될 index.js를 생성합니다. 이때 아무 텍스트 편집기 중 하나를 사용하면 됩니다. (ex. vi, vim, 메모장 등)

```javascript
// index.js
  const jwt = require("jsonwebtoken");
  const { SSMClient, GetParameterCommand } = require("@aws-sdk/client-ssm");

  const ssmClient = new SSMClient({
    region: "us-east-1", // Lambda@Edge는 us-east-1에서 실행
  });

  const PARAMETER_NAME = '/jwt-public-key'; // ParameterStore의 키 이름

  // SSM Parameter Store에서 공개키 가져오기
  const getPublicKey = async () => {
    const command = new GetParameterCommand({
      Name: PARAMETER_NAME, // SSM에 저장된 키 경로
      WithDecryption: true  // ParameterStore가 암호화되어 있는 경우
    });

    try {
      const response = await ssmClient.send(command);
      return response.Parameter.Value;
    } catch (error) {
      console.error("🔴 SSM에서 공개키 가져오기 실패:", error);
      throw new Error("public key 조회 실패");
    }
  };

  exports.handler = async (event, context, callback) => { // Lambda가 실행되는 부분
    const request = event.Records[0].cf.request;
    const headers = request.headers;

    let token = null;

    // Authorization 헤더에서 JWT 추출
    if (headers.authorization) {
      const authHeader = headers.authorization[0].value;
      if (authHeader.startsWith("Bearer ")) {
        token = authHeader.split(" ")[1];
      }
    }

    if (!token) {
      return callback(null, { status: '401', body: 'Not Found Token.' });
    }

    try {
      // 공개 키 가져오기
      const publicKey = await getPublicKey();

      // JWT 검증
      const decoded = jwt.verify(token, publicKey, { algorithms: ["RS256"] });
    } catch (err) {
      console.error("JWT Verification Error:", err.message);
      return callback(null, { status: '401', body: 'Invalid or Expired Token.' });
    }

    return callback(null, request); // 요청을 계속 진행
  };
```

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket23.png" width="50%" height="30%" />
</div>

그리고 지금까지 생성한 파일 및 폴더를 zip 파일로 압축합니다.

```bash
  zip -r lambda-package.zip .
```

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket24.png" width="50%" height="30%" />
</div>

이제 이 zip 파일을 S3에 업로드하겠습니다.

우리는 Lambda@Edge를 생성할 것이므로 `us-east-1` 리전에 S3 버킷을 하나 생성하고 방금 생성한 zip 파일을 업로드합니다.

![Lambda 코드 S3 업로드](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket25.png)

그리고 해당 zip 파일을 선택하여 상세 페이지에 들어간 후 S3 객체 URL을 복사합니다.

![S3 경로 복사](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket26.png)

이제 Lambda로 다시 돌아가 코드를 업로드해보겠습니다.

앞서 만들었던 Lambda의 우측 상단의 `Amazon S3 위치`를 선택하여 복사한 URL을 붙여넣습니다.

![코드 업로드](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket27.png)

업로드에 성공하면 다음과 같이 우리가 로컬에서 생성했던 파일들이 보입니다.

그러면 보이는 모든 파일을 전체 선택 후 저장(⌘ + s / ctrl + s)하고 아래 Deploy를 선택합니다.

![코드 배포](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket28.png)

정상적으로 배포되었다면 우리가 생성한 Lambda의 홈으로 이동하게 됩니다.

## 9. 생성한 Lambda@Edge를 CloudFront에 트리거 설정하기

이제 생성한 Lambda를 CloudFront에 트리거 설정해보겠습니다.

Lambda의 홈 아래 버전 탭에서 `Publish new version`을 선택하여 새로운 버전을 생성합니다.

![버전 생성](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket29.png)

버전을 생성하면 아래와 같은 화면이 나오는데요.

여기서 `트리거 추가`를 선택하여 앞서 생성한 CloudFront와 연결합니다.

![트리거 설정1](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket30.png)

트리거 설정은 아래와 같이 작성합니다.

우리가 트리거를 새로 생성하기에 `새로운 CloudFront 트리거 구성`을 선택합니다.

배포는 앞서 생성한 CloudFront를 선택하고 캐시 동작은 `*`로 지정합니다.

그리고 CloudFront 이벤트라는 것이 있는데 크게 4가지가 있습니다. 

4가지의 이벤트는 각기 다른 CloudFront의 트리거 설정 위치로 아래 그림을 보면 쉽게 이해할 수 있습니다.

<div class="div-post-img">
  <img src="https://docs.aws.amazon.com/images/AmazonCloudFront/latest/DeveloperGuide/images/cloudfront-events-that-trigger-lambda-functions.png" width="50%" height="30%" />
  <p>AWS CloudFront Events</p>
</div>

이 4가지 중 우리는 CloudFront에 도달하기 전에 Lambda 함수를 통해 JWT 인증을 할 것이므로 `Viewer request`로 설정합니다.

마지막으로 가장 중요한 `Lambda@Edge로 배포 확인`을 선택합니다.

![트리거 설정2](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket31.png)

트리거 생성을 완료하면 아래와 같이 Lambda가 CloudFront에 트리거로 설정된 것을 볼 수 있습니다.

![트리거 설정3](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket32.png)

트리거 설정은 완료되는데 수 초에서 수 분 정도 소요됩니다.

완료 확인은 CloudFront의 마지막 수정이 날짜가 나오면 됩니다.

![트리거 설정4](https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket35.png)

## 10. API 테스트 도구로 이미지 조회 테스트해보기

이제 API 테스트 도구로 이미지를 조회해보겠습니다.

이때 아무 API 테스트 도구를 사용해도 좋습니다. 저는 postman을 사용하겠습니다.

앞서 4번 항목에서 CloudFront의 배포 도메인을 통해 이미지를 조회해봅시다.

그러면 이미지가 잘 나오던 이전과는 달리 `Not Found Token` 이라는 문구를 볼 수 있을 것입니다.

(만약, 나오지 않는다면 설정이 잘못된 것이니 다시 설정해야 합니다.)

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket36.png" width="50%" height="30%" />
</div>

이제 이런 문구가 나오는 이유는 CloudFront에 도달하기 전 Lambda가 요청을 가로채 토큰 검증을 했기 때문입니다.

그럼 이제 앞서 5번에서 테스트를 위해 저장했던 jwt를 통해 테스트해보겠습니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket37.png" width="90%" height="50%" />
</div>

그림과 같이 jwt를 넣었을 때는 이미지가 잘 나오는 것을 볼 수 있습니다.

이번에는 테스트를 위해 jwt의 마지막 한자리를 빼고 요청을 보내겠습니다.

<div class="div-post-img">
  <img src="https://d36u0n6bmvvikl.cloudfront.net/blog/private_bucket/private_bucket38.png" width="90%" height="50%" />
</div>

이번에는 토큰이 유효하지 않기 때문에 `Invalid or Expired Token`이 나오는 것을 확인할 수 있습니다.

## 11. 정리

여기까지 private 버킷에 있는 이미지를 특정 사용자만 조회할 수 있도록 처리하는 방법과 그와 함께 인프라를 직접 구현해보았습니다.

해당 방법은 프로필과 같은 개인정보를 보호하기 위해 필요한 좋은 기술이라고 생각합니다.

지금까지의 내용을 천천히 적용해보시고 각자 실무에 적용한다면 많은 도움이 되실거라 믿습니다.