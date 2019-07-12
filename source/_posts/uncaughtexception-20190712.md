---
title: Node.js 서버 장애의 교훈. UncaughtException
categories:
  - Tech
tags:
  - nodejs
  - uncaughtException
  - SSR
  - axios
  - sentry
abbrlink: 32124
date: 2019-07-12 21:06:50
---

**Node 기반의 SSR(Server Side Rendering) 랜더링** 서비스를 운영하다 1급 장애를 맞았다.

### 장애 현상
1. 배포 후, 일정 시간 사용자 유입 이후 다음과 같은 서버 에러가 발생
   ```bash
   [ERR_INVALID_PROTOCOL]: Protocol “http:” not supported. Expected “https:”
      at new ClientRequest (_http_client.js:119:11)
      at Object.request (http.js:42:10)
      at RedirectableRequest.module.exports.RedirectableRequest._performRequest (/home1/irteam/deploy/mobile/webpack:/node_modules/follow-redirects/index.js:219:1)
      at RedirectableRequest.module.exports.RedirectableRequest._processResponse (/home1/irteam/deploy/mobile/webpack:/node_modules/follow-redirects/index.js:330:1)
      at ClientRequest.RedirectableRequest._onNativeResponse (/home1/irteam/deploy/mobile/webpack:/node_modules/follow-redirects/index.js:53:1)
      at Object.onceWrapper (events.js:277:13)
      at ClientRequest.emit (events.js:189:13)
      at ClientRequest.EventEmitter.emit (domain.js:441:20)
      at HTTPParser.parserOnIncomingClient [as onIncoming] (_http_client.js:556:21)
      at HTTPParser.parserOnHeadersComplete (_http_common.js:109:17)
   ```
2. 에러 로그와 함께 node 서버의 **CPU가 90%** 에 도달하면서 사용자 요청을 받을 수 없는 상황 발생. 
3. 사용자 요청을 막고 node 서버 재기동 시도. 하지만 **복구 불가**.
4. **일정 시간(15분) 경과** 후 node 서버 재기동 시 정상 동작

<!-- more -->

### 장애 원인
장애의 원인은 사실 사내 시스템 구조 때문에 발생한 단순 `운영 이슈`였다.
간단히 설명한다면 브라우저에서 의도적으로 단기간에 많은 요청을 하게 되면 요청을 block 시키고 다음과 같은 에러페이지를 보여준다.

{% asset_img block.png %}

마찬가지로 서버에서 API 서버로 호출 시에도 위와 같은 페이지로 forwarding 되고 이를 막기 위해서는 사내 시스템에 호출하는 서버를 화이트 리스트로 등록해야만 했다. ㅠㅠ

원인 파악과 별개로 사실 몇가지 의문이 들었다.

1. API에서 에러가 났는데 `왜? Protocol “http:” not supported. Expected “https:” 메시지가 나오는가?`
2. API에서 에러가 발생하면 에러 처리를 할 수 있도록 `에러 Flow를 테스트 코드와 함께 검증을 해놓은 상태인데 왜 이런 상황이 발생했나?`
   ```js
   // node 서버 코드
   try {
     const data = await getLoginUserInfo();
     // ...
   } catch (e) {
     // 에러 상황 확인 후, sentry로 에러 로깅
   }
   ```
3. API에서 에러가 발생했는데 `왜? CPU는 90%를 사용하였나?`


### 장애 현상의 궁금증

` Protocol “http:” not supported. Expected “https:"` 에러의 시작점은 `axios 모듈`에서 발생하였다.
이렇게 된 이유는 https 로 사용자 요청이 들어왔을 시, node server에서는 API 서버로 요청하게 되고 이를 `공격자`로 판단하여 302 HTTP Status 코드와 함께 http 프로토콜의 페이지를 node server로 전달한다.

이때 node server의 axios가 http 프로토콜 형태의 페이지로 요청을 보내려고 하다보니 Node.js Core 모듈의 http client에서 Exception이 발생하였다.

다음은 Node.js Core 모듈의 http client 소스의 내용이다.

