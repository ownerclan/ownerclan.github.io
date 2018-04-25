---
layout: post
title:  "Promise & async/await 1회차"
date:   2018-04-23
description: "Promise 입문"
---
## Promise

369 게임을 하는 프로그램을 짜보자.

```typescript
setTimeout(() => {
    console.log(1);
    setTimeout(() => {
        console.log(2);
        setTimeout(() => {
            console.log("짝");
        }, 1000)
    }, 1000)
  }, 1000)
```

이 얼마나 읽기 어려운 코드인가. 1초마다 각각 *1*, *2*, *짝*을 출력하는 코드가 이렇게 복잡하다니. 10까지 가면 들여쓰기를 10번 해야 한다. 이런걸 **Callback hell**이라고 한다.  여기에 문제를 느끼고 만든 것이 바로 **Promise**이다.

```typescript
console.log(1)
delay(1000)
    .then(() => console.log(2))
    .then(() => delay(1000))
    .then(() => console.log("짝"));
```

아직도 좀 읽기 어렵지만, 그래도 위와 달리 뭔가 순서대로 실행된다는 느낌은 오지 않는가? 좀더 읽기 좋게 바꿔보고 하는 김에 6까지는 세어보자.

```typescript
const delay1s = () => delay(1000);

console.log(1)
delay1s()
    .then(() => console.log(2))
    .then(delay1s)
    .then(() => console.log("짝"))
    .then(delay1s)
    .then(() => console.log(4))
    .then(delay1s)
    .then(() => console.log(5))
    .then(delay1s)
    .then(() => console.log(6))
console.log("땡")
```

> 이렇게 `then`이 연달아서 나오는걸 Promise chain이라고 한다.

들여쓰기를 6번 해야하는 것은 피할수 있게 되었다. 그럼 여기서 `delay`함수는 어디서 온것인지가 궁금할 것이다. 일단 `delay`는 이렇게 만들면 된다.

```typescript
function delay(time: number) {
    return new Promise<void>((resolve) => setTimeout(resolve, time));
}
```

이 코드를 지금 이해하려고 하진 말자. 그런데 대충 느낌으로 읽어보면 **약속(Promise)**을 만드는데 그 약속이 **해결(resolve)**되는 것은 `time`밀리초 이후라는 것이다. 보다시피 내부적으로 setTimeout을 사용하는걸 알 수 있다. 같은 동작을 하지만 코드를 더 읽기 쉽게 해준다. 이제 `delay` 덕분에 중간에 몇 초 쉬는 것은 처음보다 꽤 쉬운일이 되었다.

그런데 코드를 실행해보면 *땡*이 *6*이후에 나오는게 아니라 *1*이후에 바로 나온다. 왜냐하면 Promise는 약속일 뿐이지 기다리지 않기 때문이다. 기다리려면 `then`을 사용해야한다. 그러면 `delay1s()`이후로 코드가 두 갈래로 나뉘어져서 실행되나? 하나는 369를 계속하고 하나는 그 뒤의 코드를 계속 실행하고?

```typescript
console.log(1)
delay1s()
    .then(() => console.log(2))
    .then(delay1s)
    .then(() => console.log("짝"))
    .then(delay1s)
    .then(() => console.log(4))
    .then(delay1s)
    .then(() => console.log(5))
    .then(delay1s)
    .then(() => console.log(6))

console.log("왜 계속 안하세요?")
while (true) {
}
```

몇 시간을 기다려도 절대 *2*가 출력되는 일은 없을 것이다. NodeJS든 그냥 브라우저든 JavaScript 코드의 마지막 줄에는 하나가 빠져있다고 보면 된다.

```javsript
runEventLoop(); //'이제 남아있는 이벤트들을 처리합시다'라는 뜻이다.
```

물론 저렇게 쓰라는게 아니고 실제로 저런식으로 동작한다는 것이다. 코드 실행이 끝나면 이벤트를 기다리며 jQuery에서 설정한 버튼의 마우스 클릭 이벤트를 기다렸다 처리하든, `setTimeout`으로 10초있다가 실행하라는 명령을 처리하든 알아서 한다.

일단 이벤트 루프에 대해선 당분간 잊어버리고 다음 예제를 보자.

