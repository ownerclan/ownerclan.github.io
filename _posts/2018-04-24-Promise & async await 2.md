---
layout: post
title:  "Promise & async/await 2회차"
date:   2018-04-24
description: "Promise에서 자주 등장하는 패턴들과 Generator로 async/await 직접 구현하기"
---
## 구구단 외우기

구구단을 출력하는 프로그램을 짜보자.

```typescript
function play(n: number) {
    for (let i = 2; i <= n; i++) {
        for (let j = 2; j <= n; j++) {
            console.log(`${i} * ${j} = ${i * j}`);
        }
    }
}

play(9)
```

당연히 잘 된다. 그럼 이제 1초씩 딜레이를 줘보자.

```typescript
function play(n: number) {
    for (let i = 2; i <= n; i++) {
        for (let j = 2; j <= n; j++) {
            delay1s().then(() => console.log(`${i} * ${j} = ${i * j}`));
        }
    }
}
```

1초 후에 쏟아져 나오는걸 볼 수 있다. 여기서 명심해야 될것은 `then`으로 Promise chain을 이어주지 않으면 순차적으로 실행되지 않는 다는 것이다.

```typescript
function play(n: number) {
    for (let i = 2; i <= n; i++) {
        let promise = Promise.resolve();
        for (let j = 2; j <= n; j++) {
            promise = promise
                .then(delay1s)
                .then(() => console.log(`${i} * ${j} = ${i * j}`));
        }
    }
}
```

이러면 구구단 2단부터 9단까지 동시에 외우는 걸 볼 수 있다. 코드를 다시 보자. Promise는 총 몇 개인가? `play(9)`니까 Promise는 8개 생성된다. 그리고 그 Promise들은 `then`으로 연결되지 않았다. 즉 Promise chain이 하나가 아니라 8개이다!

해결책은 간단한데, 그냥 전부 한 chain으로 연결해버리면 된다.

```typescript
function play(n: number) {
    let promise = Promise.resolve();
    for (let i = 2; i <= n; i++) {
        for (let j = 2; j <= n; j++) {
            promise = promise
                .then(delay1s)
                .then(() => console.log(`${i} * ${j} = ${i * j}`));
        }
    }
}
```

사실 여기까지는 할만 하다. 물론 Promise chain을 매번 머릿속에 그려야하는 게 거슬리긴 하지만, 그냥 `then`으로 이어질때만 순차로 실행되는 것만 명심해도 대부분의 문제를 해결할 수 있다. 

## 실시간 369

다시 369를 해보자. 이번엔 횟수가 정해져있지 않고 틀릴 때까지 한다.

```typescript
function isClap(n: number) {
    return n.toString().match(/[369]/);
}

function godOf369(n: number) {
    return isClap(n) ? "짝" : n;
}

function player(n: number): number | "짝" {
    const clap = isClap(n);
    return (Math.random() < 0.9 ? clap : !clap) ? "짝" : n;
}
```

`player`는 박수를 치던 숫자를 말하던 0.9의 확률로 맞춘다. `godOf369`는 언제나 맞춘다. 두 함수의 결과값을 비교해서 게임을 중단할지 말지 결정할 수 있다.

```typescript
let i = 1;
while (true) {
    const answer = player(i);
    console.log(answer);
    if (answer !== godOf369(i)) {
        break;
    }
    i++;
}
```

여기까진 쉽다. 이제 1초씩 딜레이를 줘보자.

```typescript
let promise = Promise.resolve();
let i = 1;
while (true) {
    promise = promise.then((x) => {
        const answer = player(i);
        console.log(answer);
        if (answer !== godOf369(i)) {
            // 여기에 뭘 넣어야하는가?
        }
    }).then(delay1s);
    i++;
}
```

무한루프를 만들고 말았다. 어떻게 해야 저 루프를 멈출 수 있을까? 사실 저기 주석에 무슨 코드를 넣더라도 무한 369 게임을 멈추지 못한다. 왜냐하면 게임이 계속 될지 끝날지는 **미래**(`then` 이후)에 결정되는데, **현재**(`while (true)` 내부)는 그걸 전혀 모르고 Promise chain을 계속 잇기만 하기 때문이다. 이걸 해결하려면 Promise안에서 Promise chain을 구성해야한다. 즉 Promise를 만드는 Promise함수를 만들어야 한다.