```js
function ClientRequest(input, options, cb) {
  // ...
  if (protocol !== expectedProtocol) {
    // 프로토콜이 다르다고 에러를 던짐
    throw new ERR_INVALID_PROTOCOL(protocol, expectedProtocol);
  }
  // ...
}
```
> 출처: https://github.com/nodejs/node/blob/master/lib/_http_client.js#L144

그래서 장애시 위와 같은 에러메세지가 발생하게 된 것이다.

여기서 놓친게 있었는데...


1. **axios는 302 HTTP Status 코드를 받게 되면 자동으로 redirect 요청을 보낸다.**
   ([axios의 기본 설정값](https://github.com/axios/axios)은 maxRedirects 가 `5` 이다.)

   ```js
   // `maxRedirects` defines the maximum number of redirects to follow in node.js.
   // If set to 0, no redirects will be followed.
   maxRedirects: 5, // default
   ```

2. **Node.js에서는 비동기 처리 내에서 Exception이 발생하면  `uncaughtException` 에러로 처리된다.**
   **중요한 것은 이 경우 `try ~ catch` 영역으로 에러 처리를 했어도 에러를 잡을 수 없다**


결국, axios의 redirect 옵션에 의해 request가 불러지고, 이로 인해 Node.js Core에서 `uncaughtException`이 던져지면서  Node Instance는 죽고, pm2는 다시 Node instance를 살리고, 다시 사용자 요청이 오면 Node Instance가 죽고 다시 pm2가 Node Instance를 살리는 상황이 계속 반복되었다.
당연히, 이로 인해 CPU는 90%까지 치솟게되었다. 주르륵

{% asset_img 90p.png %}


### UncaughtException

사실 Node 서버 개발에서 `uncaughtException` 에러 처리는 기본 작업이다. 그렇지 않으면 에러가 나면 서버가 죽어버리기 때문이다.
물론 사전에 이런 작업에 대한 방어 처리를 해놓았었다.
바로 Sentry에서 제공하는 errorHandler를 이용해서... 요렇게
```js
import {
  Handlers
} from '@sentry/node';

// ...
app.use(handlers.errorHandler());  
```

하지만, 막상 `uncaughtException` 이 발생하자 pm2로 `uncaughtException`이 전달되었고 Node Instance는 죽기 시작했다.
실제 Sentry errorHandler의 코드를 봤더니 다음과 같았다.

```js
if (!caughtFirstError) {  // 첫번째 uncaughtException이 발생한 경우
  // ...
  hub.captureException(error, { originalException: error });
  if (!calledFatalError) {
    calledFatalError = true;
  }
  // ...
} else if (calledFatalError) { // 두번째 uncaughtException이 발생한 경우
  // we hit an error *after* calling onFatalError - pretty boned at this point, just shut it down
  logger.warn('uncaught exception after calling fatal error shutdown callback - this is bad! forcing shutdown');
  // ...
  global.process.exit(1); // Application 종료!
}
```

> 출처: https://github.com/getsentry/sentry-javascript/blob/master/packages/node/src/integrations/onuncaughtexception.ts#L66-L93

<u>첫번째 `uncaughtException` 에러가 발생하면 sentry로 로그를 보내고, 
두번째 `uncaughtException` 에러가 또 발생하면 Node instance를 강제로 종료하였다.</u>

`uncaughtException` 을 처리할 거면 계속 처리할 것이지 왜 Sentry는 이랬을까? 

사실 [Node.js의 정식 문서](https://nodejs.org/api/process.html#process_warning_using_uncaughtexception_correctly)에서는 UncaughtException이 발생하는 경우 `Application을 재시작`하라고 가이드 하고 있다.
Sentry는 가이드 대로 충실하게 따랐던 거죠 ㅠㅠ 



### 결론: 장애의 교훈

1. axios를 SSR에서 사용하는 경우, 꼭! `maxRedirects` 옵션을 0으로 설정하여 redirect되지 않도록 하자.
2. Node.js의 비동기 작업에서 에러가 나는 경우 `uncaughtException` 이 발생할 수 있다. 
3. 이땐 `try~catch`로도 에러를 잡을 수가 없다.
4. `uncaughtException`이 발생하면 Node Application은 재시작하자!