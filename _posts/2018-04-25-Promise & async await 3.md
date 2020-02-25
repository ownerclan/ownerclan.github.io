---
layout: post
title:  "Promise & async/await 3회차(完)"
date:   2018-04-25
description: "맨땅에서 새(new) Promise 만들기"
---
> 마지막 세션을 시작하기 전에 앞서, Promise든 async/await든 모두 문법 설탕에 불과하다는 것을 상기하고 싶다. 얘들 없이 Callback만 가지고도 모든 문제를 해결할 수 있다. 다만 코드가 훨씬 복잡해질 뿐이다. 동시에 이건 Promise나 async/await을 알기 전에 못 만든 프로그램을 얘네를 배웠다고 만들 수 있게 되진 않는다는 걸 뜻한다. 여차하면 Callback Hell에 빠진 프로그램이라도 만들 수 있어야 한다. 일부러 Promise 쓰지말고 고통받으란 건 아니고, JavaScript의 이벤트 처리 방식에 대해 제대로 이해하게 가장 중요하다는 것이다.

## async/await

```typescript
import * as request from "request-promise-native";

async function scrap() {
    try {
        const google = await request("http://www.google.com");
        console.log(google);
        const nosite = await request("http://www.없는사이트.com/");
        console.log(nosite);
    } catch (ex) {
        console.error(ex)
    }
}
```

```typescript
function scrap() {
    return request("http://www.google.com")
        .then((google) => console.log(google))
        .then(() => request("http://www.없는사이트.com"))
        .then((nosite) => console.log(nosite))
        .catch((ex) => console.error(ex));
}
```

위 두 코드는 똑같이 동작한다. 

*async/await*은 *try...catch...finally*와 같이 쓸 수 있다. 이 말인 즉슨 비동기 함수 내부에서 *throw*된 에러도 *catch*로 간다는 것이다. *finally*도 우리 생각대로 동작한다. 함수가 비동기든 동기든 *await*만 해주면 우리가 원하는 상식적인 방식으로 당연한 듯이 동작한다. 보다시피 위의 *async/await*를 이용한 코드는 *async/await*만 가리고 보면 비동기가 없던 좋았던 시절의 코드와 똑같다.  

*async/await*에 대해서 더 길게 할 말이 없다. 한줄로 정리하면 다음과 같다.

```
async는 함수가 Promise Generator란 뜻이고, await은 Promise를 yield한다는 뜻이다.
```

이게 잘 이해가 안되면 이전 자료를 보면 된다.

## 맨땅에서 Promise 만들기

지금까지 여러 예제에서 예시로 써먹은 `delay`의 구현을 살펴보자.

```typescript
function delay(time: number) {
    return new Promise<void>((resolve) => setTimeout(resolve, time));
}
```

Promise를 await한다는 `resolve`가 호출되기를 기다린다는 것이다. 자세히 보면 `resolve`는 Callback인걸 알 수 있다. 하지만 뭔가 다르다. 이 경우엔 우리가 Callback을 넘겨주는게 아니라, `resolve`라는 **Callback을 넘겨 받는다!** 여태 남이 만든 함수에 Callback을 넘겨주면서, 내가 넘겨준 함수가 언제 불릴지, 언젠가 불리긴 할지 때문에 밤잠을 설친적이 있을것이다? 그런데 이젠 상황이 반대가 되었다.

사실 Callback Hell이 정신건강에 안 좋아서 그렇지 Callback 자체는 비동기 프로그래밍을 하는 가장 유연한 방법이다. Promise로 모든 Callback Hell을 해결할 수는 없지만 그 반대는 성립한다.  `resolve`를 가지고 뭘 할 수 있는지를 살펴보자.

아래는 100,000으로 시작해서 통장에 100,000,000이상 쌓일때까지 복권을 사는 프로그램이다.

```typescript
(async () => {
    let money = 100000;

    async function buyLottery() {
        if (money < 10000) {
            throw new Error("파산!");
        }
        money -= 10000;
        return new Promise<number>(async (resolve, reject) => {
            await new delay(1000);
            const random = Math.random();
            if( random < 0.5 ) {
                const win = Math.pow(10, Math.floor(1/random));
                resolve(win);
            } else {
                reject(new Error("꽝!"));
            }
        });
    }
    
    while (true) {
        if (money > 100000000) {
            console.log("ㅎㅎ")
            break;
        }

        try {
            const earn = await buyLottery();
            money += earn;
            console.log(earn, money)   
        } catch (ex) {
            if (ex.message === "파산!") {
                console.error("ㅠㅠ");
                break;
            }
            console.log(ex.message, money);
        }
    }
})();
```

