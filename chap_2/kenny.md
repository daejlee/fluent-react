자바스크립트 확장 구문(Javascript Syntax eXtension)에 대해 알아본다. 자바스크립트 XML이라고도 한다.

# 2.1 자바스크립트 XML

JSX는 JS 코드 내에서 HTML과 유사한 코드를 작성할 수 있게 하는 JS용 구문 확장자다. JSX 코드는 컴파일 후 JS 코드로 변환된다.

JSX를 이용했을 때와 그렇지 않았을 때의 차이는 크다.

|JSX|no JSX|
-|-
![alt text](image.png)|![alt text](image-1.png)

# 2.2 JSX의 장점

- 가독성
- 새 엘리먼트를 생성할 수 있는 문자가 HTML 문자열에 포함되어 있을 시 JSX 컴파일 과정에서 sanitize됨
- 타입 안정성
- 컴포넌트 기반
- 리액트 뿐만 아닌 범용성

# 2.3 JSX의 약점

- 러닝 커브 (러닝 커브가.. 있었나.. 처음 배울 때는 헷갈렸던 것 같기도 하다)
- 컴파일 도구 필요
    - Vue.js등의 프레임워크는 script 태그만 추가하면 브라우저에서 즉시 작동
- 관심사 혼합
- JS 호환성 부족
    - JSX 내부에서 인라인 표현식은 되지만 인라인 블록은 지원안함(switch, if 같이)

# 2.4 내부 동작

컴파일 과정을 알아본다. tokenization, parsing, code generation 단계로 나뉜다.

- 토큰화 - 문자열을 토큰으로 분해
- 구문 분석 - 토큰을 구문 트리로 변환
- 코드 생성 - 컴파일러가 AST(abstract syntax tree)에서 기계어를 생성

여러 컴파일러가 존재하는데, JS에서는 JIT(just in time) 컴파일러를 사용한다.

런타임은 엔진과 연동해 환경에 맞는 컨텍스트 헬퍼와 기능을 제공한다. (chrome 같은 웹 브라우저)

크롬의 경우 엔진과 연동하는 Chromium 런타임을 제공한다. 서버측은 V8 엔진을 사용하는 Node.js 런타임을 사용한다.

런타임은 JS 엔진에 브라우저 런타임이 제공하는 window, document 객체같은 컨텍스트를 제공한다. Node.js에 window 객체가 없는 이유도 런타임이 달라서 그렇다.

## 2.4.2 JSX로 자바스크립트 구문 확장하기

JSX를 동작하게 하려면 새 구문을 이해하는 엔진이 필요하거나, 구문을 처리해야 한다. JSX만을 위한 엔진을 만드는 것은 JS 엔진이 다양한 곳에서 사용된다는 점을 생각했을 때 비합리적이다.

JSX 문자열을 이해하는 렉서와 구문 분석기가 필요하다.

이를 Babel, TypeScript, Traceur, SWC같은 도구가 해결해준다.

![alt text](image-2.png)

그렇기에 JSX는 빌드가 필요하다. 코드가 바닐라 JS로 변환되어 최종 배포 번들에 포함되는 과정을 transpile이라고 하는데, 코드를 변환 후 컴파일 한다는 말이다.

# 2.5 JSX 프라그마

프라그마(Pragma): 컴파일러에 특정 작업을 수행하도록 지시하는 지시어

JSX에서는 ‘<’같은 JSX 프라그마가 함수 호출로 변환된다. (JS 엔진에서 ‘<’는 비교 연산에서만 쓰인다)

구문 분석기가 ‘<’ 프라그마를 만날 때 호출하는 함수는 React.createElement, 혹은 _jsxs가 기본값이다.

JSX 프라그마는 React.createElement 대신 ‘<’ 문자를 사용하는 별칭일 뿐이다.

![alt text](image-3.png)

# 2.6 표현식

엘리먼트 트리 내부에서 코드를 실행할 수 있다는 것이 JSX의 강력한 기능 중 하나다.

짚고 넘어갈 점은, JSX 표현식은 말 그대로 “표현식”이라는 것이다. 문장(statement)를 실행시키진 못한다.

```jsx
// O
const a = 1;
const b = 2;

const MyComponent = () => <Box>b가 a보다 큽니까? {b > a ? "예" : "아니오"}</Box>;

// X, 평가되지 않는다.
const MyComponent = () => <Box>표현식입니다: {
	const a = 1;
	const b = 2;
	
	if (a > b) {
		3
	}
}</Box/>;
```

# 2.8 복습하기

