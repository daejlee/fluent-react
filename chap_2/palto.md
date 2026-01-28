# JSX

자바스크립트 확장 구문(자바스크립트 XML)

- 별도의 언어가 아닌 컴파일러나 트랜스파일러에 의해 일반 자바스크립트 코드로 변환되는 확장 구문  
  (JSX -> \_jsxs, \_jsx function(React.createElement) -> JavaScript)
- 자바스크립트 코드 내에서 HTML과 유사한 코드 작성할 수 있는 자바스크립트용 구문 확장자

| html                                                | jsx                                                                                                                                            |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| 자바스크립트 표현식 삽입 불가                       | 중괄호로 자바스크립트 표현식 삽입 가능 <br> (인라인 블록(문장)은 값은 반환하지 않으므로 상태를 설정할 수 있지만, JSX에서는 표현식만 사용 가능) |
| 속성명은 소문자/하이픈 사용 (e.g., class, tabindex) | 속성명은 카멜케이스 사용 (e.g., className, tabIndex)                                                                                           |

## 장단점

장점

- **더 쉬운 읽기 및 쓰기**: HTML에 익숙하면 JSX문법을 이해하기 쉬움
- **향상된 보안 (데이터소독)**: HTML문자열 내 `<>` 문자를 컴파일 시 안전하게 변환 (`< -> &lt;`)
- **강력한 타이핑**: 오류를 사전에 잡아내고, 타입스크립트/propTypes/JSDoc으로 타입 안정성 확보 가능
- **컴포넌트 기반 아키텍처**: 코드 모듈화 및 유지보수 용이
- **광범위한 사용**: 리액트 외 다양한 라이브러리/프레임워크에서 지원

단점

- **학습 곡선 가중**: 기존 JS/HTML과 달라 추가 학습 필요
- **전용 도구 필요**: 바닐라JS로 컴파일 필요, 개발도구 필수 (vue.js는 `<script>` 태그만으로 동작)
- **관심사 혼합**: 표현과 논리의 분리 어려움, 관심사 혼합 우려
- **자바스크립트 호환성 부족**: 인라인 표현식 지원, 인라인 블록(switch, if 등) 미지원

## 내부동작

### 컴파일러

프로그래밍 언어로 적힌 텍스트파일(코드)을 컴퓨터가 해석하고 실행하기 위해선, 이런 텍스트 파일의 키워드들을 식별해야함.

컴파일러를 사용해 코드를 컴파일해야함.

- 컴파일러는 특정 규칙에 따라 고급 프로그래밍언어로 작성된 소스 코드를 구문트리로 변환하는 소프트웨어. (구문트리: 자바스크립트 객체 같은 트리 자료 구조)
- 토큰화 -> 구문분석 -> 코드 생성 과정을 거침

#### 1. 토큰화

- 문자열을 의미 있는 최소 단위인 토큰으로 분해하는 과정입
- 렉서(Lexer)와 토크나이저의 차이

  - 토크나이저: 단순히 문자열을 토큰으로 분리
  - 렉서: 상태를 유지하면서(이전에 읽은 토큰의 정보를 기억하면서) 토큰 간의 관계(부모-자식)를 파악

- 렉서의 동작 방식
  - 정규 표현식 기반의 렉서 규칙을 사용
  - 변수명, 키워드, 연산자 등을 감지하고 숫자로 매핑 (예: const → 0, let → 1, function → 2)

#### 2. 구문 분석

- 토큰을 가져와 구문트리로 변환하는 과정
- 구문 분석기로 인해 문자열은 JSON객체가 됨
- 언어 엔진이 이런 자료 구조를 사용해 코드 생성 과정을 실행

before

```
const a = 1;
let b = 2;
console.log(a + b);
```

after

```
{
  "type": "Program",
  "body": [
    {
      "type": "VariableDeclaration",
      "kind": "const",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "value": 1
          }
        }
      ]
    },
    {
      "type": "VariableDeclaration",
      "kind": "let",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "b"
          },
          "init": {
            "type": "Literal",
            "value": 2
          }
        }
      ]
    },
    {
      "type": "ExpressionStatement",
      "expression": {
        "type": "CallExpression",
        "callee": {
          "type": "MemberExpression",
          "object": {
            "type": "Identifier",
            "name": "console"
          },
          "property": {
            "type": "Identifier",
            "name": "log"
          }
        },
        "arguments": [
          {
            "type": "BinaryExpression",
            "operator": "+",
            "left": {
              "type": "Identifier",
              "name": "a"
            },
            "right": {
              "type": "Identifier",
              "name": "b"
            }
          }
        ]
      }
    }
  ]
}
```