```typescript
function do369() {
    console.log(1);
    return delay1s()
        .then(() => console.log(2))
        .then(delay1s)
        .then(() => console.log("짝"))
        .then(delay1s)
        .then(() => console.log(4))
        .then(delay1s)
        .then(() => console.log(5))
        .then(delay1s)
        .then(() => console.log(6));

}

do369().then(() => console.log("저쪽 때문에 헷갈려"));
do369().then(() => console.log("저쪽 때문에 헷갈려"));
```

보다시피 2개의 369게임이 동시에 진행되는걸 볼 수 있다. 한 게임이 다른 한쪽을 막는일 없이. 그럼 한쪽이 끝나고  다른 게임을 시작하게 하려면 어떡해야 하나?

```typescript
do369()
    .then(() => do369())
    .then(() => console.log("똑같은데?"));
```

## 그래서 어디에 쓰는가

지금까지 한건 프로그램이 실행되다가 기다리게 만드는, 일반적인 상황에선 쓸모없는 동작을 구현했다.  실제로 Promise를 많이 쓰게 되는건 파일이나 HTTP 요청 등 I/O를 다룰 때이다. 다음 코드를 실행하면 HTTP 응답이 오는 순서대로 사이트의 title이 뜬다.

```typescript
import * as cheerio from "cheerio";
import * as request from "request-promise-native";

request("https://www.facebook.com")
    .then((res) => {
        const $ = cheerio.load(res);
        console.log($("title").text());
    });

request("https://www.instagram.com")
    .then((res) => {
        const $ = cheerio.load(res);
        console.log($("title").text());
    });

request("https://www.twitter.com")
    .then((res) => {
        const $ = cheerio.load(res);
        console.log($("title").text());
    });
```

여기서 `res`가 무엇인지 궁금할텐데, Promise는 값을 넘겨줄 수 있다. 함수의 *return*과 같다. 그리고 `then`에서 또다시 값을 넘겨줄 수 있다. 마찬가지로 그냥 *return*을 하면 된다.

```typescript
function getTitleFromHTML(html: string) {
    const $ = cheerio.load(html);
    return $("title").text();
}

request("https://www.facebook.com")
    .then(getTitleFromHTML)
    .then((title) => console.log(title));

request("https://www.instagram.com")
    .then(getTitleFromHTML)
    .then((title) => console.log(title));

request("https://www.twitter.com")
    .then(getTitleFromHTML)
    .then((title) => console.log(title));

```

이렇게 다시 쓸 수 있다. `title`을 넘겨받아서 다시 출력한다. `then`에 넘겨줘야할 함수는 Promise를 리턴해야하는데, `getTitleFromHTML`은 사실 Promise을

혹시 `request`는 뭘 *return*하는지 궁금할 수 있다. `then`으로 넘겨받는 값 말고 말이다. 궁금할 땐 console.log로 찍어보자.

```typescript
console.log(request("https://www.twitter.com"));
```

`Promise { <pending> }`라고 뜰 것이다. `request`는 Promise를 돌려준다. pending은 아직 이벤트가 처리되지 않고 남아있다는 뜻이다.

`then`안에서 다시 Promise를 쓰려면 어떻게 하면 될까?

```typescript
request("https://www.facebook.com")
    .then((html) => {
        console.log(getTitleFromHTML(html));
        return request("https://www.instagram.com");
    })
    .then((html) => {
        console.log(getTitleFromHTML(html));
        return request("https://www.twitter.com");
    })
    .then((html) => {
        console.log(getTitleFromHTML(html));
    });
```

이러면 차례대로 `request`를 호출하게 된다.

간혹, HTTP 요청을 여러개 보내놓고 모두 응답이 올때까지 기다려야할 때가 있다. 이때 `Promise.all`을 쓰면 된다.

```typescript
const promises = [
    request("https://www.facebook.com"),
    request("https://www.instagram.com"),
    request("https://www.twitter.com"),
];
Promise.all(promises)
    .then((htmls) => {
        for(const i of htmls) {
            console.log(getTitleFromHTML(i))
        }
    });
```

> 사실 예제를 자세히 살펴보면 `then`안에 도대체 뭐가 와야하는지 헷갈릴 수 있다. Promise를 돌려주는 함수도 올 수 있고, `getTitleFromHTML` 같은 평소에 보던 함수도 올 수 있고 말이다. 이건 Promise의 정의가 무엇인지랑 관련있는데 나중에 살펴보자.

