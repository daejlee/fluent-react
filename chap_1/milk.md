# 입문자를 위한 지식
리액트로 지금은 함수형 프로그래밍을 통해 훅을 사용하고, 간단하게 외부 라이브러리로 컴포넌트를 불러와 사용하기도 합니다. 하지만 이게 어떻게 나왔을지에 대해 궁금해 태초의 시대로 돌아가보자 합니다.

### 바닐라 자바스크립트
과거에는 html 로 직접 `<button id="likeButton" />`을 만들어서  `document.getElementById('likeButton')`라는 자바스크립트 문법을 사용해 직접 찾아 이를 컨트롤 하였습니다.

그리고 fetch API도 없었던 시절이어서 `XMLHttpRequest`를 사용하여 네트워크 통신을 했습니다.

> ### XMLHttpRequest vs fetch API 의 차이
> XMLHttpRequest는 콜백으로만 사용해서 async await를 지원하지 않습니다. 비동기 프로그래밍이 불가능하며, json 파싱이 지원안하기도 하고, xhr.readyState === 4 같은 매직넘버(의미를 바로 알 수 없는 숫자)가 있어 공부 난이도가 더 높다고 합니다.

### jQuery
jQuery가 등장한 후 보다 쉽게 사용자 인터페이스를 직접 조작할 수 있게 되었습니다. 그러나 번들크기가 크고, $선택자가 `document.querySelector`로 자바스크립트 내부에서 기본적으로 지원하며, 성능 저하가 이슈가 있어 대체되었습니다. 또한 테스트도 하기 어렵다는 단점이 있었습니다.

### Backbone
MVC 아키텍처를 따르는 구조입니다. MVC를 우리 리액트 개발자가 알기 쉽게 비유하자면 다음과 같습니다.

- Model: 백엔드로 요청할 객체, 폼 데이터
- View: 눈에 보여지는 컴포넌트
- Controller: 인풋, 라디오 버튼 등 조작하면 변수값을 변경해줘야하는데 이런 로직들을 담은 코드

```javascript
// Model - 데이터와 비즈니스 로직
const Todo = Backbone.Model.extend({
  defaults: {
    title: '',
    completed: false
  }
});

// View - UI 렌더링과 이벤트 처리
const TodoView = Backbone.View.extend({
  tagName: 'li',
  template: _.template('<%= title %>'),
  
  events: {
    'click': 'toggleCompleted'  // 이벤트 -> Controller 역할
  },
  
  render: function() {
    this.$el.html(this.template(this.model.toJSON()));
    return this;
  },
  
  toggleCompleted: function() {
    this.model.set('completed', !this.model.get('completed'));
  }
});

// 사용
const todo = new Todo({ title: '리액트 공부하기' });
const view = new TodoView({ model: todo });
$('#app').append(view.render().el);
```

하지만 이 Controller라는건 아주 방대합니다. 이벤트 처리, 훅, 유틸, 눈에 보이지 않는 백그라운드 상의 비즈니스 로직 등등 분류가 너무 많으며, 또 컨트롤러끼리의 도메인 경계를 나누기 어렵습니다.

Backbone은 보일러플레이트코드가 많고, 양방향 데이터 바인딩이 부족해 데이터의 변화를 감지하는 로직을 직접 구현해야했습니다.

그리고 뷰단에 이벤트들을 발행하는데 이런 수 많은 이벤트들을 중앙에서 관리할 수 없어 영향도 파악이나 유지 보수가 어려웠습니다. 지금 당장 위의 예시코드만 봐도 그렇게 느껴질겁니다.

### Knockout
옵저버블과 바인딩을 작성하는 방법을 제공하는 라이브러리였습니다. 쉽게 말해 데이터 감지를 도와주는 라이브러리입니다. MVVM 아키텍처를 따르는 구조입니다. MVVM을 우리 리액트 개발자가 알기 쉽게 비유하자면 다음과 같습니다.

