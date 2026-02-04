# 4.1 재조정 이해하기

가상 DOM을 재조정을 통해 호스트 환경에서 실체화한다.

리액트는 최소한의 DOM API만을 호출해 브라우저에 가상 DOM을 반영한다.

어떻게 최소화하나? 한 번의 실제 DOM 업데이트로 가상 DOM의 변경 사항을 일괄 처리한다.

# 4.2 일괄 처리

문서 조각과 유사하게, 여러 가상 DOM 업데이트를 모아 한 번의 DOM 업데이트로 결합해 실제 DOM에 대한 업데이트를 일괄 처리한다.

![image.png](attachment:fa38cf15-7072-485e-a73d-b6eed3778dca:image.png)

실제 DOM이라면 3번 바뀌어야 하지만, 가상 DOM은 3번의 변화를 한 번으로 끝낼 수 있다.

# 4.3 기존 기술

기존 리액트는 렌더링에 스택 데이터 구조를 사용했다.

이름에서 알 수 있듯이 스택 재조정자는 작업을 순차적으로 변경 사항을 렌더링한다. 비싼 연산이 렌더링을 막아버리는 병목 현상이 발생할 수 있다. 우선순위를 지정할 수가 없다.

또한 스택 재조정자는 업데이트를 중단, 취소할 수가 없다.

이런 문제를 해결하기 위해 리액트에서 파이버 재조정자를 개발했다.

# 4.4 파이버 재조정자

조정자를 위한 작업 단위 파이버가 등장한다. “특정 시점에 존재하는 실제 컴포넌트 트리를 나타내는 리액트 내부 데이터 구조”라고 설명한다.

- 작업을 작은 단위로 분할해 우선순위를 설정
- 작업들은 일시 정지 후 나중에 시작 가능
- 이전 작업을 재사용할 수 있으며 필요 없을시 폐기 가능

파이버의 위 작업들은 비동기적으로 발생한다.

렌더링 단계는 렌더 단계와 커밋 단계로 나뉜다.

1. 렌더 단계에서 사용자에게 노출되지 않는 비동기 작업 수행 - 파이버의 작업(우선순위 지정 및 중지, 폐기)
2. 커밋 단계에서 DOM에 실제 변경 사항을 반영하기 위한 작업 수행 - `commitWork()` 실행(동기적, 중단 X)

```jsx
<ul>
	<l1>one</li>
	<l1>two</li>
	<l1>three</li>
</ul>

const l3 = {
	return: ul,
	index: 2,
}

const l2 = {
	sibling: l3,
	return: ul,
	index: 1,
}

const l1 = {
	sibling: l2,
	return: ul,
	index: 0,
}

const ul = {
	// ...
	child: l1,
}
```

## 파이버 트리

파이버 트리에 대해 알아보자.

리액트 내부에 총 두 개가 존재하는데, 하나는 현 모습을 담은 것이고, 다른 하나는 작업 중인 상태를 나타내는 `workInProgress` 트리다.

파이버 작업이 끝나면 리액트는 포인터만 변경해 `workInProgess` 트리를 현 트리로 바꾼다.

이 기술을 더블 버퍼링이라고 한다. 이 더블 버퍼링은 커밋 단계에서 수행된다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/907641dc-2bf8-4fd3-8c6a-84e5873017ac/93b84933-3dc3-47ed-84ab-dcc065048ef7/image.png)

현재 UI 렌더링을 위한 트리인 `current`를 기준으로 업데이트가 발생하면 파이버는 리액트에서 새로 받은 데이터로 새로운 `workInProgress` 트리를 빌드하고, 작업이 끝나면 다음 렌더링에 이 트리를 사용한다. 최종적으로 렌더링되어 반영이 끝나면 `current`가 `workInProgress`로 변경된다.

## 파이버 트리와 파이버

1. 리액트는 `beginWork()` 함수를 호출해 파이버 작업을 수행한다. 자식이 없는 파이버를 만날 때까지 트리 형식으로 시작된다.
2. 끝나면 `completeWork()` 함수를 실행해 파이버 작업을 완료한다.
3. 형제가 있다면 형제로 넘어간다.
4. 위 작업이 모두 끝나면 `return`으로 돌아가 작업 완료를 알린다.

```jsx
<A1>
  <B1>Hi</B1>
  <B2>
    <C1>
      <D1 />
      <D2 />
    </C1>
  </B2>
  <B3 />
</A1>
```

1. A1의 `beginWork`
2. B1 `beginWork`
3. B1 `completeWork`
4. B2 `beginWork`
5. C1 `beginWork`
6. D1 `beginWork`, `completeWork`
7. D2 `beginWork`, `completeWork`
8. C1 `completeWork`
9. B2 `completeWork`
10. B3 `beginWork`, `completeWork`
11. A1 `completeWork`
12. `commitWork`