- JSX란 무엇이며, 장단점은?
    - 자바스크립트 연장 구문, HTML을 작성하듯이 JS 코드를 작성할 수 있어 가독성이 좋다(선언적이다) 하지만 트랜스파일링이 필요하고, JS 지원이 아쉬운 점이 있다(표현식은 되지만 문장 실행은 안됨)
- JSX와 HTML의 차이점은?
    - HTML은 마크업 언어이고, JSX는 JS 코드를 HTML처럼 작성할 수 있게 연장된 문법이라는 점에서 다르다.
- 텍스트 문자열은 어떻게 기계어가 되나?
    - 컴파일 과정을 거쳐 기계어가 된다. 토큰화, 구문 분석, 코드 생성 과정을 거친다.
- JSX 표현식이란 무엇이며 어떤 좋은 점이 있나요?
    - JSX 내부에서 표현식 실행이 가능한 특징을 뜻하며, 인라인에서 바로 표현식을 사용할 수 있다는 점이 코드 작성 시 편의성을 높여준다.

# 3.1 가상 DOM 소개

가상 DOM은 실제 DOM의 가벼운 복사본으로, JS 객체로 구성된다.

UI를 변경할 때 마다 가상 DOM을 먼저 업데이트하고, 실제 변경 사항에 맞춰 DOM을 업데이트한다.

그 이유는 실제 DOM의 업데이트가 느리고 비싸기 때문.

# 3.2 실제 DOM

DOM은 웹 페이지의 현 상태를 실시간으로 표현하여, 상호작용대로 계속 업데이트된다.

![image.png](attachment:267c5aa3-52a1-4b43-b34b-41782ef07b8c:image.png)

document.querySelector 메서드는 CSS 선택자를 기반으로 DOM에서 엘리먼트를 선택한다.
(DFS 알고리즘으로 엘리먼트를 탐색한다고 하네요)

크고 복잡한 문서의 경우 속도가 느려지는데, document.getElementById는 id 속성이 고유할 것으로 예상되어 더 효율적이다. (모던 브라우저에선 id-엘리먼트 1대1 매핑을 위해 해시 테이블도 활용한다네요)

다만 id가 고유할 것을 강제하지 않고, 고유하다 해도 한 페이지에서 재사용되는 컴포넌트에 적합치 않다.

이렇듯 실제 DOM 조작은 복잡하고, 위험하다. 가상 DOM으로 이런 위협으로부터 보호받을 수 있다.

## 3.2.1 실제 DOM의 문제점

- 성능
    - DOM을 변경할 때 마다 reflow, repaint가 일어난다.
    - offsetWidth 속성을 읽는 것 만으로도 레이아웃 계산이 들어간다.
    - 브라우저의 reflow를 최소화하고, 계산 시 한 번의 작업으로 필요한 정보를 확보하게끔 해야한다.
    - 가상 DOM을 실제 DOM 작업의 중간 계층으로 활용해 처리한다.

<aside>
	
	💡근데.. 이렇게 밀리초 아끼는게 실제 효과가 있나요?

	있다고 합니다.

	[밀리초 단위로 수익을 창출  |  web.dev]
	(https://web.dev/case-studies/milliseconds-make-millions?hl=ko)

</aside>

- 브라우저 간 호환성
    - 브라우저마다 DOM 모델링 방식이 달라 웹앱의 일관성이 보장되지 않고, 버그가 발생할 수 있다.
    - 리액트에서 SyntheticEvent라는 래퍼 솔루션을 사용해 해결.

## 3.2.2 문서 조각

문서 조각은 DOM 노드를 저장하는 컨테이너로, 가상 DOM의 전신이다. 임시 저장소처럼 동작해 업데이트가 완료되면 문서 조각을 DOM에 추가하여 reflow, repaint를 한 번만 발생시킨다.

```jsx
const fragment = document.createDocumentFragment();
for (let i = 0; i < 100; i++) {
	const li = document.createElement("li");
	li.textContent = `목록 항목 ${i + 1}`;
	fragment.appendChild(li);
}
document.getElementById("myList").appendChild(fragment);
```

가상 DOM은 이 문서 조각 개념을 더 나은 방식으로 구현했다.

- 문서 조각은 DOM을 업데이트하기 전, 변경 사항을 그룹화하여 최적화
    - 가상 DOM은 앱 UI 전반에 걸쳐 차이점을 파악, 일괄 업데이트로 최적화 극대화
    - 문서 조각 관련 복잡한 작업을 개발자가 모르게 내부적으로 처리

