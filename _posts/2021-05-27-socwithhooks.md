---
title: "React Hook VS Container Component"
category: dev
---

# 관심사의 분리

관심사의 분리, 영어로는 Separation of concerns라고 하는 주제는 리액트에서 꽤나 중요한 주제이다. 이를 위해 리액트에서 널리 사용되는 방법은 Presentational/Container 컴포넌트를 도입하는 것이다. Presentational 컴포넌트는 UI렌더링만 담당하고 그 외 상태관리 등의 로직은 Container 컴포넌트가 담당하게 된다.

이에 관해서는 Dan Abramov가 [2015년에 쓴 글](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)이 유명하다. 그런데 글의 첫 부분을 읽어보면 다음과 같은 문단이 추가되어있음을 확인할 수 있다.

>Update from 2019: I wrote this article a long time ago and my views have since evolved. In particular, I don’t suggest splitting your components like this anymore. If you find it natural in your codebase, this pattern can be handy. But I’ve seen it enforced without any necessity and with almost dogmatic fervor far too many times. The main reason I found it useful was because it let me separate complex stateful logic from other aspects of the component. Hooks let me do the same thing without an arbitrary division. This text is left intact for historical reasons but don’t take it too seriously.

컴포넌트를 Presentational/Container로 나누는 것이 자연스러울 때는 나누는게 좋겠지만 그렇지 않다면 굳이 그럴 필요가 없으며,  React Hook이 그 역할을 대신할 수 있다는 내용이다. 그렇다면 React Hook은 Presentational/Container 컴포넌트를 대체할 수 있을까? 그렇지 않다면 언제 전자를 사용하고 언제 후자를 사용해야할까?

내 생각에 그 둘은 서로를 대체하지 않으며, 프로젝트 구조, 네이밍, 테스팅 등 여러가지 면에서 차이가 있다. 그 중 내가 가장 중요하다고 느낀 것은 바로 재사용성에서의 차이이다. React Hook와 Presentational/Container 컴포넌트는 모두 관심사의 분리를 돕는다. 그 과정에서 React Hook의 도입은 로직을 재사용 가능하게 하고, Presentational/Container 컴포넌트의 도입은 컴포넌트를 재사용 가능하게 한다. 이 둘은 따로 사용될 수도 있고, 함께 사용될 수도 있다.

# 재사용성(Reusability)의 차이

## Presentational/Container 컴포넌트

뭔가 복잡한 로직을 거쳐서 얻은 tasks 배열을 버튼과 함께 렌더하는 List컴포넌트는 Presentational/Container페턴을 이용하여 아래와 같이 구현될 수 있다.

~~~jsx
// ListContainer.jsx
import { useSelector, useDispatch } from 'react-redux';

import List from '../presentational/List';

export default function ListContainer() {
  const tasks = useSelector(
      // complexLogic
  );

  const processedTasks = // complexProcess on tasks

  const handleClick = () => {
      // complexLogic using dispatch
      // used to delete task
  }

  return (
    <List 
      items={processedTasks}
      onClick={handleClick}
    />
  );
}

// List.jsx
export default function List({ items, onClick }) {
  return (
    <ol>
      {items.map(({ id, title }) => (
        <li key={id}>
          {title}
        </li>
      ))}
      ,
      <button type="button" onClick={onClick}>
        클릭
      </button>
    </ol>
  );
}
~~~

좋다. ListContainer는 필요한 정보, 함수등을 List에 넘겨 사용할 뿐이고, List는 받은 것을 이용해서 UI를 렌더링할 뿐이다. 중요한 것은 사실 List컴포넌트는 삭제 가능한 할일 목록을 렌더링하는데만 사용할 수 있는 것은 아니라는 점이다. List컴포넌트는 어디에서나 재사용될 수 있다. List는 id와 title을 가진 객체들의 배열, 그리고 onClick에 사용될 함수 하나만 받으면 된다. 그 내용이 무엇인지 List컴포넌트는 관심을 가지지 않는다.

## React Custom Hook

이번에는 같은 일을 하는 컴포넌트를 2개로 분리하지 않고 훅을 이용해 작성해보겠다.

~~~jsx
// useTasks.jsx
import { useSelector, useDispatch } from 'react-redux';

export default function useTasks() {
  const data = useSelector(
      // complexLogic
  );

  const tasks = // complexProcess on data

  const deleteTask = (id) => {
      // complexLogic using dispatch
      // used to delete task
  }

  return { tasks, deleteTask };
}

// List.jsx
import useTasks from './useTasks';

export default function List() {
  const { tasks, deleteTask } = useTasks();

  return (
    <ol>
      {tasks.map(({ id, title }) => (
        <li key={id}>
          {title}
        </li>
      ))}
      ,
      <button type="button" onClick={deleteTask}>
        클릭
      </button>
    </ol>
  );
}
~~~