## catch

VS Code를 쓰고 있다면 알겠지만, Promise이후에 따라올 수 있는 건 `then`말고 `catch`도 있다. *try/catch*할 때 그 *catch* 맞다. 웹사이트 주소를 받아서 title을 돌려주는 함수를 만들어보자.

```typescript
function getTitle(url: string) {
    return request(url)
        .then(getTitleFromHTML);
}

getTitle("http://www.없는사이트.com/");
```

그러면

```
(node:19464) UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 1): RequestError: Error: getaddrinfo ENOTFOUND www.xn--sh1b80y51dh0b9v0a.com www.xn--sh1b80y51dh0b9v0a.com:80
(node:19464) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

이런 에러가 뜬다. 무슨 뜻인지는 나중에 다시 알아보자. Promise는 약속이란 뜻이지만 무조건 성공한다는 걸 보장하진 않는다. `then`으로 갈 수 없는 상황도 존재한다. 대부분의 Promise가 `catch`또한 구현해주어야 한다. 그럼 `getTitle` 함수를 마저 완성해보자.

```typescript
function getTitle(url: string) {
    return request(url)
        .then(getTitleFromHTML)
        .catch((ex) => console.error("크롤링에 실패했습니다."));
}
```

저기 `ex`또한 우리가 *try/catch* 쓸 때 쓰는 그 `ex`가 맞다. 저기서 다시 `then`이나 `catch`를 붙여서 Promise chain을 이어나갈 수도 있다.

```typescript
function getTitle(url: string) {
    return request(url)
        .then(getTitleFromHTML)
        .catch((ex) => {
            throw new Error("크롤링에 실패했습니다.");
        })
        .catch((ex) => {
            console.error(ex);
        });
}
```

*throw*를 하면 `catch`로 받고 *return*을 하면 `then`으로 받는다. 물론 뭘 줄지는 알 수 없으므로, 위의 예시처럼 자기가 만든 Promise chain이라서 `catch`만 하면 된다는걸 아는 상황이 아닌 이상 언제나 `then`과 `catch` 둘다 구현해 놓아야한다.

## Promise 또한 값이다

앞서 `request(...)`를 `console.log`로 직접 찍어보았다. 그랬더니 얘는 Promise라고 했다. Promise 또한 값이다. 특별한 문법이 아닌, 기존의 Javascript 문법을 가지고 만든 일종의 **패턴**이다. 하도 많이 쓰다보니까 Javascript 스펙에 포함되어버리긴 했지만 말이다.

Promise가 값이란 걸 알면, Promise chain을 직접 작성할 필요없이 코드를 통해 생성하는 것도 가능하다.  n번째에 틀리는 369 게임을 만들어보자.

```typescript
function play369(n: number) {
    console.log(1);
    let promise = Promise.resolve();
    for (let i = 2; i < n; i++) {
        promise = promise.then(delay1s);
        if ( i.toString().match(/[369]/) ) {
            promise = promise.then(() => console.log("짝"));
        } else {
            promise = promise.then(() => console.log(i));
        }
    }
    promise = promise.then(delay1s);
    if ( n.toString().match(/[369]/) ) {
        promise = promise.then(() => console.log(n));
    } else {
        promise = promise.then(() => console.log("짝"));
    }
}
```

*for*문을 돌면서 Promise chain을 만드는 것에 주목하자.

> ### async/await는 언제 하나?
>
> *async/await*으로 Promise를 완전히 대체할 수 있다면, 굳이 Promise를 배우지 않아도 될텐데 안타깝게도 그렇지가 못하다. 일단 기능으로 따지면 `Promise.all`가 그렇다. 또 *async/await*는 긴 Promise chain을 좀더 보기 좋게 작성할 수 있게 도와줄 뿐이지, 복잡한 비동기 처리를 하려면 Promise의 개념을 꼭 이해해야 한다. 일단 Promise에 완전히 익숙해지기 전에는 **당장 *async/await*을 쓰는 것 보다 오히려 더 헷갈리겠지만** Promise chain을 작성하면서 헤매는걸 권장한다.