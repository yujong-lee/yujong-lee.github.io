---
title: "React 컴포넌트 렌더링 이해하기"
category: dev
---

**(작성 중)**

프로젝트의 크기가 작을 때는 렌더링이 어떻게 되는지 신경 쓸 일이 거의 없다. 그런데 크기가 커지기 시작하면 문제가 발생한다. 첫째, 리렌더링이 도대체 어떻게 일어나는지 헷갈리고 앱의 동작을 완벽하게 이해하지 못한다고 느끼게 된다. 둘째, 일부 상황에서 성능 이슈가 발생할 수 있다.

# 렌더링이란 무엇인가

리액트는 다음의 과정을 거쳐 DOM을 업데이트한다.

1. render 단계: React.createElement로 생성
2. reconciliation 단계: 이전 elements와 새로 생성된 elements 비교
3. commit 단계: DOM update (필요하다면)

# 어떻게 최적화하는가

보통 최적화라고 하면 re-rendering, 즉 render단계에 들어가는 횟수를 줄여야 한다고만 생각하기 쉽다. 거기에는 크게 두 가지 이유가 있다. 첫째, re-rendering == DOM update 라고 생각한다. 둘째, DOM update는 언제나 느리다고 생각한다.

하지만 이는 사실이 아니다. 먼저 re-render과 DOM update는 같지 않다. re-render은 되지만 DOM은 update되지 않을 수 있다. React는 변화를 한번에 처리하며, recomciliation 단계에서 DOM update가 필요한지 판단해 변화가 있을 때만 업데이트한다.

So the React team decided to batch DOM updates, so if there was a state change that resulted in thirty DOM updates, they would all happen at once, rather than running them one after another. - Kent C. Dodds

게다가 DOM update가 '반드시' 느린 것은 아니다.

But not all DOM updates are slow. In fact, it's probably a bit misleading to state simply that "the DOM is slow" because it's more nuanced than that. DOM updates like adding/removing event listeners are really fast. The slow part of the DOM is "layout" - Kent C. Dodds

따라서 React 최적화는 크게 두 가지로 나누어서 생각할 수 있다.

1. 각 render에 드는 비용 줄이기
2. render을 하는 횟수 줄이기

