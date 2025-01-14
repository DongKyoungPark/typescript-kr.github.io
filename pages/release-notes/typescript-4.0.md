---
title: TypeScript 4.0
layout: docs
permalink: /docs/handbook/release-notes/typescript-4-0.html
oneline: TypeScript 4.0 Release Notes
---

## 가변 인자 튜플 타입 (Variadic Tuple Types)

배열이나 튜플 타입 두 개를 결합하여 새로운 배열을 만드는 JavaScript의 `concat` 함수에 대해서 생각해봅시다.

```js
function concat(arr1, arr2) {
  return [...arr1, ...arr2];
}
```

그리고, 배열이나 튜플을 변수로 입력받아 첫 번째 원소를 제외한 나머지를 반환하는 `tail` 함수에 대해서도 생각해봅시다.

```js
function tail(arg) {
  const [_, ...result] = arg;
  return result;
}
```

TypeScript에서는 이 두 함수의 타입을 어떻게 정의할 수 있을까요?

`concat`의 경우, 이전 버전에서는 여러 개의 오버로드를 작성하는 방법이 유일했습니다.

```ts
function concat(arr1: [], arr2: []): [];
function concat<A>(arr1: [A], arr2: []): [A];
function concat<A, B>(arr1: [A, B], arr2: []): [A, B];
function concat<A, B, C>(arr1: [A, B, C], arr2: []): [A, B, C];
function concat<A, B, C, D>(arr1: [A, B, C, D], arr2: []): [A, B, C, D];
function concat<A, B, C, D, E>(arr1: [A, B, C, D, E], arr2: []): [A, B, C, D, E];
function concat<A, B, C, D, E, F>(arr1: [A, B, C, D, E, F], arr2: []): [A, B, C, D, E, F];)
```

음... 네, 이 오버로드들의 두 번째 배열은 전부 비어있습니다.
이때, `arr2`가 하나의 인자를 가지고 있는 경우를 추가해봅시다.

<!-- prettier-ignore -->
```ts
function concat<A2>(arr1: [], arr2: [A2]): [A2];
function concat<A1, A2>(arr1: [A1], arr2: [A2]): [A1, A2];
function concat<A1, B1, A2>(arr1: [A1, B1], arr2: [A2]): [A1, B1, A2];
function concat<A1, B1, C1, A2>(arr1: [A1, B1, C1], arr2: [A2]): [A1, B1, C1, A2];
function concat<A1, B1, C1, D1, A2>(arr1: [A1, B1, C1, D1], arr2: [A2]): [A1, B1, C1, D1, A2];
function concat<A1, B1, C1, D1, E1, A2>(arr1: [A1, B1, C1, D1, E1], arr2: [A2]): [A1, B1, C1, D1, E1, A2];
function concat<A1, B1, C1, D1, E1, F1, A2>(arr1: [A1, B1, C1, D1, E1, F1], arr2: [A2]): [A1, B1, C1, D1, E1, F1, A2];
```

이런 오버로딩 함수들은 분명 비합리적입니다.
안타깝게도, `tail` 함수를 타이핑할 때도 이와 비슷한 문제에 직면하게 됩니다.

이것은 "천 개의 오버로드로 인한 죽음(death by a thousand overloads)"의 하나의 경우이며, 심지어 대부분 문제를 해결하지도 못합니다.
우리가 작성하고자 하는 만큼의 오버로드에 한해서만 올바른 타입을 제공합니다.
포괄적인 케이스를 만들고 싶다면, 다음과 같은 오버로드가 필요합니다.

```ts
function concat<T, U>(arr1: T[], arr2: U[]): Array<T | U>;
```

그러나 위 시그니처는 튜플을 사용할 때 입력 길이나 요소 순서에 대한 어떤 것도 처리하지 않습니다.

TypeScript 4.0은 타입 추론 개선을 포함한 두 가지 핵심적인 변화를 도입해 이러한 타이핑을 가능하도록 만들었습니다.