#### 3. 코드 생성

- 컴파일러가 추상 구문 트리 AST에서 기계어를 생성하는 과정
- 결과로 생성된 기계어는 자바스크립트 엔진에 의해 실행됨

### 컴파일러 종류

- 네이티브 컴파일러
- 크로스 컴파일러
- JIT 컴파일러 (v8)
- 인터프리터

### 트랜스파일러

- JS 구문을 이해하고 실행하기 위해 JS 엔진이 존재  
  <-> 자바스크립트 구문을 확장하려면 새로운 구문을 이해하는 다른 엔진이 필요하거나, **엔진보다 앞서 새로운 구문을 처리해야함**  
  (전자보다 후자가 나음)
- JSX를 브라우저에서 직접 사용하지 않고, JS파일로 변환해주는 빌드단계가 필요 => **트랜스파일러**
  - JSX파일 -> JSX전처리기(토큰화 -> 구문분석 -> 변환) -> JS엔진(JS파일 -> 컴파일러 -> 실행)

1. Babel

- 특징: 가장 전통적이고 안정적인 JavaScript 트랜스파일러
- 사용처: Create React App (CRA), 레거시 프로젝트
- 장점: 플러그인 생태계가 풍부하고 안정적
- 단점: 상대적으로 느린 빌드 속도

2. TypeScript Compiler (tsc)

- 특징: TypeScript 공식 컴파일러, 타입 체크 + 트랜스파일 동시 수행
- 사용처: TypeScript 프로젝트 전반, Angular
- 장점: 강력한 타입 시스템, IDE 지원 우수
- 단점: 빌드 속도 느림, 번들링 기능 없어 Webpack 등 필요

3. SWC (Speedy Web Compiler)

- 특징: Rust로 작성된 초고속 트랜스파일러
- 사용처: Next.js 13+, Turbopack
- 장점: Babel 대비 20배 이상 빠른 속도, Babel 플러그인 대부분 호환
- 단점: 플러그인 생태계가 Babel보다 부족, 일부 edge case 버그

4. Vite

- 특징: esbuild 기반 차세대 빌드 도구 (번들러 + 트랜스파일러)
- 사용처: React, Vue, Svelte 등 모던 프레임워크
- 장점: 초고속 개발 서버, 간단한 설정, 뛰어난 DX (개발 경험)
- 단점: 프로덕션 빌드는 Rollup 사용으로 esbuild보다 느림

5. esbuild

- 특징: Go로 작성된 극한의 속도를 자랑하는 번들러 + 트랜스파일러
- 사용처: Vite 내부, Remix
- 장점: 압도적인 빌드 속도 (Webpack 대비 100배), 설정 간단
- 단점: 플러그인 생태계 제한적, 타입 체크 미지원

### JSX변환

변환 전

```
function App() {
  return (
    <div className="container">
      <h1>Hello World</h1>
      <p>Welcome to React</p>
    </div>
  );
}
```

변환 후

```
import { jsx as _jsx, jsxs as _jsxs } from "react/jsx-runtime";

function App() {
  return _jsxs("div", {
    className: "container",
    children: [
      _jsx("h1", { children: "Hello World" }),
      _jsx("p", { children: "Welcome to React" })
    ]
  });
}
```

- 보통 키워드, 연산자 등이 프라그마인 js 와 달리 jsx는 `<`가 프라그마

# 가상 DOM

## 실제 DOM의 문제점

- 성능
  - 크고 복잡한 웹 페이지를 처리할 때 성능 부담이 큼
  - 엘리먼트 추가나 제거, 텍스트, 속성 업데이트 등으로 DOM을 변경해야 할 때마다 브라우저는 리플로우함 -> 이 과정이 많아지고 복잡해질수록 리소스가 커짐
  - 성능저하는 곧 사용성 저하로 이어짐
- 브라우저 간 호환성
  - 브라우저마다 문서 모델링 방식이 달라 웹 애플리케이션의 일관성이 보장되지 않음
  - 특정 DOM 엘리먼트와 속성을 지원하지 않는 브라우저가 있을 수 있음
- 보안 이슈 (XSS)

