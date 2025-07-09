---
title: "타입스크립트에선 Object.assign대신 Spread(...) 연산자를 쓰세요"
description: "얕은 복사와 객체 병합을 목적으로 Object.assign()을 사용합니다. 하지만 타입스크립트에선 대부분의 경우 Spread(...) 연산자를 쓰는 게 더 낫습니다."
pubDate: "2025-07-03 15:08:00"
updatedDate: "2025-07-07 21:31:00"
tags: ["javascript", "typescript"]
ccl: true
---

> [!WARNING]
> Object.assign을 아는 독자를 대상으로 작성한 포스트입니다.

일반적으로 얕은 복사로 객체를 병합한다면 표준 함수인 [Object.assign](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)이 하나의 선택지가 될 수 있다. 하지만 2014년에 [Object Spread(...) 연산자](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax#spread_in_object_literals)가 제안[^1]되었고 2016년에 [타입스크립트 2.1](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html#object-spread-and-rest)에서 등장했다. 자바스크립트에선 ES2018(ES9)에 도입되었다. 도입 이후 왜 `Object.assign`보다 Spread 연산자가 나은지 설명한다.

[^1]: [Github](https://github.com/tc39/proposal-object-rest-spread)
## 강력한 타입 추론
```ts showLineNumbers
interface User {
  name: string;
  birth: Date;
}

const users: User[] = [];
const result = users.map((user) =>
  Object.assign({}, user, {
    birth: user.birth.toISOString(),
  })
);
```
우리는 `result`의 타입이 `{ birth: string; name: string; }[];` 인걸로 기대한다. 하지만 타입스크립트는 `(User & { birth: string; })[];`로 추론되어 `result.birth`는 `Date & string` 타입이다.

이러한 원인은 `Object.assign` 함수 시그니처에서 찾을 수 있다.<br/>
```ts
assign<T extends {}, U, V>(target: T, source1: U, source2: V): T & U & V;
```
리턴 타입이 `T & U & V`라 `{} & User & {birth: string}`이고 최종적으로 `(User & { birth: string; })[];` 타입이 추론된 것이다.

반면에 Spread 연산자는 Object Literal로 평가되므로 우리가 기대한 타입으로 추론된다.
```ts {9}
interface User {
  name: string;
  birth: Date;
}

const users: User[] = [];

const result = users.map((user) => ({
  ...user,
  birth: user.birth.toISOString(),
  })
);

result[0].birth; // string 타입
```
이제 변수 `result`의 타입은 `{ birth: string; name: string; }[];`로 추론된다.
## 항상 덮어쓰는 속성 감지
`tsconfig.json`파일에서 `compilerOptions.strictNullChecks`[^2]를 `true`로 설정하면 사용할 수 있는 기능이다.
[^2]: [strict](https://www.typescriptlang.org/tsconfig/#strict)를 활성화하면 자동으로 strictNullChecks도 활성화된다. 하지만 다른 설정도 같이 활성화되니 주의
```ts showLineNumbers title="main.ts"
interface User {
  name: string;
  birth: Date;
}
const users: User[] = [];

const result = users.map((user) => ({
  // 'name'이(가) 두 번 이상 지정되어 이 사용량을 덮어씁니다.ts(2783)
  // main.ts(14, 3): 이 스프레드는 항상 이 속성을 덮어씁니다.
  name: { // [!code error]
    firstName: user.name.split(" ")[0], // [!code error]
    lastName: user.name.split(" ")[1], // [!code error]
  }, // [!code error]
  ...user,
  birth: user.birth.toISOString(),
}));
```
먼저 등장한 `name`은 Spread 연산 때문에 덮어 쓰인다.

Spread 연산과 타입스크립트 컴파일 옵션을 적절히 조합하면 이러한 실수도 방지할 수 있다.

## `Object.assign`과 비교
모든 상황에서 Spread 연산자를 대체할 수 있는 건 아니다.

### Setter
`Object.assgin`함수의 첫 번째 매개변수인 `target: T`에 Setter가 있고 병합하려는 경우 Setter가 트리거 되지만 Spread 연산은 그렇지 않다. 트리거가 되어야 하는 경우 Spread 연산자를 사용해선 안 된다.
```ts showLineNumbers
const objectAssign = Object.assign(
  {
    set key(key: string) {
      console.log(key);
    },
  },
  { key: "lZa75HrEHw" }
);
// 콘솔에 "lZa75HrEHw"가 출력됨

const objectSpread = {
  set key(key: string) {
      console.log(key);
    },
  ...{key: "lZa75HrEHw"}
};
// 아무것도 출력되지 않음
```
다만 타입스크립트 컴파일러의 `target` 옵션을 ES2018 미만으로 설정하는 경우 `Object.assign`을 사용하는 코드로 트랜스파일링된다. 이 경우 Setter가 동작한다. 

아래는 위 코드를 ES2017로 트랜스파일링한 결과이다.
```ts showLineNumbers
"use strict";
const objectAssign = Object.assign({
    set key(key) {
        console.log(key);
    },
}, { key: "lZa75HrEHw" });
// 콘솔에 "lZa75HrEHw"가 출력됨
const objectSpread = Object.assign({ set key(key) {
        console.log(key);
    } }, { key: "lZa75HrEHw" });
// 아무것도 출력되지 않음
```
이 경우 `"lZa75HrEHw"` 문자열이 두 번 출력된다.
### 성능
크롬 기준 대부분 벤치마크에서 `Object.assign`이 근소하게 더 빠르다.

참고 링크: https://www.measurethat.net/Benchmarks/Show/3816/0/lodash-merge-vs-objectassign-vs-spread-new-obj