1번은 브라우저나 React에서 제공하는 프로파일링 도구를 이용해 개선할 수 있다. 프로파일링 도구는 각 render에 드는 비용이 얼마나 되는지 알려준다. 다음 링크에서 영상을 참고하면 좋다. [ChromeDevTools 영상](https://twitter.com/kentcdodds/status/1171158009277403136?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1171158009277403136%7Ctwgr%5E%7Ctwcon%5Es1_&ref_url=https%3A%2F%2Fkentcdodds.com%2Fblog%2Ffix-the-slow-render-before-you-fix-the-re-render), [React DevTools 영상](https://twitter.com/brian_d_vaughn/status/1126950967201546240?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1126950967201546240%7Ctwgr%5E%7Ctwcon%5Es1_&ref_url=https%3A%2F%2Fkentcdodds.com%2Fblog%2Ffix-the-slow-render-before-you-fix-the-re-render)

2번은 조금 복잡하다. 이것을 위해서는 React가 언제 re-render을 하는지 이해할 필요가 있다.

# 언제 re-render 되는가

React Component는 다음 네 가지 상황에서 re-render 된다. (forceUpdate() 제외)

1. Props가 변경되었을 때
2. State가 변경되었을 때
3. Context value가 변경되었을 때 (해당 컴포넌트가 listen하는)
4. 부모 컴포넌트가 re-render되었을 때

따라서 re-render를 줄이려면 Props, State, 또는 Context value의 변경을 줄이거나, 부모 컴포넌트의 re-render을 줄이면 된다. 대부분의 최적화는 Props의 변경을 줄이는 1번을 타겟팅한다. 불변객체(immutable.js 등)을 사용하거나, useMemo, useCallback을 사용하는 식이다. 하지만 그것보다 먼저 고려해야하는 것이 바로 4번, 부모의 컴포넌트의 re-render을 줄이는 것이다.

# memo하지 않고 최적화 하기

보통 1, 2, 3은 하나씩 발생되는 것이 아니라 연쇄적으로 발생된다. 예를 들어 부모의 State가 변경되어 부모 컴포넌트가 re-render되고, 이에 따라 자식 컴포넌트가 re-render되는 식이다. 아래 예시가 딱 그렇다. 버튼을 클릭할 때마다 State가 변하고, App이 re-render되며, 이에 따라 VerySlow도 re-render된다. 따라서 버튼을 클릭할 때마다 200ms의 딜레이가 발생하게 된다.

~~~jsx
export default function App() {
  const [count, setCount] = useState(20);

  return (
    <div>
      <h2>{count}</h2>
      <button 
        type="button" 
        onClick={() => setCount((c) => c + 1)}
      >
        +
      </button>
      <VerySlow />
    </div>
  );
}

const VerySlow = () => {
  const now = performance.now();
  while (performance.now() - now < 200) {
      // 200ms slow down
  }
  return <p>Slow</p>;
}
~~~

React.memo()를 이용하면 쉽게 해결할 수 있다. React.memo()는 Hook이 등장하기 전에 많이 쓰이던 Higher-Order Components인데, 이를 통해 반환된 컴포넌트는 같은 props에 대해서 같은 결과를 바로 반환한다. VerySlow는 props를 받지 않으므로 언제나 <p>Slow</p>를 곧바로 리턴하게된다. (memo와 useMemo의 차이는 [이 글](https://sustainable-dev.tistory.com/137)을 참고하면 좋다.)

~~~jsx
const VerySlow = memo(() => {
  const now = performance.now();
  while (performance.now() - now < 200) {
      // 200ms slow down
  }
  return <p>Slow</p>;
})
~~~

하지만 이것이 올바른 해법일까? 어찌보면 이것은 너무 성급한 최적화라는 생각이 든다. 지금 App 컴포넌트는 너무 못생겼다. Count를 사용하는 부분을 따로 분리해주면 훨씬 좋을 것 같다.

~~~jsx
// App.js
export default function App() {
  return (
    <div>
      <Counter />
      <VerySlow />
    </div>
  );
}

const VerySlow = () => {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return <p>Slow</p>;
};

// Counter.js
export default function Counter() {
  const [count, setCount] = useState(20);

  return (
    <>
      <h2>{count}</h2>
      <button type="button" onClick={() => setCount((c) => c + 1)}>
        +
      </button>
    </>
  );
}
~~~

Counter컴포넌트를 분리하고 나니 딜레이가 더이상 발생하지 않는다. Counter컴포넌트는 VerySlow의 부모 컴포넌트가 아니다. Counter의 State변경은 Counter의 re-render만 일으킨다.

하지만 div에 count를 이용해 style을 적용해야 한다면 어떨까?  다음과 같이 말이다.

~~~js
// App.js
const fontStyle = { fontSize: ${count}px };
<div style={fontStyle}>
~~~

이러면 count는 VerySlow의 부모 컴포넌트인 App에 존재할 수 밖에 없다. memo()를 사용하는 것 외에는 방법이 없을까?
그렇지 않다. children props를 이용해서 컴포넌트의 재사용성을 가져가면서 해결할 수 있다.

먼저 다음과 같이 CounterWrapper을 만든다.

~~~jsx
// CounterWrapper.js
export default function CounterWrapper({ children }) {
  const [count, setCount] = useState(20);

  const fontStyle = { fontSize: ${count}px };

  return (
    <div style={fontStyle}>
      <h2>{count}</h2>
      <button type="button" onClick={() => setCount((c) => c + 1)}>
        +
      </button>

      {children}
    </div>
  );
}
~~~

그리고 App에서 CounterWrapper의 children으로 VerySlow를 사용한다.

~~~jsx
// App.js
export default function App() {
  return (
    <CounterWrapper>
      <VerySlow />
    </CounterWrapper>
  );
}
~~~

count가 변경되면 CounterWrapper는 re-render된다. 하지만 이때 이전에 App에서 얻은 것과 동일한 children을 가지고 있으므로 React는 해당 하위 트리를 방문하지 않는다. 따라서 CounterWrapper는 re-render되지만, VerySlow는 re-render되지 않는다.

[React Hook이 Container 컴포넌트를 대체하지 않는 것처럼](https://yujonglee.com/wiki/socwithhooks/), memo는 props.chidren을 이용하는 이 패턴을 대체하지 않는다. 이 두가지 접근은 상호보완적이다. 특히 props.chidren을 이용하는 패턴은 2020년 말에 발표된 [Zero-Bundle-Size React Server Components](https://reactjs.org/blog/2020/12/21/data-fetching-with-react-server-components.html)가 준비되면 더 쓸모가 많을 것이다.

# 참고

1. [Before You memo()](https://overreacted.io/before-you-memo/)
2. [A Foot In Each Realm: Zero-Bundle-Size React Server Components](https://mark-okeeffe-11887.medium.com/a-foot-in-each-realm-zero-bundle-size-react-server-components-5d84a7afad92)
3. [Fix the slow render before you fix the re-render](https://kentcdodds.com/blog/fix-the-slow-render-before-you-fix-the-re-render)
4. [Optimizing Performance](https://reactjs.org/docs/optimizing-performance.html)
5. [Composition vs Inheritance](https://reactjs.org/docs/composition-vs-inheritance.html)