```typescript
function play369(n = 0) {
    const answer = player(n);
    console.log(answer);
    if (answer === godOf369(n)) {
        delay1s().then(() => play369(n + 1));
    }
}
```

이렇게 재귀호출로 구현하면 된다. 근데 매번 이런 식으로 해야하는가?

```typescript
function until(fn: (i: number) => any, n = 0): Promise<any> {
    return Promise.resolve(fn(n))
        .then((gostop) => {
            if (gostop) {
                return until(fn, n + 1);
            }
        });
}
```

그래서 `until`이란 함수를 만들었다. `fn`이 *false*를 돌려줄 때까지 실행하는 함수이다. 이제 `until`을 사용해서 `play369`를 다시 만들어보자.

```typescript
function play369() {
    return until((i) => {
        const answer = player(i);
        console.log(answer);
        if (answer !== godOf369(i)) {
            return false;
        }
        return delay1s().then(() => true);
    }, 1);
}
```

사실 코드가 더 짧아지진 않았지만 재귀호출과 Promise chain에 대해 잊어버릴 수 있게 되었다. 이런 식으로 앞의 구구단도 다시 만들어보자. 구구단은 언제까지 Promise chain이 이어져야할지 미리 알 수 있으므로 굳이 `until`을 사용할 필요가 없다.

```typescript
function range(from, end, fn: (i: number) => any): Promise<void> {
    if (end > from ) {
        return fn(from).then(() => range(from + 1, end, fn));
    } else {
        return Promise.resolve();
    }
}
```

`range`는 `for (let i = from;i <= end; i++)`과 비슷하다.

```typescript
range(2, 10, (i) =>
    range(2, 10, (j) => {
        console.log(`${i} * ${j} = ${i * j}`);
        return delay1s();
    }),
);
```

> `return delay1s()`에서 `return`을 빼면 어떻게 될까?

어찌저찌 해결은 하고 있지만, 매번 비동기 코드를 짤 때마다 머리를 굴려야한다면(막상 실제론 별로 복잡한 것도 아닌데도) 너무 피곤한 일이 아닐 수 없다. 비동기만 아니면 손쉽게 짤 코드들도 마치 프로그래밍을 처음 배우는 것 마냥 새로 생각해야하니 말이다.