첫 번째 변화는 튜플 타입 구문의 스프레드 연산자에서 제네릭 타입을 사용할 수 있다는 점입니다.
우리가 작동하는 실제 타입을 모르더라도 튜플과 배열에 대한 고차함수를 표현할 수 있다는 뜻입니다.
이러한 튜플 타입에서 제네릭 스프레드 연산자가 인스턴스화(혹은, 실제 타입으로 대체)되면 또 다른 배열이나 튜플 타입 세트를 생산할 수 있습니다.

예를 들어, `tail` 같은 함수를 "천 개의 오버로드로 인한 죽음(death by a thousand overloads)"이슈 없이 타이핑 할 수 있게 됩니다.

```ts
function tail<T extends any[]>(arr: readonly [any, ...T]) {
  const [_ignored, ...rest] = arr;
  return rest;
}

const myTuple = [1, 2, 3, 4] as const;
const myArray = ["hello", "world"];

const r1 = tail(myTuple);
//    ^ = const r1: [2, 3, 4]

const r2 = tail([...myTuple, ...myArray] as const);
//    ^ = const r2: [2, 3, 4, ...string[]]
```

두 번째 변화는 나머지 요소가 끝뿐만 아니라 튜플의 어느 곳에서도 발생할 수 있다는 것입니다.

```ts
type Strings = [string, string];
type Numbers = [number, number];

type StrStrNumNumBool = [...Strings, ...Numbers, boolean];
//   ^ = type StrStrNumNumBool = [string, string, number, number, boolean]
```

이전에는, TypeScript는 다음과 같은 오류를 생성했었습니다:

```
A rest element must be last in a tuple type.
```

TypeScript 4.0에서는 이러한 제한이 완화되었습니다.

길이가 정해지지 않은 타입을 확장하려고할 때, 결과의 타입은 제한되지 않으며, 다음 모든 요소가 결과의 나머지 요소 타입에 포함되는 점에 유의하시기 바랍니다.

```ts
type Strings = [string, string];
type Numbers = number[];

type Unbounded = [...Strings, ...Numbers, boolean];
//   ^ = type Unbounded = [string, string, ...(number | boolean)[]]
```

이 두 가지 동작을 함께 결합하여, `concat`에 대해 타입이 제대로 정의된 시그니처를 작성할 수 있습니다.

```ts
type Arr = readonly any[];

function concat<T extends Arr, U extends Arr>(arr1: T, arr2: U): [...T, ...U] {
  return [...arr1, ...arr2];
}
```

하나의 시그니처가 조금 길더라도, 반복할 필요가 없는 하나의 시그니처일 뿐이며, 모든 배열과 튜플에서 예측 가능한 행동을 제공합니다.