- Model: 백엔드로 요청할 객체, 폼 데이터
- View: HTML 템플릿 (눈에 보여지는 컴포넌트)
- ViewModel: 상태 관리와 데이터 바인딩 로직 (리액트의 useState, 커스텀 훅 같은 역할)

```javascript
// ViewModel - 상태 관리와 로직
function TodoViewModel() {
  this.title = ko.observable('리액트 공부하기');
  this.completed = ko.observable(false);
  
  // computed - 계산된 값 (리액트의 useMemo와 유사)
  this.displayText = ko.computed(() => {
    return this.completed() ? '✓ ' + this.title() : this.title();
  });
  
  // 액션
  this.toggleCompleted = () => {
    this.completed(!this.completed());
  };
}

// HTML View에 바인딩
// <li data-bind="text: displayText, click: toggleCompleted"></li>

// 사용
ko.applyBindings(new TodoViewModel());
```

하지만 HTML에 `data-bind` 속성으로 로직을 작성해야 해서 View와 로직이 분리되지 않았습니다. 또한 옵저버블 문법(`title()`, `completed()`)이 복잡하고, 중첩된 데이터 구조를 다루기 어려웠습니다.

### Angular.js
구글이 만든 본격적인 프레임워크입니다. MVC 패턴을 사용하며, 양방향 데이터 바인딩, 모듈식 아키텍처, 의존성 주입 등 현대적인 개념들을 도입했습니다.

**주요 특징:**
- **양방향 데이터 바인딩**: View와 Model이 자동으로 동기화됨 (리액트의 useState처럼 값이 바뀌면 화면도 자동으로 업데이트)
- **모듈식 아키텍처**: 기능별로 모듈을 나눠서 관리
- **의존성 주입(DI)**: 필요한 서비스를 자동으로 주입 (리액트의 Context API와 비슷한 개념)

```javascript
// Controller - 로직과 상태 관리
angular.module('todoApp', [])
  .controller('TodoController', function($scope, TodoService) {
    // 의존성 주입: TodoService가 자동으로 주입됨
    $scope.title = '리액트 공부하기';
    $scope.completed = false;
    
    // 양방향 바인딩: $scope 값이 변하면 View도 자동 업데이트
    $scope.toggleCompleted = function() {
      $scope.completed = !$scope.completed;
    };
    
    $scope.saveTodo = function() {
      TodoService.save({ title: $scope.title });
    };
  })
  .service('TodoService', function($http) {
    // 서비스 - 재사용 가능한 비즈니스 로직
    this.save = function(todo) {
      return $http.post('/api/todos', todo);
    };
  });
```

```html
<!-- HTML View -->
<div ng-controller="TodoController">
  <input ng-model="title" />  <!-- 양방향 바인딩 -->
  <button ng-click="toggleCompleted()">토글</button>
  <p>{{ completed ? '완료' : '미완료' }}</p>
</div>
```

코드를 보면 무슨 프론트엔드가 백엔드도아니고 controller가 servcice를 가지고 있습니다. 스프링인가요? 여기서 의존성주입을 자동으로 하준다고 표현했습니다.

그런데 의존성 주입에 모듈식 아키텍처. 뭔가 있어보이지만 실제 위 코드를 보면 어지럽습니다. 러닝 커브가 높습니다.

또한 **$scope의 양방향 바인딩이 성능 문제**를 일으켰습니다. 모든 변경사항을 감지하기 위해 Dirty Checking을 사용하는데, 데이터가 많아지면 느려졌습니다. 그리고 프레임워크가 너무 무거웠습니다.

### React.js
페이스북이 만든 라이브러리로, 앞선 프레임워크들의 문제를 해결하기 위해 새로운 접근 방식을 도입했습니다.

**주요 특징:**
- **함수형 프로그래밍**: 순수 함수로 컴포넌트를 작성하여 예측 가능하고 테스트하기 쉬운 코드
- **함수 컴포넌트 패턴**: 클래스가 아닌 함수로 UI를 정의 (간결하고 이해하기 쉬움)
- **Hooks**: useState, useEffect 등으로 상태와 생명주기를 함수 내에서 관리
- **Flux 아키텍처**: 단방향 데이터 흐름으로 상태 변화를 예측 가능하게 만듦 (Redux, Zustand 등)

