---
title: 3xx Redirect 코드에서의 HTTP 모듈들의 동작
categories: [Retrospect]
tags: ['WebClient', 'FeignClient', 'FollowRedirect']
toc: true

date: 2024-10-25
last_modified_at: 2024-10-25
---

## 1. 서론

이번에도 회사에서 있었던 일을 공유해보려고 합니다.

저는 회사에서 AWS QuickSight 담당을 맡고 있습니다.

그래서 특히, QuickSight 데이터를 자주 보시는 마케팅 팀의 현묵님과 자주 커뮤니케이션하는 편입니다.

하루는 현묵님께서 10월 17일 이후로 앱스플라이어 데이터가 이상해서 한 번 확인해달라는 요청을 해주셨습니다.

확인해보니 실제로 앱스플라이어 데이터가 10월 17일 이후로 불러오지 못하고 있었습니다.

원래 정상적으로 작동되던 로직이 갑자기 왜 안되는지 의문을 품었고 그 원인을 파악하고자 코드를 보았습니다.

## 2. 기존 로직 분석

우선, 앱스플라이어 데이터를 CSV 형식 데이터로 얻기 위해서는 총 두 번의 호출을 해야 했습니다.

첫 번째 호출에서는 응답이 302 Redirect 코드와 함께 Redirect URL을 제공했고, 그 Redirect URL을 통해 두 번째 호출을 하면 CSV 파일을 제공하는 형식이였습니다.

앱스플라이어에서 CSV 파일을 제공하면 저희 AWS S3 버킷에 해당 CSV 파일을 업로드하고 AWS Glue를 통해 가공하여 AWS QuickSight 데이터로 사용했습니다.

처음에는 원래 정상 작동되던 로직이였기 때문에 앱스플라이어 측의 일시적인 문제라고 생각되었지만, 시간이 지나도 고쳐지지 않자 저희 코드를 깊게 살펴보기 시작했습니다.

기존 로직은 WebClient를 통해 첫 번째 호출을 하고 302, 307, 308 코드를 받은 경우에는 한 번 더 호출하게 되어 있었습니다.

실제로 디버깅을 해 본 결과, 첫 번째 호출을 하고 나서는 Redirect URL을 받았지만 두 번째 호출의 response는 null이였습니다.

여기까지도 앱스플라이어 측 문제라고 생각했습니다.

하지만, 앱스플라이어에서 제공하는 API 테스트 도구에서 확인을 해보면 정상적으로 CSV 데이터를 주었고 또한, 해당 배치 프로그램을 저희 New API 서버에 마이그레이션한 승원님 코드에서도 정상적으로 CSV 파일을 제공 받았습니다.

그래서 저는 해당 기존 코드에 문제가 있다는 것에 확신을 가지게 되었고 승원님께서 작성하신 FeignClient와 기존 코드인 WebClient의 차이점에 대해 알아보게 되었습니다.

## 3. FollowRedirect 옵션

먼저, WebClient 모듈의 재호출하는 부분에서 NPE가 발생했기 때문에 FeignClient 모듈에서의 재호출을 하지 않아도 CSV 파일을 제공 받는 부분을 주의깊게 살펴보았습니다.

FeignClient 문서를 보니 Redirect 관련 response 코드를 받은 경우에는 디폴트로 자동 호출되도록 되어 있었습니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/retrospect/feignClient.png" width="70%" height="40%" />
</div>

하지만, WebClient 문서는 디폴트로 자동 호출되지 않도록 되어 있었습니다.

<div class="div-post-img">
  <img src="{{ site.url }}/assets/img/retrospect/webClient.png" width="70%" height="40%" />
</div>

그래서 FeignClient와 동일하게 해당 옵션을 다음과 같이 활성화했습니다. 

```java
  // DataBufferLimitException 관련 처리
  ExchangeStrategies strategies = ExchangeStrategies.builder().codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(-1)).build();

  // WebClient 설정
  WebClient webClient = WebClient.builder()
                                 .baseUrl(uri)
                                 .clientConnector(new ReactorClientHttpConnector(
                                    HttpClient.create().followRedirect(true) // 3xx Redirect 응답 코드 재 호출 활성화
                                 ))
                                 .exchangeStrategies(strategies)
                                 .build();
```

그리고 기존에 302, 307, 308 코드를 받은 경우 재호출하는 로직을 삭제했더니 정상적으로 CSV 파일을 제공 받았습니다.

사실 문제는 해결했지만, 아직까지도 10월 17일 이전까지 잘 동작하던 코드인데 왜 갑자기 되지 않는지는 이유를 잘 모르겠습니다. (앱스플라이어가 진짜 뭔가 수정을 한건지....)

## 참고

* [FeignClient 문서](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/appendix.html)

* [WebClient 문서](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-builder.html#webflux-client-builder-jdk-httpclient)