이것도 좋다. List컴포넌트는 자신이 어떤 목록을 어떤 일을 하는 버튼과 렌더할지에 대한 구현은 포함하지 않는다. 그것은 useTasks훅에 작성되어있다. List컴포넌트가 하는 일은 어떤 훅 하나를 이용해 필요한 배열과 함수를 제공받아 렌더하는 것 뿐이다. useTasks훅이 어떻게 그 정보를 제공하는지 List컴포넌트는 관심을 가지지 않는다.

## 차이

ListContainer는 tasks와 handleClick이 어떻게 만들어졌는지 알고 있지만, **그것을 재사용할 수는 없다.** 하지만 useTasks훅은 호출되면 { tasks, deleteTask } 를 반환하는 명확한 인터페이스를 가지고 있으며, **재사용 가능하고**, 독립적으로 테스트될 수 있다.

ListContainer가 사용하는 **List컴포넌트는 어디든지 사용될 수 있다.** 그 내용이 무엇이든 props로 넘기면 된다. 하지만 내부적으로 useTasks훅을 사용하는 List컴포넌트는 **재사용되기 어렵다.** 이것은 삭제 가능한 할일 목록을 렌더하는 용도 외에는 쓰이지 못한다.

## 통합

중요한 것은 우리가 장점만을 취할 수 있다는 점이다. 아래의 예시는 Container 컴포넌트에 React Custom Hook를 도입한 것이다.

~~~jsx
// useTasks.jsx
import { useSelector, useDispatch } from 'react-redux';

export default function useTasks() {
  // 생략
  return { tasks, deleteTask };
}

// ListContainer.jsx
import List from '../presentational/List';
import useTasks from './useTasks';

export default function ListContainer() {
  const { tasks, deleteTask } = useTasks();

  return (
    <List 
      items={tasks}
      onClick={deleteTask}
    />
  );
}

// List.jsx
export default function List({ items, onClick }) {
  return (
    <ol>
      {items.map(({ id, title }) => (
        <li key={id}>
          {title}
        </li>
      ))}
      ,
      <button type="button" onClick={onClick}>
        클릭
      </button>
    </ol>
  );
}
~~~

이렇게 되면 우리는 모든 장점을 얻는다. 관심사의 분리, 재사용 가능한 useTasks흑과 List컴포넌트가 바로 그것이다.

# 결론

이제 나는 Dan Abramov의 글을 꽤 잘 이해하게 되었다고 생각한다.

>I don’t suggest splitting your components like this anymore. If you find it natural in your codebase, this pattern can be handy.

-> Presentational/Container 패턴은 필요할 때, 즉 컴포넌트의 재사용성이 필요할 때 사용하면 된다.

>The main reason I found it useful was because it let me separate complex stateful logic from other aspects of the component. Hooks let me do the same thing without an arbitrary division.

-> 훅이 생기기 이전에는, complex stateful logic을 컴포넌트에서 분리하기 위해서는 Presentational/Container 패턴을 도입해야했다. 그 상황에서는 그것이 부자연스러운 방법이라고 해도 말이다.

**하지만 이제는 그럴 필요가 없다. 자신이 무엇을 재사용하고 싶은지에 따라서 선택하면 된다. 심지어는 두개 다 선택해도 된다.**

# 참고

React공식 문서에서 [Hook의 개요를 설명하는 글](https://ko.reactjs.org/docs/hooks-intro.html)을 참고하면 위의 내용이 더 잘 이해될 수 있다. 애초에 Hook이 도입된 동기가 상태 관련 로직을 재사용하기 위해서였다.

>React는 컴포넌트간에 재사용 가능한 로직을 붙이는 방법을 제공하지 않습니다. (예를 들어, 스토어에 연결하는 것) 만약 이전부터 React를 사용해왔다면, 당신은 render props이나 고차 컴포넌트와 같은 패턴을 통해 이러한 문제를 해결하는 벙법에 익숙할 것입니다. 그러나 이런 패턴의 사용은 컴포넌트의 재구성을 강요하며, 코드의 추적을 어렵게 만듭니다. React 개발자 도구에서 React 애플리케이션을 본다면, providers, consumers, 고차 컴포넌트, render props 그리고 다른 추상화에 대한 레이어로 둘러싸인 “래퍼 지옥(wrapper hell)“을 볼 가능성이 높습니다. 개발자 도구에서 걸러낼 수 있지만, 이 문제의 요점은 심층적이었습니다. React는 상태 관련 로직을 공유하기 위해 좀 더 좋은 기초 요소가 필요했습니다.

Hook을 사용하면 컴포넌트로부터 상태 관련 로직을 추상화할 수 있습니다. 이를 이용해 독립적인 테스트와 재사용이 가능합니다. Hook은 계층의 변화 없이 상태 관련 로직을 재사용할 수 있도록 도와줍니다. 이것은 많은 컴포넌트 혹은 커뮤니티 사이에서 Hook을 공유하기 쉽게 만들어줍니다.