```javascript
// 함수 컴포넌트 - 순수 함수처럼 작성
function TodoItem({ initialTitle }) {
  // Hooks - 함수 내에서 상태 관리
  const [title, setTitle] = useState(initialTitle);
  const [completed, setCompleted] = useState(false);
  
  // 부수 효과 관리
  useEffect(() => {
    console.log('Todo 상태 변경:', completed);
  }, [completed]);
  
  const toggleCompleted = () => {
    setCompleted(!completed);  // 단방향: 상태 변경 -> 리렌더링
  };
  
  return (
    <div>
      <input 
        value={title} 
        onChange={(e) => setTitle(e.target.value)}  // 제어된 컴포넌트
      />
      <button onClick={toggleCompleted}>토글</button>
      <p>{completed ? '완료' : '미완료'}</p>
    </div>
  );
}

// Flux 패턴 (Redux 예시)
// Action -> Dispatcher -> Store -> View (단방향 흐름)
const todoSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      state.push(action.payload);  // 불변성 유지
    }
  }
});
```

리액트의 장점을 이렇게 볼 수도 있습니다.
- **단방향 데이터 흐름**으로 디버깅과 상태 추적이 쉬움 (Angular의 양방향 바인딩 문제 해결)
- **Virtual DOM**으로 효율적인 렌더링 (변경된 부분만 업데이트)
- **컴포넌트 재사용성**이 높고 생태계가 풍부함
- 함수형 프로그래밍으로 **테스트와 유지보수가 용이함**

하지만 억까도 필요합니다. 개인적으로 이렇게 생각합니다.
- **러닝 커브가 높음**: 우리는 이미 리액트를 알아서 쉽다고 생각했지만, 프론트모르는 사람들 보면 Vue.js, Angular.js를 오히려 더 잘씁니다. JSX, 불변성, 함수형 프로그래밍 개념이 초보자에게는 어렵습니다.
- **너무 많은 선택지**: 상태관리(zustand, redux, recoil), 라우팅(tanstack route, react route), 스타일링방식(tailwind, scss, styled-components) 등 직접 결정해야하는게 너무 많음. Angular는 올인원인데 React는 모든 걸 선택해야 합니다.
- **SSR 복잡도**: 순수 React로 SSR을 직접 구현하는 건 가능하지만 매우 복잡합니다 (라우팅, 번들링, 하이드레이션 등). 실무에서는 Next.js나 Remix 같은 프레임워크가 필수적입니다.
- **Server Components와 벤더락인**: 최근 나온 서버 컴포넌트는 Next.js와 강하게 결합되어 있습니다. 다른 대안(Remix 등)도 있지만, Next.js가 사실상 표준이라 선택의 폭이 좁습니다.

## 결론
각 프레임워크는 당시의 문제를 해결하기 위해 등장했습니다. jQuery는 DOM 조작을, Backbone은 구조화를, Angular는 완전한 프레임워크를, 그리고 React는 단방향 데이터 흐름과 컴포넌트 재사용성을 제공했습니다.

**현재 React를 선택하는 이유:**
- 한국에서 가장 큰 생태계와 커뮤니티
- 활발한 메인테넌스와 지속적인 업데이트
- 풍부한 라이브러리와 학습 자료
- 함수형 프로그래밍과 단방향 데이터 흐름의 명확성

하지만 **프로젝트 요구사항에 따라** Vue.js(빠른 학습), Svelte(번들 크기), Angular(대규모 엔터프라이즈)가 더 나을 수 있습니다.

이 역사를 이해하면 React가 왜 이렇게 설계되었는지, 그리고 앞으로 어떤 방향으로 발전할지 예측할 수 있습니다. **맹목적인 선택보다는, 왜 이 기술을 사용하는지 이해하는 것이 더 중요합니다.**