[bluebird](http://bluebirdjs.com)와 같은 라이브러리는 이런 패턴들을 위한 함수들을 제공하고 있다. 방금 우리가 만든 `until`이나 `range`같은 함수들 말이다. 그럼 bluebird를 쓰고 그 함수들을 외우란 말인가? 아니다. 앞으로 bluebird를 쓸 일은 없을 것이고 `until`, `range`도 잊어버려도 된다.

## Generator

> 아래 예제를 따라할 때, 또는 실제로 generator를 사용할 땐 `tsconfig.json`에서 `"downlevelIteration": true`를 설정하자. 아니면 `"target": "es6"`로 해도 된다.

```typescript
const infinity = getInfinity();
console.log("난 출력되야함");
for (const i of infinity) {
}
console.log("난 출력되면 안됨");
```

저기서 `난 출력되야함 `은 출력되고 `난 출력되면 안됨`을 출력되지 않게 하려면 어떻게 해야할까? 위의 *for*문을 무한 루프로 만들어 버리면 된다. 그렇게 하려면 `infinity`에 무슨 값을 주면 될까? 즉 `getInfinity`를 어떻게 구현하면 될까?

무한한 길이의 배열을 돌려주면 될까? 무한한 길이의 배열은 어떻게 만드는가? 무한루프를 돌면서 `push`를 하면 될거 같긴 한데, 그러면 `난 출력되야함`도 출력이 안 될 것이다. 대신 메모리가 부족하다는 에러 메시지를 볼 수 있을 것이다.

일단 `infinity`는 배열일 수는 없다. 그러면 무엇이어야 하는가? *for*문을 돌 수 있으면서, 끝이 나지 않아야 한다. 사실 이렇게 묻는 건 반칙인데, **generator**라는 (아마) 듣도보도 못한 게 정답이기 때문이다. 정답은 아래와 같다.

```typescript
function* infinity() {
    while (true) {
        yield;
    }
}
```

일단 `function` 뒤에 * 가 붙어 있는 것에 주목하라. generator를 정의하는 특별한 문법이다. *yield*는 *return*과 비슷한데, 여러번 할 수 있다는 차이가 있다. 즉 *yield*를 해도 generator는 끝나지 않는다. 그럼 어디다 쓰는 놈이란 말인가. *yield*가 값을 반환하도록 해보자.

```typescript
function* numbers() {
    let i = 1;
    while (true) {
        yield i++;
    }
}

for (const i of numbers()) {
    console.log(i);
}
```

실행하면 무한을 향해 1씩 증가하는 수열을 볼 수 있을 것이다.  `numbers`는 무한루프를 돌면서 `i`를 생성(yield)한다. 핵심은 i의 값을 보존하는 것이다. 함수는 *return*을 하거나 마지막 줄에 도달해 끝나게 되면 그 안에 정의했던 값들이 전부 사라지는데(누가 궁금해하는가?), generator는 그 값을 보존하며 여러차례 *yield*를 할 수 있다.

그럼 좀 더 쓸모있는 코드를 짜보자. 아래는 소수를 무한히 생성하는 generator이다.

```typescript
function* primes() {
    let i = 2;
    const primesFound = [];
    loop1:
    while (true) {
        for ( const j of primesFound) {
            if ( i % j === 0 ) {
                i++;
                continue loop1;
            } else if ( i < j * j) {
                break;
            }
        }
        primesFound.push(i);
        yield i++;
    }
}
```

generator는 `i`와 `primesFound`의 값을 보존한다. 그리고 그 값을 다음 소수를 구해서 *yield*하는데 사용한다.

그래서 이게 Promise랑 무슨 상관이냐고?

`number`가 1초마다 숫자를 반환하도록 해보자. 시간이 많다면 지금까지 배운걸 바탕으로 코드를 이렇게 저렇게 끄적여 보자. 곧 불가능하다는 걸 깨달을 텐데, 사실 generator에서 값을 받아오는 방법이 *for*만 있는게 아니다. 아래 코드는 위에 소개한 *for*를 이용한 방법과 정확히 똑같이 동작한다.

```typescript
const i = numbers();
while (true) {
    const {done, value} = i.next();
    if (done) {
        break;
    } else {
        console.log(value);
    }
}
```

`next`는 generator가 *yield*하는 값(`value`)과 generator가 끝났는지(`done`)을 가져온다. 여태 영원히 안 끝나는 generator만 소개했는데 generator도 끝날 수 있다. 그냥 코드 마지막에 도달하거나 중간에 *return*을 하면 된다. 그래서 `done`이 *true*면 generator가 끝난 것이니 *while*문을 나간다.

물론 훨씬 길고 읽기도 어려우니 평소엔 이렇게 쓰지 않는 것이다.

여태 Promise로 이런저런 삽질을 해봤다면 위 코드를 보고 감이 올 것이다. `next`를 `then`에 넣고 `done`에 따라 재귀호출을 해서 Promise chain을 이어나가거나 중단하면 되겠군!

```typescript
function forBy1s(generator: Iterator<any>) {
    const waitUntilYield = () => {
        const i = generator.next();
        if (i.done) {
            return Promise.resolve();
        } else {
            console.log(i.value);
            return delay1s()
                .then(waitUntilYield);

        }
    };
    return waitUntilYield();
}

forBy1s(numbers());
```

`waitUntilYield`는 `generator`가 *yield*를 할 때마다 그 다음에 1초 쉰다음에 다시 *yield*를 기다린다. 저기 다른 generator를 넣어도 다 1초마다 출력되는 걸 볼 수 있다. `forBy1s(primes())`도 해보라.

또 어찌저찌 문제는 해결했지만, 뭐가 나아진 걸까?  일단, `forBy1s(primes())`를 generator 안 쓰고 구현한다고 생각해보면(한번 해보는 것도 괜찮다) 그보다는 이게 그나마 나은 것 같다. 더 중요한 건 이 코드는 Promise와 관련된 패턴에 대해 일반적인 해법을 제시한다는 것이다.

여태까지의 Promise 문제에 대한 해법은 비동기 패턴을 함수로 만들어 분리하고(`until`, `range` 등), 실제로 처리할 작업들을 함수나 generator에서는 비동기와 무관한 작업들을 하도록 하였다. 이 또한, 다양한 문제에 효율적인 해결책이지만, 크롤링 같은 현실의 문제를 해결할 때는 비동기 패턴과 작업이 깔끔하게 분리되지 않는 경우가 많다. generator는 그런 문제에 대해 비교적 깔끔한 해법을 제공한다.

그럼 이제 `printNumbersBy1s`라는 함수를 만들어보자. 이 함수는 `forBy1s(numbers)`와 똑같이 동작해야 한다.

```typescript
function* printNumbersBy1s(): IterableIterator<Promise<void>> {
    const i = 1;
    while (true) {
        console.log(i++);
        yield delay1s();
    }
}
```

이제 `printNumbersBy1s` 안에 숫자를 출력하는 코드와 1초 기다리는 코드가 모두 들어있다. 그런데 Promise를 *yield* 한다는 건 무슨 의미인가?

혹시 이렇게 하면

```typescript
for (const i of printNumbersBy1s()) {
}
```

역시 생각대로 되지 않는다. Promise를 *yield*한다고 나머지가 다 알아서 되는건 아니다. 다만 우리가, 위에서 `forBy1s`를 구현한 것처럼, *yield*된 Promise를 어떻게 잘 해주면 될 것 같지 않은가?

```typescript
function async_(coroutine: () => Iterator<Promise<any>>) {
    const iterator = coroutine();
    const waitUntilYield = () => {
        const i = iterator.next();
        if (i.done) {
            return Promise.resolve(i.value);
        } else {
            return i.value.then(waitUntilYield);
        }
    };
    return waitUntilYield();
}

async_(printNumbersBy1s);
```

> 사실 *async*를 함수명으로 써도 문법적으로 문제가 없지만 혹시 헷갈릴까봐 _를 붙였다.

`async_`는 *yield*된 Promise들을 받아서 Promise chain을 잇는다. 그리고 `i.done === true`면 그만둔다. `async_`가 우리가 찾아해매던, 궁극의 유틸리티 함수라고 할 수 있다. 거의 대부분의 Promise hell을 해결할 수 있다. 숨은 공신은 바로 generator이다. generator안에서 변수들이 유지되기 때문에, 그 값들을 `then`을 통해 넘겨주는 골치아픈 일을 피할 수 있다. 즉 generator는 맥락(context)를 보존한다.

그럼 *async/await*은 안 배워도 되나?

그렇다. 왜냐면 이미 알고 있기 때문이다. 아래 코드에서 `async_`를 *async*로 *yield*를 *await*로 바꿔 읽어보자.

```typescript
async_(function*() {
    const i = 1;
    while (true) {
        console.log(i++);
        yield delay1s();
    }
});
```

```typescript
(async () => {
    const i = 1;
    while (true) {
        console.log(i++);
        await delay1s();
    }
})();
```

async가 `async_`로, generator가 arrow function으로, *yield*가 *await*으로 바뀐것 빼고는 똑같다. 우리는 지금까지 *async/await*의 내부 동작을 배운 것이다. 그걸 사용하는 것은 훨씬 쉬운 일일 것이다. 이젠 *async/await*으로 맘편하게 코딩하다가 헷갈리면 그때 여기로 돌아오면 된다.

> 위에 `async_` 함수의 인자가 `coroutine`라고 되어 있는데, coroutine이 무엇인지 궁금할 수 있다. generator와 *async/await*이 대표적인 coroutine이다. 함수(async)가 실행되다가 중간에 나갔다가 다시 돌아와서(*await*) 실행할수 있는 걸 coroutine이라고 한다. Javascript는 범용적인 coroutine인 generator와 Promise를 이용한 비동기 처리를 위한 coroutine인 *async/await*을 지원하는 것이다.

> 아직 자신이 없다면 구구단, 369, 소수 예제를 generator를 이용해 구현해보자.