## 가상 DOM

### 실제 DOM 문제점 보완

- 실제 DOM의 복잡성을 추상화 하고 UI를 더 가볍게 표현함으로써 효율적이고 성능이 뛰어난 UI제작 가능
- SyntheticEvent: 브라우저의 기본 이벤트를 둘러싼 래퍼 객체로 브라우저간 일관성을 보장하기위해 설계
  - 통합 인터페이스: 브라우저간 이벤트 처리를 하는 차이들을 추상화해 이벤트와 상호작용하는 일관된 방법 제공
  - 이벤트 위임: 이벤트 리스너를 엘리먼트에 직접 추가x -> 루트에서 이벤트를 받아 특정 브라우저에서 지원하지 않는 이벤트 문제를 해결
  - 다양한 기능 개선: 특정 이벤트를 처리하는 방식을 일관되게 변경
  - 네이티브 이벤트에 접근
- 문서조각: DOM 노드를 저장하는 가벼운 컨테이너로 기본 DOM에 영향을 주지 않고 여러가지 업데이트를 수행할 수 있는 임시 저장소처럼 동작
  - 일괄 업데이트: 과도한 리플로우 방지
  - 메모리 효율성: 문서 조각에 추가된 노드는 문서의 실제 DOM에서 제거하여 큰 영역을 재정렬할 때 메모리 사용량 최적화에 일조함
  - 중복렌더링 방지
  - 단일 렌더링
  - 효율적인 비교 알고리즘

### 리액트 엘리먼트

- 리액트 가상DOM 트리의 요소가 엘리먼트 형식으로 표현
- React.createElement(\_jsx) 함수 사용하여 생성
- 컴포넌트 형식

```
<div className="container" id="main">
  <h1>Hello World</h1>
</div>
```

- 엘리먼트 형식

```
{
  $$typeof: Symbol(react.element),
  type: 'div',
  key: null,
  ref: null,
  props: {
    className: 'container',
    id: 'main',
    children: {
      $$typeof: Symbol(react.element),
      type: 'h1',
      key: null,
      ref: null,
      props: {
        children: 'Hello World'
      },
      _owner: null,
      _store: {}
    }
  },
  _owner: null,
  _store: {}
}
```

#### 구성요소

- \$\$typeof: 객체가 유효한 리액트 엘리먼트인지 확인할 때 사용하는 특수한 심벌 (XSS 공격 방지용)
- type: 'div' 같은 HTML 태그 문자열 또는 컴포넌트 함수/클래스
- ref: DOM 노드나 컴포넌트 인스턴스 직접 접근용 객체
- props: 컴포넌트에 전달되는 데이터 (attributes + children 포함)
- key: 리스트 렌더링 시 각 엘리먼트를 구분하는 고유 식별자
- \_owner: (개발 모드) 이 엘리먼트를 생성한 부모 컴포넌트 정보
- \_store: 내부 검증 상태 저장용 객체, 내부적으로 엘리먼트 상태와 콘텍스트의 다양한 측면을 추적할 때 사용되며, 리액트 내부 구현 세부사항이기에 직접접근은 불가능

### 가상 DOM과 실제 DOM 비교

- 리액트의 가상 DOM은 실제 DOM과 개념이 유사함.
- 리액트 컴포넌트 렌더링 -> 새 가상 DOM 트리 생성 -> 기존 가상 DOM트리와 비교(diffing algorithm) -> 이전 트리를 새 트리와 일치하도록 업데이트하는 데 필요한 최소 변경 횟수를 계산함(reconciliation) -> 기존 트리에 업데이트하며 실제 DOM 업데이트

#### diffing algorithm

- 두 트리의 루트에 있는 노드가 다르면, 기존 트리 전체를 새 트리로 변경
- 루트 노드가 동일하면 비교를 재귀적으로 하며 노드 속성이 변경된 경우에만 업데이트
- 자식 노드가 다른 경우 변경된 자식 노드만 업데이트
- 노드 자식들이 동일하지만 순서가 변경된 경우, 실제 dom에서 노드의 순서를 재설정
- 노드가 제거되면 실제 DOM에서도 제거, 추가되면 추가
- 노드의 종류(type)이 변경되면 이전 노드를 제거하고, 변경된 종류의 새 노드를 생성
- 노드에 key가 있다면 이를 비교하여 노드 교체가 필요한지 파악.
