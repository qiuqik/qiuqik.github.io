---

layout: post

author: Yiqiu

title: 如何使用 Reducer 管理 React 组件的状态

date: 2025-06-29 13:43:40 +0800

categories: [React]

tags: [blog]     # *TAG names should always be lowercase*

---
# 什么是 Reducer

> Reducer 是 React 中的一个概念，用于管理组件的状态。它是一个函数，接受当前状态和一个 action，返回新的状态。

一个例子：
```javascript
import { useReducer } from 'react';

function reducer(state, action) {//reducer 函数，传入 action 和 state变量
  if (action.type === 'incremented_age') {
    return {
      age: state.age + 1
    };
  }
  throw Error('Unknown action.');
}

export default function Counter() {
  const [state, dispatch] = useReducer(reducer, { age: 48 }); //初始值state.age = 48

  return (
    <>
      <button onClick={() => {
        dispatch({ type: 'incremented_age' })//调用 reducer，传入 action.type = 'incremented_age'
      }}>
        Increment age
      </button>
      <p>Hello! You are {state.age}.</p>
    </>
  );
}

```
效果：
![](https://qiu-document.oss-cn-shanghai.aliyuncs.com/markdown-pictures/PixPin_2025-06-29_14-46-42.gif)



# 为什么是 Reducer

Reducer 命名来源于数组的 reduce() 方法，以官方一个很简单的例子为例：
```javascript
const arr = [1, 2, 3, 4, 5];
const sum = arr.reduce(
  (result, number) => result + number
); // sum = 1 + 2 + 3 + 4 + 5 = 15

```
它返回 1 到 5 的累加值 15，它将每次的结果放到 `result` 中，与当前 `number` 相加后返回，这个 `result` 再与下一个 `number` 相加，直到所有的数字都相加完。

传递给 `reduce` 的函数 `(result, number) => result + number` 被称为 `reducer`，它接受 `目前的结果` 和 `当前的值` ，然后返回 `下一个结果`。 `React` 中的 `reducer` 和这个是一样的：它们都接受 `目前的状态` 和`action`，然后返回 `下一个状态`。这样，`action` 会随着时间推移累积到状态中。

# 如何将 useState 重构成 useReducer

一个 useState 例子：
```javascript
import { useState } from 'react';
import AddTask from './AddTask.js';
import TaskList from './TaskList.js';

export default function TaskApp() {
  const [tasks, setTasks] = useState(initialTasks);

  function handleAddTask(text) {
    setTasks([
      ...tasks,
      {
        id: nextId++,
        text: text,
        done: false,
      },
    ]);
  }

  function handleChangeTask(task) {
    setTasks(
      tasks.map((t) => {
        if (t.id === task.id) {
          return task;
        } else {
          return t;
        }
      })
    );
  }

  function handleDeleteTask(taskId) {
    setTasks(tasks.filter((t) => t.id !== taskId));
  }

  return (
    <>
      <h1>布拉格的行程安排</h1>
      <AddTask onAddTask={handleAddTask} />
      <TaskList
        tasks={tasks}
        onChangeTask={handleChangeTask}
        onDeleteTask={handleDeleteTask}
      />
    </>
  );
}

let nextId = 3;
const initialTasks = [
  {id: 0, text: '参观卡夫卡博物馆', done: true},
  {id: 1, text: '看木偶戏', done: false},
  {id: 2, text: '打卡列侬墙', done: false},
];
```

![](https://qiu-document.oss-cn-shanghai.aliyuncs.com/markdown-pictures/202506291502322.png)

这个组件的每个事件处理程序都通过 `setTasks` 来更新状态。随着这个组件的不断迭代，其状态逻辑也会越来越多。为了降低这种复杂度，并让所有逻辑都可以存放在一个易于理解的地方，你可以将这些状态逻辑移到组件之外的一个称为 **reducer** 的函数中。

Reducer 是处理状态的另一种方式。你可以通过三个步骤将 `useState` 迁移到 `useReducer`：

1. 将设置状态的逻辑 **修改** 成 dispatch 的一个 action；
2. **编写** 一个 reducer 函数；
3. 在你的组件中 **使用** reducer。

## 第 1 步: 将设置状态的逻辑修改成 dispatch 的一个 action 

你的事件处理程序目前是通过设置状态来 实现逻辑的：
```javascript
function handleAddTask(text) {
  setTasks([
    ...tasks,
    {
      id: nextId++,
      text: text,
      done: false,
    },
  ]);
}

function handleChangeTask(task) {
  setTasks(
    tasks.map((t) => {
      if (t.id === task.id) {
        return task;
      } else {
        return t;
      }
    })
  );
}

function handleDeleteTask(taskId) {
  setTasks(tasks.filter((t) => t.id !== taskId));
}
```
移除所有的状态设置逻辑。只留下三个事件处理函数：

- `handleAddTask(text)` 在用户点击 “添加” 时被调用。
- `handleChangeTask(task)` 在用户切换任务或点击 “保存” 时被调用。
- `handleDeleteTask(taskId)` 在用户点击 “删除” 时被调用。

使用 **reducer** 管理状态与直接设置状态略有不同。它不是通过设置状态来告诉 React “要做什么”，而是通过事件处理程序 dispatch 一个 “action” 来指明 “用户刚刚做了什么”。（而状态更新逻辑则保存在其他地方！）因此，我们不再通过事件处理器直接 “设置 `task`”，而是 dispatch 一个 “添加/修改/删除任务” 的 action。这更加符合用户的思维。

```javascript
function handleAddTask(text) {
  dispatch({
    type: 'added',
    id: nextId++,
    text: text,
  });
}

function handleChangeTask(task) {
  dispatch({
    type: 'changed',
    task: task,
  });
}

function handleDeleteTask(taskId) {
  dispatch({
    type: 'deleted',
    id: taskId,
  });
}
```
你传递给 `dispatch` 的对象叫做 “action”：
```javascript
    // "action" 对象：
    {
      type: 'deleted',
      id: taskId,
    }
```

它是一个普通的 JavaScript 对象。它的结构是由你决定的，但通常来说，它应该至少包含可以表明 发生了什么事情 的信息。

## 第 2 步: 编写一个 reducer 函数 
reducer 函数就是你放置状态逻辑的地方。它接受两个参数，分别为当前 state 和 action 对象，并且返回的是更新后的 state：
```javascript
function yourReducer(state, action) {
  // 给 React 返回更新后的状态
}
```
React 会将状态设置为你从 reducer 返回的状态。

在这个例子中，要将状态设置逻辑从事件处理程序移到 reducer 函数中，你需要：

1. 声明当前状态（tasks）作为第一个参数；
2. 声明 action 对象作为第二个参数；
3. 从 reducer 返回 下一个 状态（React 会将旧的状态设置为这个最新的状态）。

下面是所有迁移到 reducer 函数的状态设置逻辑：
```javascript
function tasksReducer(tasks, action) {
  if (action.type === 'added') {
    return [
      ...tasks,
      {
        id: action.id,
        text: action.text,
        done: false,
      },
    ];
  } else if (action.type === 'changed') {
    return tasks.map((t) => {
      if (t.id === action.task.id) {
        return action.task;
      } else {
        return t;
      }
    });
  } else if (action.type === 'deleted') {
    return tasks.filter((t) => t.id !== action.id);
  } else {
    throw Error('未知 action: ' + action.type);
  }
}
```

由于 reducer 函数接受 state（tasks）作为参数，因此你可以 在组件之外声明它。这减少了代码的缩进级别，提升了代码的可读性。

## 第 3 步: 在组件中使用 reducer 
最后，你需要将 tasksReducer 导入到组件中。记得先从 React 中导入 useReducer Hook：
```javascript
import { useReducer } from 'react';
```
接下来，你就可以替换掉之前的 useState:
```javascript
const [tasks, setTasks] = useState(initialTasks);
```
只需要像下面这样使用 useReducer:
```javascript
const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
```
useReducer 和 useState 很相似——你必须给它传递一个初始状态，它会返回一个有状态的值和一个设置该状态的函数（在这个例子中就是 dispatch 函数）。但是，它们两个之间还是有点差异的。

useReducer 钩子接受 2 个参数：

- 一个 reducer 函数
- 一个初始的 state

它返回如下内容：

- 一个有状态的值
- 一个 dispatch 函数（用来 “派发” 用户操作给 reducer）


如果有需要，可以把 reducer 与组件分离：

App.js
```javascript
import { useReducer } from 'react';
import AddTask from './AddTask.js';
import TaskList from './TaskList.js';
import tasksReducer from './tasksReducer.js';

export default function TaskApp() {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  function handleAddTask(text) {
    dispatch({
      type: 'added',
      id: nextId++,
      text: text,
    });
  }

  function handleChangeTask(task) {
    dispatch({
      type: 'changed',
      task: task,
    });
  }

  function handleDeleteTask(taskId) {
    dispatch({
      type: 'deleted',
      id: taskId,
    });
  }

  return (
    <>
      <h1>布拉格的行程安排</h1>
      <AddTask onAddTask={handleAddTask} />
      <TaskList
        tasks={tasks}
        onChangeTask={handleChangeTask}
        onDeleteTask={handleDeleteTask}
      />
    </>
  );
}

let nextId = 3;
const initialTasks = [
  {id: 0, text: '参观卡夫卡博物馆', done: true},
  {id: 1, text: '看木偶戏', done: false},
  {id: 2, text: '打卡列侬墙', done: false},
];

```

tasksReducer.js
```javascript
export default function tasksReducer(tasks, action) {
  switch (action.type) {
    case 'added': {
      return [
        ...tasks,
        {
          id: action.id,
          text: action.text,
          done: false,
        },
      ];
    }
    case 'changed': {
      return tasks.map((t) => {
        if (t.id === action.task.id) {
          return action.task;
        } else {
          return t;
        }
      });
    }
    case 'deleted': {
      return tasks.filter((t) => t.id !== action.id);
    }
    default: {
      throw Error('未知 action：' + action.type);
    }
  }
}

```

当像这样分离关注点时，我们可以更容易地理解组件逻辑。现在，事件处理程序只通过派发 action 来指定 发生了什么，而 reducer 函数通过响应 actions 来决定 状态如何更新。

# 使用 Immer 简化 reducer 
与在平常的 state 中 修改对象 和 数组 一样，你可以使用 Immer 这个库来简化 reducer。在这里，useImmerReducer 让你可以通过 `push` 或 `arr[i] =` 来修改 state ：

package.json
```json
{
  "dependencies": {
    "immer": "1.7.3",
    "react": "latest",
    "react-dom": "latest",
    "react-scripts": "latest",
    "use-immer": "0.5.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject"
  },
  "devDependencies": {}
}
```

App.js
```javascript
import { useImmerReducer } from 'use-immer';
import AddTask from './AddTask.js';
import TaskList from './TaskList.js';

function tasksReducer(draft, action) {
  switch (action.type) {
    case 'added': {
      draft.push({
        id: action.id,
        text: action.text,
        done: false,
      });
      break;
    }
    case 'changed': {
      const index = draft.findIndex((t) => t.id === action.task.id);
      draft[index] = action.task;
      break;
    }
    case 'deleted': {
      return draft.filter((t) => t.id !== action.id);
    }
    default: {
      throw Error('未知 action：' + action.type);
    }
  }
}

export default function TaskApp() {
  const [tasks, dispatch] = useImmerReducer(tasksReducer, initialTasks);

  function handleAddTask(text) {
    dispatch({
      type: 'added',
      id: nextId++,
      text: text,
    });
  }

  function handleChangeTask(task) {
    dispatch({
      type: 'changed',
      task: task,
    });
  }

  function handleDeleteTask(taskId) {
    dispatch({
      type: 'deleted',
      id: taskId,
    });
  }

  return (
    <>
      <h1>布拉格的行程安排</h1>
      <AddTask onAddTask={handleAddTask} />
      <TaskList
        tasks={tasks}
        onChangeTask={handleChangeTask}
        onDeleteTask={handleDeleteTask}
      />
    </>
  );
}

let nextId = 3;
const initialTasks = [
  {id: 0, text: '参观卡夫卡博物馆', done: true},
  {id: 1, text: '看木偶戏', done: false},
  {id: 2, text: '打卡列侬墙', done: false},
];

```
Reducer 应该是纯净的，所以它们不应该去修改 state。而 Immer 为你提供了一种特殊的 draft 对象，你可以通过它安全地修改 state。在底层，Immer 会基于当前 state 创建一个副本。这就是通过 useImmerReducer 来管理 reducer 时，可以修改第一个参数，且不需要返回一个新的 state 的原因。