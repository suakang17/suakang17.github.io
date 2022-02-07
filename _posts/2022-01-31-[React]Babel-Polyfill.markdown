---
layout: post
title:  "[React]Babel-Polyfill only one 에러"
date:   2022-01-31 12:49:10 +0900
categories: Devlife
---

# [React]Babel-Polyfill only one 에러
## Babel이란? 
Babel은 자바스크립트 컴파일러로, 주로 ECMAScript 2015+ 코드(넷스케이프에서 javascript의 핵심개발언어를 ECMA 인터내셔널에 제출하고 표준화시켜 발표된 표준 스크립트 기술 규격)를 오래된 브라우저나
환경에서 호환되는 과거 버전의 자바스크립트로 변환시키는데 사용되는 툴체인이다.

우리가 Babel을 이용해 할 수 있는 일에는 
 1. syntax 변환 
 2. 우리의 작업 환경에서 누락된 Polyfill을 타사의 Polyfill을 이용해 대신하기 
 3. 소스코드 변환(codemods) 등이 있다.

Babel은 syntax 변환을 통해 javascript의 최신 버전을 지원하며, Babel의 플러그인은 브라우저의 지원 없이 바로 당장 새 syntax를 적용할 수 있게 도운다.

## JSX와 React에서 사용하기
Babel은 JSX환경에서 사용 가능하다. `Babel-sublime` 패키지를 추가로 설치하면 syntax highlighting이 가능하다고 하니 참고하자.
우선 윈도우 기준으로 Babel을 설치해보자. 
```
npm install --save-dev @babel/preset-react
```
그리고 기본 세팅을 위해 `@babel/preset-react` 를 추가해주자.
```javascript
export default React.createClass({
  getInitialState() {
    return { num: this.getRandomNumber() };
  },

  getRandomNumber() {
    return Math.ceil(Math.random() * 6);
  },

  render() {
    return (
      <div>
        Your dice roll:
        {this.state.num}
      </div>
    );
  },
});
```

## React에서 만난 Babel 오류
<code>Uncaught Error : only one instance of babel-polyfill is allowed</code>

한 페이지에서는 딱 하나의 babel-polyfill만 허용된다. 두 개의 babel-polyfill을 실행하게 되면 반복적인 전역 객체 수정을 막기 위해 위 오류를 띄운다고 한다.

babel-polyfill 소스를 보면 내부적으로 전역 변수 하나를 두어 babel-polyfill이 이미 실행되었는지 체크하고 이미 실행된 경우 위 오류를 띄우는 것을 볼 수 있다.
```javascript
if (global._babelPolyfill) {
  throw new Error("only one instance of babel-polyfill is allowed");
}
global._babelPolyfill = true;

...
```
해결방법은 오류 뜻 그대로 babel-polyfill을 두 개를 사용했으니, 하나를 찾아 제거해주면 된다.

출처 [Babel 공식문서][Babel-공식문서]

[Babel-공식문서]:https://babeljs.io/docs/en/