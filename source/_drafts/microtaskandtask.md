---
title: microtaskandtask
---
## event loop와 동시성 처리
### 구조
### stack
### task - task queue
### microtask - microtask queue




https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5
https://www.youtube.com/watch?v=8aGhZQkoFbQ
https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/


javascript는 기본적으로 하나의 쓰레드에서 동작한다. 하나의 쓰레드를 가지고 있다는 것은 하나의 stack을 가지고 있다는 의미와 같고, 하나의 stack이 있다는 의미는 동시에 단 하나의 작업만을 할수 있다는 의미이다.
sync 무거우면, asynchros callback



setTimeout

여러 프로그램 덩이를 시간에 따라 매 순간 한 번씩 엔진을 실행시키는 ‘이벤트 루프event loop’라는 장치다

event loop는 stack과 task/microtask queue를 보고 있다.
만약 stack이 비어 있다면 
먼저 micro task queue에 있는 callback을 꺼내서 stack에 넣는다.
micro task queuer가 비어있다면 
task queue에 있는 callback을 꺼내서 stack에 넣는다.

setTimeout(0)은 기다린다. stack이 비워질때까지 그때 실행된다.

setTimeout이 실행되면 stack에 setTimeout이 등록되고 바로 stack에서 꺼내서 브라우저에게 요청한다.
브라우저는 setTimeout에서 명시한 시간이 경과되면 callback을 task queue (작업 대기열)에 등록한다.



This allows the browser to give preference to performance sensitive tasks such as user-input. 

https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/


브라우저가 내부에서 JavaScript / DOM으로 이동할 수 있도록 작업이 예약되어 이러한 작업이 순차적으로 이루어 지도록합니다. 작업간에 브라우저는 업데이트를 렌더링 할 수 있습니다. 마우스 클릭에서 이벤트 콜백으로 이동하려면 HTML 구문 분석과 마찬가지로 작업 예약이 필요하며 위의 예인 setTimeout이 필요합니다.

task는 브라우저가 해야할 중요한 일들의 순서를 보장하기 위해 queue에 대기한다.
microtasks는 현재 실행하고 있는 스크립트 바로 다음에 바로 스케줄된다.
microtask queue는 

Microtasks include mutation observer callbacks, and as in the above example, promise callbacks.