이 기능은 그 자체만으로도 훌륭하지만, 조금 더 정교한 시나리오에서도 빛을 발합니다.
예를 들어,[함수의 매개변수를 부분적으로 적용하여 새로운 함수를 반환하는](https://en.wikipedia.org/wiki/Partial_application) `partialCall` 함수가 있다고 생각해봅시다.
`partialCall`은 다음과 같은 함수를 가집니다. - `f`가 예상하는 몇 가지 인수와 함께 `f`라고 지정하겠습니다.
그 후, `f`가 여전히 필요로 하는 다른 인수를 가지고, 그것을 받을 때 `f`를 호출하는 새로운 함수를 반환합니다.

```js
function partialCall(f, ...headArgs) {
  return (...tailArgs) => f(...headArgs, ...tailArgs);
}
```

TypeScript 4.0은 나머지 파라미터들과 튜플 원소들에 대한 추론 프로세스를 개선하여 타입을 지정할 수 있고 "그냥 동작"하도록 할 수 있습니다.

```ts
type Arr = readonly unknown[];

function partialCall<T extends Arr, U extends Arr, R>(
  f: (...args: [...T, ...U]) => R,
  ...headArgs: T
) {
  return (...tailArgs: U) => f(...headArgs, ...tailArgs);
}
```

이 경우, `partialCall`은 처음에 취할 수 있는 파라미터와 할 수 없는 파라미터를 파악하고, 남은 것들은 적절히 수용하고 거부하는 함수들을 반환합니다.

```ts
// @errors: 2345 2554 2554 2345
type Arr = readonly unknown[];

function partialCall<T extends Arr, U extends Arr, R>(
  f: (...args: [...T, ...U]) => R,
  ...headArgs: T
) {
  return (...tailArgs: U) => f(...headArgs, ...tailArgs);
}
// ---cut---
const foo = (x: string, y: number, z: boolean) => {};

const f1 = partialCall(foo, 100);

const f2 = partialCall(foo, "hello", 100, true, "oops");

// 작동합니다!
const f3 = partialCall(foo, "hello");
//    ^ = const f3: (y: number, z: boolean) => void

// f3으로 뭘 할 수 있을까요?

// 작동합니다!
f3(123, true);

f3();

f3(123, "hello");
```

가변 인자 튜플 타입은 특히 기능 구성과 관련하여 많은 새로운 흥미로운 패턴을 가능하게 합니다.
우리는 JavaScript에 내장된 `bind` 메서드의 타입 체킹을 더 잘하기 위해 이를 활용할 수 있을 것이라고 기대합니다.
몇 가지 다른 추론 개선 및 패턴들도 여기에 포함되어 있으며, 가변 인자 튜플에 대해 더 알아보고 싶다면, [the pull request](https://github.com/microsoft/TypeScript/pull/39094)를 참고해보세요.

## 라벨링된 튜플 요소 (Labeled Tuple Elements)

튜플 타입과 매개 변수 목록에 대해 개선하는 것은 일반적인 JavaScript 관용구에 대한 타입 유효성 검사를 강화시켜주기 때문에 중요합니다 - 실제로 인수 목록을 자르고 다른 함수로 전달만 해주면 됩니다.
나머지 매개 변수(rest parameter)에 튜플 타입을 사용할 수 있다는 생각은 아주 중요합니다.

예를 들어, 튜플 타입을 나머지 매개 변수로 사용하는 다음 함수는...

```ts
function foo(...args: [string, number]): void {
  // ...
}
```

...다음 함수와 다르지 않아야 합니다...

```ts
function foo(arg0: string, arg1: number): void {
  // ...
}
```

...`foo`의 모든 호출자에 대해서도.

```ts
// @errors: 2554
function foo(arg0: string, arg1: number): void {
  // ...
}
// ---cut---
foo("hello", 42);

foo("hello", 42, true);
foo("hello");
```

그러나 차이점이 보이기 시작한 부분은: 가독성입니다.
첫 번째 예시에서는, 첫 번째와 두 번째 요소에 대한 매개 변수 이름이 없습니다.
타입-검사에는 전혀 영향이 없지만, 튜플 위치에 라벨이 없는 것은 사용하기 어렵게 만듭니다 - 의도를 전달하기 어렵습니다.

TypeScript 4.0에서 튜플 타입에 라벨을 제공하는 이유입니다.

```ts
type Range = [start: number, end: number];
```

매개 변수 목록과 튜플 타입 사이의 연결을 강화하기 위해, 나머지 요소와 선택적 요소에 대한 구문이 매개 변수 목록의 구문을 반영합니다.

```ts
type Foo = [first: number, second?: string, ...rest: any[]];
```

라벨링 된 튜플을 사용할 때는 몇 가지 규칙이 있습니다.
하나는 튜플 요소를 라벨링 할 때, 튜플에 있는 다른 모든 요소들 역시 라벨링 되어야 합니다.

```ts
// @errors: 5084
type Bar = [first: string, number];
```

당연하게도 - 라벨은 구조 분해할 때 변수 이름을 다르게 지정할 필요가 없습니다.
이것은 순전히 문서화와 도구를 위해 필요합니다.

```ts
function foo(x: [first: string, second: number]) {
    // ...

    // 주의: 'first'와 'second'에 대해 이름 지을 필요 없음
    const [a, b] = x;
    a
//  ^ = const a: string
    b
//  ^ = const b: number
}
```

전반적으로, 라벨링 된 튜플은 안전한 타입 방식으로 오버로드를 구현하는 것과 튜플과 인수 목록의 패턴을 활용할 때 편리합니다.
사실, TypeScript 에디터 지원은 가능한 경우 오버로드로 표시하려 합니다.

![라벨링된 튜플의 유니언을 매개변수 목록에서처럼 두 가지 시그니처로 보여주는 시그니처 도움말](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2020/08/signatureHelpLabeledTuples.gif)

더 알고 싶으시면, 라벨링된 튜플 요소에 대한 [풀 리퀘스트](https://github.com/microsoft/TypeScript/pull/38234)를 확인해보세요

## 생성자로부터 클래스 프로퍼티 타입 추론하기 (Class Property Inference from Constructors)

TypeScript 4.0에서는 `noImplicitAny`가 활성화되었을 때 클래스 내의 프로퍼티 타입을 결정하기 위해 제어 흐름 분석을 사용할 수 있습니다.  

<!--prettier-ignore -->
```ts
class Square {
  // 이전에 any로 추론했습니다.
  area;
// ^?
  sideLength;
// ^?
  constructor(sideLength: number) {
    this.sideLength = sideLength;
    this.area = sideLength ** 2;
  }
}
```

생성자의 모든 경로가 인스턴스 멤버에 할당한 것이 아닐 경우, 프로퍼티는 잠재적으로 `undefined`가 됩니다.

<!--prettier-ignore -->
```ts
// @errors: 2532
class Square {
  sideLength;
// ^?

  constructor(sideLength: number) {
    if (Math.random()) {
      this.sideLength = sideLength;
    }
  }

  get area() {
    return this.sideLength ** 2;
  }
}
```

더 많은 내용이 있는 경우(e.g. `initialize` 메서드 등이 있는 경우), `strictPropertyInitialization` 모드에서는 확정적 할당 단언(`!`)에 따라 명시적으로 타입을 선언해야 합니다.

```ts
class Square {
  // 확정적 할당 단언
  //        v
  sideLength!: number;
  //         ^^^^^^^^
  // 타입 표기

  constructor(sideLength: number) {
    this.initialize(sideLength);
  }

  initialize(sideLength: number) {
    this.sideLength = sideLength;
  }

  get area() {
    return this.sideLength ** 2;
  }
}
```

더 자세히 알고 싶다면, [코드를 실행하는 Pull Request를 보세요](https://github.com/microsoft/TypeScript/pull/379200).

## Short-Circuiting Assignment Operators

JavaScript, and a lot of other languages, support a set of operators called _compound assignment_ operators.
Compound assignment operators apply an operator to two arguments, and then assign the result to the left side.
You may have seen these before:

```ts
// Addition
// a = a + b
a += b;

// Subtraction
// a = a - b
a -= b;

// Multiplication
// a = a * b
a *= b;

// Division
// a = a / b
a /= b;

// Exponentiation
// a = a ** b
a **= b;

// Left Bit Shift
// a = a << b
a <<= b;
```

So many operators in JavaScript have a corresponding assignment operator!
Up until recently, however, there were three notable exceptions: logical _and_ (`&&`), logical _or_ (`||`), and nullish coalescing (`??`).

That's why TypeScript 4.0 supports a new ECMAScript feature to add three new assignment operators: `&&=`, `||=`, and `??=`.

These operators are great for substituting any example where a user might write code like the following:

```ts
a = a && b;
a = a || b;
a = a ?? b;
```

Or a similar `if` block like

```ts
// could be 'a ||= b'
if (!a) {
  a = b;
}
```

There are even some patterns we've seen (or, uh, written ourselves) to lazily initialize values, only if they'll be needed.

```ts
let values: string[];
(values ?? (values = [])).push("hello");

// After
(values ??= []).push("hello");
```

(look, we're not proud of _all_ the code we write...)

On the rare case that you use getters or setters with side-effects, it's worth noting that these operators only perform assignments if necessary.
In that sense, not only is the right side of the operator "short-circuited" - the assignment itself is too.

```ts
obj.prop ||= foo();

// roughly equivalent to either of the following

obj.prop || (obj.prop = foo());

if (!obj.prop) {
    obj.prop = foo();
}
```

[Try running the following example](https://www.typescriptlang.org/play?ts=Nightly#code/MYewdgzgLgBCBGArGBeGBvAsAKBnmA5gKawAOATiKQBQCUGO+TMokIANkQHTsgHUAiYlChFyMABYBDCDHIBXMANoBuHI2Z4A9FpgAlIqXZTgRGAFsiAQg2byJeeTAwAslKgSu5KWAAmIczoYAB4YAAYuAFY1XHwAXwAaWxgIEhgKKmoAfQA3KXYALhh4EA4iH3osWM1WCDKePkFUkTFJGTlFZRimOJw4mJwAM0VgKABLcBhB0qCqplr63n4BcjGCCVgIMd8zIjz2eXciXy7k+yhHZygFIhje7BwFzgblgBUJMdlwM3yAdykAJ6yBSQGAeMzNUTkU7YBCILgZUioOBIBGUJEAHwxUxmqnU2Ce3CWgnenzgYDMACo6pZxpYIJSOqDwSkSFCYXC0VQYFi0NMQHQVEA) to see how that differs from _always_ performing the assignment.

```ts
const obj = {
    get prop() {
        console.log("getter has run");

        // Replace me!
        return Math.random() < 0.5;
    },
    set prop(_val: boolean) {
        console.log("setter has run");
    }
};

function foo() {
    console.log("right side evaluated");
    return true;
}

console.log("This one always runs the setter");
obj.prop = obj.prop || foo();

console.log("This one *sometimes* runs the setter");
obj.prop ||= foo();
```

We'd like to extend a big thanks to community member [Wenlu Wang](https://github.com/Kingwl) for this contribution!

For more details, you can [take a look at the pull request here](https://github.com/microsoft/TypeScript/pull/37727).
You can also [check out TC39's proposal repository for this feature](https://github.com/tc39/proposal-logical-assignment/).

## `unknown` on `catch` Clause Bindings

Since the beginning days of TypeScript, `catch` clause variables have always been typed as `any`.
This meant that TypeScript allowed you to do anything you wanted with them.

```ts
try {
  // Do some work
} catch (x) {
  // x has type 'any' - have fun!
  console.log(x.message);
  console.log(x.toUpperCase());
  x++;
  x.yadda.yadda.yadda();
}
```

The above has some undesirable behavior if we're trying to prevent _more_ errors from happening in our error-handling code!
Because these variables have the type `any` by default, they lack any type-safety which could have errored on invalid operations.

That's why TypeScript 4.0 now lets you specify the type of `catch` clause variables as `unknown` instead.
`unknown` is safer than `any` because it reminds us that we need to perform some sorts of type-checks before operating on our values.

<!--prettier-ignore -->
```ts
// @errors: 2571
try {
  // ...
} catch (e: unknown) {
  // Can't access values on unknowns
  console.log(e.toUpperCase());

  if (typeof e === "string") {
    // We've narrowed 'e' down to the type 'string'.
    console.log(e.toUpperCase());
  }
}
```

While the types of `catch` variables won't change by default, we might consider a new `--strict` mode flag in the future so that users can opt in to this behavior.
In the meantime, it should be possible to write a lint rule to force `catch` variables to have an explicit annotation of either `: any` or `: unknown`.

For more details you can [peek at the changes for this feature](https://github.com/microsoft/TypeScript/pull/39015).

## Custom JSX Factories

When using JSX, a [_fragment_](https://reactjs.org/docs/fragments.html) is a type of JSX element that allows us to return multiple child elements.
When we first implemented fragments in TypeScript, we didn't have a great idea about how other libraries would utilize them.
Nowadays most other libraries that encourage using JSX and support fragments have a similar API shape.

In TypeScript 4.0, users can customize the fragment factory through the new `jsxFragmentFactory` option.

As an example, the following `tsconfig.json` file tells TypeScript to transform JSX in a way compatible with React, but switches each factory invocation to `h` instead of `React.createElement`, and uses `Fragment` instead of `React.Fragment`.

```json5
{
  compilerOptions: {
    target: "esnext",
    module: "commonjs",
    jsx: "react",
    jsxFactory: "h",
    jsxFragmentFactory: "Fragment",
  },
}
```

In cases where you need to have a different JSX factory on a per-file basis<!-- (maybe you like to ship React, Preact, and Inferno to give a blazing fast experience) -->, you can take advantage of the new `/** @jsxFrag */` pragma comment.
For example, the following...

```tsx
// @noErrors
// Note: these pragma comments need to be written
// with a JSDoc-style multiline syntax to take effect.

/** @jsx h */
/** @jsxFrag Fragment */

import { h, Fragment } from "preact";

export const Header = (
  <>
    <h1>Welcome</h1>
  </>
);
```

...will get transformed to this output JavaScript...

```tsx
// @noErrors
// @showEmit
// Note: these pragma comments need to be written
// with a JSDoc-style multiline syntax to take effect.

/** @jsx h */
/** @jsxFrag Fragment */

import { h, Fragment } from "preact";

export const Header = (
  <>
    <h1>Welcome</h1>
  </>
);
```

We'd like to extend a big thanks to community member [Noj Vek](https://github.com/nojvek) for sending this pull request and patiently working with our team on it.

You can see that [the pull request](https://github.com/microsoft/TypeScript/pull/38720) for more details!

## Speed Improvements in `build` mode with `--noEmitOnError`

Previously, compiling a program after a previous compile with errors under `--incremental` would be extremely slow when using the `--noEmitOnError` flag.
This is because none of the information from the last compilation would be cached in a `.tsbuildinfo` file based on the `--noEmitOnError` flag.

TypeScript 4.0 changes this which gives a great speed boost in these scenarios, and in turn improves `--build` mode scenarios (which imply both `--incremental` and `--noEmitOnError`).

For details, [read up more on the pull request](https://github.com/microsoft/TypeScript/pull/38853).

## `--incremental` with `--noEmit`

TypeScript 4.0 allows us to use the `--noEmit` flag when while still leveraging `--incremental` compiles.
This was previously not allowed, as `--incremental` needs to emit a `.tsbuildinfo` files; however, the use-case to enable faster incremental builds is important enough to enable for all users.

For more details, you can [see the implementing pull request](https://github.com/microsoft/TypeScript/pull/39122).

## Editor Improvements

The TypeScript compiler doesn't only power the editing experience for TypeScript itself in most major editors - it also powers the JavaScript experience in the Visual Studio family of editors and more.
For that reason, much of our work focuses on improving editor scenarios - the place you spend most of your time as a developer.

Using new TypeScript/JavaScript functionality in your editor will differ depending on your editor, but

* Visual Studio Code supports [selecting different versions of TypeScript](https://code.visualstudio.com/docs/typescript/typescript-compiling#_using-the-workspace-version-of-typescript). Alternatively, there's the [JavaScript/TypeScript Nightly Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-next) to stay on the bleeding edge (which is typically very stable).
* Visual Studio 2017/2019 have [the SDK installers above] and [MSBuild installs](https://www.nuget.org/packages/Microsoft.TypeScript.MSBuild).
* Sublime Text 3 supports [selecting different versions of TypeScript](https://github.com/microsoft/TypeScript-Sublime-Plugin#note-using-different-versions-of-typescript)

You can check out a partial [list of editors that have support for TypeScript](https://github.com/Microsoft/TypeScript/wiki/TypeScript-Editor-Support) to learn more about whether your favorite editor has support to use new versions.

### Convert to Optional Chaining

Optional chaining is a recent feature that's received a lot of love.
That's why TypeScript 4.0 brings a new refactoring to convert common patterns to take advantage of [optional chaining](https://devblogs.microsoft.com/typescript/announcing-typescript-3-7/#optional-chaining) and [nullish coalescing](https://devblogs.microsoft.com/typescript/announcing-typescript-3-7/#nullish-coalescing)!

![Converting `a && a.b.c && a.b.c.d.e.f()` to `a?.b.c?.d.e.f.()`](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2020/08/convertToOptionalChain-4-0.gif)

Keep in mind that while this refactoring doesn't _perfectly_ capture the same behavior due to subtleties with truthiness/falsiness in JavaScript, we believe it should capture the intent for most use-cases, especially when TypeScript has more precise knowledge of your types.

For more details, [check out the pull request for this feature](https://github.com/microsoft/TypeScript/pull/39135).

### `/** @deprecated */` Support

TypeScript's editing support now recognizes when a declaration has been marked with a `/** @deprecated *` JSDoc comment.
That information is surfaced in completion lists and as a suggestion diagnostic that editors can handle specially.
In an editor like VS Code, deprecated values are typically displayed a strike-though style ~~like this~~.

![Some examples of deprecated declarations with strikethrough text in the editor](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2020/06/deprecated_4-0.png)

This new functionality is available thanks to [Wenlu Wang](https://github.com/Kingwl).
See [the pull request](https://github.com/microsoft/TypeScript/pull/38523) for more details.

### Partial Semantic Mode at Startup

We've heard a lot from users suffering from long startup times, especially on bigger projects.
The culprit is usually a process called _program construction_.
This is the process of starting with an initial set of root files, parsing them, finding their dependencies, parsing those dependencies, finding those dependencies' dependencies, and so on.
The bigger your project is, the longer you'll have to wait before you can get basic editor operations like go-to-definition or quick info.

That's why we've been working on a new mode for editors to provide a _partial_ experience until the full language service experience has loaded up.
The core idea is that editors can run a lightweight partial server that only looks at the current files that the editor has open.

It's hard to say precisely what sorts of improvements you'll see, but anecdotally, it used to take anywhere between _20 seconds to a minute_ before TypeScript would become fully responsive on the Visual Studio Code codebase.
In contrast, **our new partial semantic mode seems to bring that delay down to just a few seconds**.
As an example, in the following video, you can see two side-by-side editors with TypeScript 3.9 running on the left and TypeScript 4.0 running on the right.

<video loop autoplay muted style="width:100%;height:100%;" src="https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2020/08/partialModeFast.mp4">
</video>

When restarting both editors on a particularly large codebase, the one with TypeScript 3.9 can't provide completions or quick info at all.
On the other hand, the editor with TypeScript 4.0 can _immediately_ give us a rich experience in the current file we're editing, despite loading the full project in the background.

Currently the only editor that supports this mode is [Visual Studio Code](http://code.visualstudio.com/) which has some UX improvements coming up in [Visual Studio Code Insiders](http://code.visualstudio.com/insiders).
We recognize that this experience may still have room for polish in UX and functionality, and we have [a list of improvements](https://github.com/microsoft/TypeScript/issues/39035) in mind.
We're looking for more feedback on what you think might be useful.

For more information, you can [see the original proposal](https://github.com/microsoft/TypeScript/issues/37713), [the implementing pull request](https://github.com/microsoft/TypeScript/pull/38561), along with [the follow-up meta issue](https://github.com/microsoft/TypeScript/issues/39035).

### Smarter Auto-Imports

Auto-import is a fantastic feature that makes coding a lot easier; however, every time auto-import doesn't seem to work, it can throw users off a lot.
One specific issue that we heard from users was that auto-imports didn't work on dependencies that were written in TypeScript - that is, until they wrote at least one explicit import somewhere else in their project.

Why would auto-imports work for `@types` packages, but not for packages that ship their own types?
It turns out that auto-imports only work on packages your project _already_ includes.
Because TypeScript has some quirky defaults that automatically add packages in `node_modules/@types` to your project, _those_ packages would be auto-imported.
On the other hand, other packages were excluded because crawling through all your `node_modules` packages can be _really_ expensive.

All of this leads to a pretty lousy getting started experience for when you're trying to auto-import something that you've just installed but haven't used yet.

TypeScript 4.0 now does a little extra work in editor scenarios to include the packages you've listed in your `package.json`'s `dependencies` (and `peerDependencies`) fields.
The information from these packages is only used to improve auto-imports, and doesn't change anything else like type-checking.
This allows us to provide auto-imports for all of your dependencies that have types, without incurring the cost of a complete `node_modules` search.

In the rare cases when your `package.json` lists more than ten typed dependencies that haven't been imported yet, this feature automatically disables itself to prevent slow project loading.
To force the feature to work, or to disable it entirely, you should be able to configure your editor.
For Visual Studio Code, this is the "Include Package JSON Auto Imports" (or `typescript.preferences.includePackageJsonAutoImports`) setting.

![Configuring 'include package JSON auto imports'](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2020/08/configurePackageJsonAutoImports4-0.png)
For more details, you can see the [proposal issue](https://github.com/microsoft/TypeScript/issues/37812) along with [the implementing pull request](https://github.com/microsoft/TypeScript/pull/38923).

## Our New Website

[The TypeScript website](https://www.typescriptlang.org/) has recently been rewritten from the ground up and rolled out!

![A screenshot of the new TypeScript website](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2020/08/ts-web.png)

[We already wrote a bit about our new site](https://devblogs.microsoft.com/typescript/announcing-the-new-typescript-website/), so you can read up more there; but it's worth mentioning that we're still looking to hear what you think!
If you have questions, comments, or suggestions, you can [file them over on the website's issue tracker](https://github.com/microsoft/TypeScript-Website).

## Breaking Changes

### `lib.d.ts` Changes

Our `lib.d.ts` declarations have changed - most specifically, types for the DOM have changed.
The most notable change may be the removal of [`document.origin`](https://developer.mozilla.org/en-US/docs/Web/API/Document/origin) which only worked in old versions of IE and Safari
MDN recommends moving to [`self.origin`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/origin).

### Properties Overriding Accessors (and vice versa) is an Error

Previously, it was only an error for properties to override accessors, or accessors to override properties, when using `useDefineForClassFields`; however, TypeScript now always issues an error when declaring a property in a derived class that would override a getter or setter in the base class.

```ts
// @errors: 1049 2610
class Base {
  get foo() {
    return 100;
  }
  set foo(value) {
    // ...
  }
}

class Derived extends Base {
  foo = 10;
}
```

```ts
// @errors: 2611
class Base {
  prop = 10;
}

class Derived extends Base {
  get prop() {
    return 100;
  }
}
```

See more details on [the implementing pull request](https://github.com/microsoft/TypeScript/pull/37894).

### Operands for `delete` must be optional

When using the `delete` operator in `strictNullChecks`, the operand must now be `any`, `unknown`, `never`, or be optional (in that it contains `undefined` in the type).
Otherwise, use of the `delete` operator is an error.

```ts
// @errors: 2790
interface Thing {
  prop: string;
}

function f(x: Thing) {
  delete x.prop;
}
```

See more details on [the implementing pull request](https://github.com/microsoft/TypeScript/pull/37921).

### Usage of TypeScript's Node Factory is Deprecated

Today TypeScript provides a set of "factory" functions for producing AST Nodes; however, TypeScript 4.0 provides a new node factory API.
As a result, for TypeScript 4.0 we've made the decision to deprecate these older functions in favor of the new ones.

For more details, [read up on the relevant pull request for this change](https://github.com/microsoft/TypeScript/pull/35282).