아래 `while (true)`로 시작하는 루프를 보자.  `buyLottery`는 *await* 했을 때 복권 당첨금액을 돌려주기 때문에 `money`에 그걸 더한다.`buyLottery` 정의 부분을 보면 `random`의 값이 0.5 이하일 때(즉 절반의 확률로) `resolve`에 당첨금을 인자로 넣어서 호출하는데, 바로 그 값이 `earn`에 전달되는 것이다.

운이 좀 나빠서 당첨이 안 됐을 때는 `reject`를 호출하게 되는데 얘도 마찬가지로 Callback이다. `resolve`가 *return*이라면 `reject`는 *throw*이다. 그래서 *catch*로 가게 된다. `ex`는 `reject`에 인자로 넘겨준 `new Error("꽝!")`이 된다. 보다시피  `buyLottery`내의 *throw*와 `reject` 둘다 구분없이 *catch*로 간다!

여기서 한가지 의문이 들 수 있는데, `resolve`와 `reject` 둘다 하면 어떻게 될까? 아니면 `resolve`를 여러번 하면 어떻게 될까? 둘다 여러번할 수도 있겠다.

다행히도 이또한 우리가 원하는 방식대로 동작한다. 제일 먼저 호출된 것만 적용되고 나머지는 무시된다. 생각해보면 당연한 것이 `resolve`를 하면 *await*에 정차했던 버스가 신호를 받고 출발하는데 그 다음에 다른 신호를 줘봤자 무슨 소용이란 말인가? 버스는 이미 떠났다. 

- *await*: 신호등


- `resolve`: 파란불
- `reject`: 우회전

이걸로  `resolve()`와 `reject()`의 의미가 좀더 잘 다가오면 좋겠다. 그럼 이런 맨 처음 들어온 것만 처리하는(사실 그럴 수밖에 없는) Promise의 동작을 이용해서 재미있는걸 해보자.

아래는 `resolve`, `reject`를 모두 사용해 구현한 웹 369 게임이다. `resolve`나 `reject` 둘 중에 먼저 호출된 것만 의미 있는 걸 이용해서 369게임에 시간 제한을 걸 수 있다.

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    <title>369</title>
</head>

<body>
    <div id="app">
        <h1>{{count}}</h1>
        <button v-on:click="next">+</button>
        <button v-on:click="clap">👏</button>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue"></script>
    <script>
        var app = new Vue({
            el: '#app',
            data: {
                count: 1
            },
            methods: {
                next() {
                    this.check(false)
                },
                clap() {
                    this.check(true)
                },
                async start() {
                    this.count = 1;
                    while (true) {
                        try {
                            await new Promise((resolve, reject) => {
                                this.check = (clapped) => {
                                    if (/[369]/
                                        	.test(this.count.toString()) !== clapped) {
                                        reject("틀림!");
                                    } else {
                                        resolve();
                                    }
                                };
                                setTimeout(() => reject("늦음!"), 2000);
                            })
                            this.count++;
                        } catch (ex) {
                            alert(ex);
                            break;
                        }
                    }
                    this.start();
                }
            },
            mounted() {
                this.start();
            }
        })
    </script>
</body>

</html>
```

Promise를 생성하는 부분만 보면 된다. 다시 콜백을 만들어서 `this.check`에 넣는다. `this.check`은 `+` 버튼이나 `👏` 버튼을 눌렀을 때 호출된다. `this.check`은 `reject`나 `resolve`를 호출한다. 즉 이 Promise는 전달된 `resolve`, `reject`를 버튼 이벤트에 다시 전달한 것이다. 사실 `this.resolve`, `this.reject` 같은 걸 만들어도 되지만 별로 좋은 코드는 아니다. 아래에는 `setTimeout`으로 2000ms 후에 `reject`를 날리는 코드가 보인다. 저 Promise 안의 `resolve` 하나와 `reject` 둘 중 가장 먼저 호출되는 것만 처리된다.

 `new Promise`를 배웠다고 뺄 수 있는 모든 Callback을 찾아 없앨 필요는 없다. 여의치 않을땐 Callback으로도 짤 수 있어야하고, 짜놓고 보니 *async/await*으로 바꿀 수 있을것 같으면 그때가서 바꾸면 된다.