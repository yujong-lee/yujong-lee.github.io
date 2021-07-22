---
title: "Redux 리듀서 Clojure로 작성해보기"
category: dev
---

현재 Clojure에 대한 코드 블럭이 이상함. 나중에 수정 예정.

## Javscript

~~~js
const initialState = {
  isLogBookOpen: false,
  completedTasks: [],
  selectedTaskId: 0,
  parentId: 0,
  nextTaskId: 1,
  remainingTasks: {
    0: { title: 'root', subTasks: [], isOpen: true },
  },
};

addTask: (state, action) => {
  const { selectedTaskId, nextTaskId } = state;
  const { payload: newTaskTitle } = action;

  const typed = (newTaskTitle !== '');

  if (!typed) {
    return;
  }

  state.nextTaskId = nextTaskId + 1;

  state.remainingTasks[selectedTaskId].subTasks.unshift(nextTaskId);

  const newTask = { title: newTaskTitle, subTasks: [], isOpen: true };

  state.remainingTasks[nextTaskId] = newTask;
}
~~~

## Clojure(script)

```clj
(def oldState
  (atom
   {:selectedTaskId :0
    :nextTaskId :2
    :remainingTasks {:0 {:title "root" :subTasks [:1] :isOpen true}
                     :1 {:title "task1" :subTasks [] :isOpen true}}}))

(defn updateId
  [updator id]
  (-> id
      name
      Integer/parseInt
      updator
      str
      keyword))

(defn addTask
  [newTaskTitle]
  (let [{selectedTaskId :selectedTaskId
         nextTaskId :nextTaskId}
        @oldState]
    (when-not (= newTaskTitle "")
      (swap! oldState update-in [:nextTaskId] #(updateId inc %))
      (swap! oldState update-in [:remainingTasks selectedTaskId :subTasks] #(conj % nextTaskId))
      (swap! oldState update-in [:remainingTasks] #(assoc % nextTaskId {:title newTaskTitle :subTasks [] :isOpen true})))))
```

매크로를 이용하면 swap! 부분을 아래와 같이 간단하게 바꿀 수 있다. 이것은 함수를 정의한 것과 다르다. 새로운 문법을 만들어낸 것이다.

```clj
(defmacro updateState
  [state & args]
  `(do ~(for [reducer (partition 2 args)]
    `(swap! ~state update-in ~@reducer))))

(when-not (= newTaskTitle "")
    (updateState
    oldState
    [:nextTaskId] #(updateId inc %)
    [:remainingTasks] #(assoc % nextTaskId {:title newTaskTitle :subTasks [] :isOpen true})
    [:remainingTasks selectedTaskId :subTasks] #(conj % nextTaskId)))
```
