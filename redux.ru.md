---
title: Redux
author: Eugene Bakin
date: 18.01.2018
--- 

[Redux](https://redux.js.org/) - это небольшая библиотечка для работы с состоянием приложения.  
Основная идея заключается в том, что вся информация, необходимая для отрисовки приложения содержится в одном POJO объекте.

Объект хранится в хранилище **store**.
Каждое обновление состояние приложения происходит через создание действия **action** и уведомления **store** об этом действии - т.е. **dispatch**.

На каждый **action** **store** запускает **reducer** - чистую функцию, которая возвращает новое состояние приложения на основе предыдущего состояния и обрабатываемого **action**.

Показывающий код (Обычно React) подписывается на изменения состояния приложения и соответствующим образом обновляет DOM.

Таким образом жизненый цикл приложения выглядит следующим образом:
* Происходит некоторое событие (Действие пользователя, setTimeout, ajaxCallback)
* Создаем **action** - POJO, описывающий это событие и **dispatch**'им его
* **store** запускает **reducer** и получает новый **state**
* Показывающий код получает уведомление и отображает новое состояние приложения

Основы **Redux** хорошо описаны в бесплатных курсах от его создателя:
* [Getting Started with Redux](https://egghead.io/series/getting-started-with-redux)
* [Building React Applications with Idiomatic Redux](https://egghead.io/series/getting-started-with-redux)

(TODO) Философия, hot reloading, time travel

# State, Action и Reducer

Эти понятия неразрывно связаны. Редьюсер генерирует новое состояние приложения на основе пришедшего действия.

Первый запуск редьюсера происходит с `(state = undefined, action = { type: @@INIT })`. Это внутренний action редакса, который обрабатывать не рекомендуется. А нужен он для того, чтобы редьюсер проинициализировал начальное состояние. 

Вообще говоря редьюсер всего один, но никто не мешает разбивать его на более маленькие функции (особенно, если приложение - крупное).  
Эти функции поменьше также принято называть редьюсерами, если они также принимают первым параметром state и вторым action, и от них ожидается то же самое поведение, что и от родительского редьюсера (т.е. вернуть state по умолчанию, если пришел undefined, referential equality (TODO))

## combineReducers
 
Из коробки в редаксе доступна утилитка `combineReducers` позволяющая создать редьюсер высшего порядка, который будет разбивать состояние-объект по ключам и для каждого из ключей запускать собственный под-редьюсер. 

[jsfiddle](https://jsfiddle.net/bakineugene/6y50z3a2/)
```javascript
import { combineReducers } from 'redux';

const PLUS1 = 'PLUS1';
const MINUS1 = 'MINUS1';

const reducerTemplate = (state = 0, action) => {
  switch (action.type) {
    case PLUS1:
      return state + 1;
    case MINUS1:
      return state - 1;
    default:
      return state;
  }
};

const reducer = combineReducers({
  counterA: reducerTemplate,
  counterB: reducerTemplate
});

reducer({ counterA: 5, counterB: 7 }, { type: PLUS1 }); // { counterA: 6, counter2: 8 }

```

Иногда combineReducers может быть недостаточно. Если для вычисления нового состояния недостаточно информации из данного ключа. Например если в примере выше мы захотим сделать дейтсвие, которое сбрасывает счетчики если сумма всех счетчиков - четное число. На уровне одного под-редьюсера очевидно недостаточно информации, чтобы принять такое решение. 

Есть три варианта решения этой проблемы.

### Пробрасывать общее состояние в дочерние редьюсеры

[jsfiddle](https://jsfiddle.net/bakineugene/k5jxug3z/1/)
```javascript

const CLEAR_IF_EVEN = 'CLEAR_IF_EVEN';

const reducerTemplate = (state = 0, action, global) => {

  switch (action.type) {
    case CLEAR_IF_EVEN:
      if ((global.counterA + global.counterB) % 2 == 0) {
        return 0;
      }
      return state;
    default:
      return state;
  }
};

const reducer = (state, action) => ({
  counterA: reducerTemplate(state.counterA, action, state),
  counterB: reducerTemplate(state.counterB, action, state)
});

console.log(reducer({counterA: 1, counterB: 1}, {type: CLEAR_IF_EVEN})); // { counterA: 0, counterB: 0 }
console.log(reducer({counterA: 2, counterB: 1}, {type: CLEAR_IF_EVEN})); // { counterA: 2, counterB: 1 }

```

Судя по багтрекеру редакса частым запросом является добавление в дочерние редьюсеры combineReducers общего состояния приложения третьим параметром.
Однако эти запросы отклоняются, кмк, по трем причинам:
1. Нет четкого понимания, что делать, если дочерний редьюсер сам становится родителем. Нужно ли пробрасывать полный стейт в его детей, или только его собственный стейт.
2. Нет понимания какой именно стейт нужно пробрасывать. Оригинальный, или уже видоизмененный прошлыми дочерними редьюсерами.
3. Такое поведение у встроенной утилиты провоцирует писать редьюсеры сильно связанные друг с другом.

В целом совет таков - использовать combineReducers для простых случаев, (а сложных случаев избегать) а если приспичило - писать отдельный дочерний редьюсер, куда явно передается нужный кусок общего состояния (см. далее). Но не стоит делать так по умолчанию.

### Использовать redux-thunk

redux-thunk - это middleware позволяющее, передавать в `store.dispatch` функцию вместо POJO. Эта функция будет немедленно вызвана с параметрами `(dispatch, getState)`; `getState()` дает возможность получить доступ к полному состоянию приложения. `dispatch` - отправить действие.


[jsfiddle](https://jsfiddle.net/bakineugene/xpum4b38/)
```javascript

const CLEAR = 'CLEAR';
import { combineReducers, createStore, applyMiddleware } from 'redux';
import ReduxThunk from 'redux-thunk';

const reducerTemplate = (state = 0, action) => {
  switch (action.type) {
    case CLEAR:
      return 0;
    default:
      return state;
  }
};

const reducer = combineReducers({
  counterA: reducerTemplate,
  counterB: reducerTemplate
});

const thunkAction = (dispatch, getState) => {
  const state = getState();
  if ((state.counterA + state.counterB) % 2 == 0) {
    dispatch({ type: CLEAR });
  }
};

const store = createStore(reducer, { counterA: 1, counterB: 2 }, applyMiddleware(ReduxThunk));
store.dispatch(thunkAction);
console.log(store.getState()) // { counterA: 1, counterB: 2 }

const storeEven = createStore(reducer, { counterA: 2, counterB: 2 }, applyMiddleware(ReduxThunk));
storeEven.dispatch(thunkAction);
console.log(storeEven.getState()) // { counterA: 0, counterB: 0 }

```

По моему мнению, этот вариант плох тем, что логика начинает выноситься за пределы редакса в так называемые **action creator**'ы. Между тем, если подумать, то action creator - это не часть редакса. С таким подхом редьюсеры постепенно превращаются в набор сеттеров. Скажем в примере выше можно было бы сделать вместо CLEAR - SET_COUNTER. 

А можно вынести вообще всю логику из редьюсеров и прийти к вырожденному случаю:

[jsfiddle](https://jsfiddle.net/bakineugene/pb1mk0t6/)
```javascript
const SET = 'SET';

const reducer = (state = {}, action) => {
  switch (action.type) {
    case SET:
      return action.state;
    default:
     return state;
    }
};


const thunkAction = (dispatch, getState) => {
  const state = getState();
    if ((state.counterA + state.counterB) % 2 == 0) {
    dispatch({
      type: SET,
      state: {
        counterA: 0,
        counterB: 0
      }
    });
  }
};
```

### Написать отдельный редьюсер, требующий дополнительных данных

[jsfiddle](https://jsfiddle.net/bakineugene/5fbfz2e2/)
```javascript

import { combineReducers } from 'redux';

const PLUS1 = 'PLUS1';
const MINUS1 = 'MINUS1';
const CLEAR_IF_EVEN = 'CLEAR_IF_EVEN';

const reducerTemplate = (state = 0, action) => {
  switch (action.type) {
    case PLUS1:
      return state + 1;
    case MINUS1:
      return state - 1;
    default:
      return state;
  }
};

const combinedReducer = combineReducers({
  counterA: reducerTemplate,
  counterB: reducerTemplate
});

const isEven = (state) => (state.counterA + state.counterB) % 2 == 0;

const clear = (state) => ({
    counterA: 0,
    counterB: 0
});

const reducer = (state, action) => {
  state = combinedReducer(state, action);

  switch (action.type) {
    case CLEAR_IF_EVEN:
      if(isEven(state)) {
        return clear(state);
      }
      return state;
    default:
      return state;
  }
};

reducer({ counterA: 5, counterB: 7 }, { type: CLEAR_IF_EVEN }); // { counterA: 0, counter2: 0 }
reducer({ counterA: 6, counterB: 7 }, { type: CLEAR_IF_EVEN }); // { counterA: 6, counter2: 7 }

```

Пока этот вариант мне нравится больше всего.

Да, остальные, при некоторых обстоятельствах, возможно, также имеют право на жизнь. Но стремиться все же стоит к тому, чтобы логика лежала в редьюсере.
Как я прочел в одном из тикетов в редаксовском баг трекере - понять куда надо складывать логику - значит постичь мастерство в редаксе.

## Referential Equality

Поскольку приложение будет прерисовываться на каждое изменение в состоянии - а перерисовываться полностью обычно дорого (даже для React), то на стороне рендеринга придется делать проверки - на необходимость перерисовки какого-либо компонента. Если состояние - это сложный объект с высоким уровнем вложенности (как обычно и бывает), то единственный способ удостовериться, что состояние действительно изменилось - сделать `deepEqual` т.е. рекурсивно сравнить между собой все свойства всех вложенных объектов.  

Но, если мы договоримся писать редьюсеры так, чтобы он возвращал новый объект только если состояние, за которое редьюсер отвечает, поменялось - то мы сможем проверять на необходимость новой отрисовки простым `===`

Коробочный `combineReducers` - соблюдает эту договоренность. `Object.assign({}, {prop: 1}, {prop: 1})` - ясное дело, нет (а соответственно и Object Rest Spread оператор).

## Immutability

Вместе с `Referential Equality` бок о бок идет `Immutability` - важное свойство, которое нужно соблюдать. Если напрямую изменить состояние приложения какой-либо мутирующей операцией `store.getState().property = 'changedProperty';` - то обнаружить что состояние приложения изменилось становится невозможно. 

## Нормализация данных

Рекомендуется, чтобы стейт не содержал дублирующейся информации. Если что-то может быть вычислено - оно должно вычисляться, а не дублироваться в стейте.

Для нормализации вложенных данных есть специальные инструменты вроде [Normalizr](https://github.com/paularmstrong/normalizr) и [Redux ORM](https://github.com/tommikaikkonen/redux-orm), но поскольку для моих задач они пока не очень подходят - ничего конкретного сказать про них не могу. 

### Селекторы

Селектор - специальная функция, вычисляющая из минимальной информации, содержащейся в стейте - некую производную информацию, с которой удобнее работать на этапе рендеринга.

Предположим у нас есть редьюсер, отслеживающий кол-во запущенных асинхронных действий.  
Каждое действие может требовать от нас показа значка загрузки пользователю, а может не требовать.  
Для этого в стейте у нас есть специальный ключ - isLoading.

```javascript
const actionReducer = (state = [], action) => {
  switch (action.type) {
    case START_ACTION:
      return Object.assign({}, state, {
        [action.id]: action.showLoading
      });
    case END_ACTION:
      return Object.keys(state).reduce((newState, key) => {
        if (key !== action.id) {
          newState[key] = state[key];
        }
        return newState;
      }, {})
  }
};

const reducer = (state, action) => {
  switch (action.type) {
    case START_ACTION:
    case END_ACTION:
      const actions = actionReducer(state.actions, action);
      return Object.assign({}, state, {
        actions,
        isLoading: actions.reduce((isLoading, nextAction) => {
          return isLoading || nextAction;
        }, false)
      });
  }
}
```

Стейт содержит лишнюю информацию, поскольку параметр `isLoading` явно вычисляется из массива `actions`, также имеющегося в стейте.

Можно вынести эту информацию в селектор и вызывать по необходимости при рендеринге.

```javascript
const reducer = (state, action) => {
  switch (action.type) {
    case START_ACTION:
    case END_ACTION:
      const actions = actionReducer(state.actions, action);
      return Object.assign({}, state, {
        actions,
        isLoading: 
      });
  }
}

const isLoading = state => state.actions.reduce((isLoading, nextAction) => {
  return isLoading || nextAction;
}, false);

console.log(isLoading({ actions: [false, true] })) // true
console.log(isLoading({ actions: [false, false] })) // false

```

### Reselect

[runkit](https://runkit.com/eugenebakin/5a81a2c88ada1b0012f2c6be)

С классическими селекторами из примера выше есть две проблемы:

1. Вычисление, которое делает селектор, будет повторяться на каждое изменение состояния, даже если та часть состояния, от которой зависит результат - не поменялась.
2. Из-за первой проблемы возникает вторая - если селектор возвращает сложную сущность (массив или объект), то это будет новый объект для каждого запуска селектора. О `Referential Equality` можно забыть.

На помощь приходит небольшая библиотечка [reselect](https://github.com/reactjs/reselect), задача которой - мемоизировать все вызовы селектора. Т.е. вызов с одним набором параметров всегда вернет один и тот же результат. Нет лишних вычислений, а также сохраняется Referential Equality насколько это возможно.

```javascript
var { createSelector } = require('reselect');
 
const packages = {
  package1: {
    items: [ 1, 2, 3 ]
  },
  package2: {
    items: [ 4, 5, 6 ]
  }
};
 
const getPackage1 = state => state.package1;
const getPackage2 = state => state.package2;
const getItems = state => state.items;

const getPackage1Items = createSelector(
  getPackage1,
  getItems
);
const getPackage2Items = createSelector(
  getPackage2,
  getItems
);

const getAllItems = createSelector(
  getPackage1Items, getPackage2Items,
  (items1, items2) => [].concat(items1, items2)
);

console.log(getAllItems(packages)) // [1, 2, 3, 4, 5, 6]

const result1 = getAllItems(packages);
const result2 = getAllItems(packages);

console.log(result1 === result2); // true
``` 

## Композиция редьюсеров

Доки и примеры редакса содержат примеры написания редьюсеров для вложенных состояний. В этих случаях советуют использовать композицию редьюсеров.

Для каждого из вложенных состояний пишется свой редьюсер обрабатывающий соответствующее действие на уровне этого состояния.

Т.е. Редьюсер обрабатывающий массив объектов вызывает на действие добавления элемента в этот массив - вызывает дочерний редьюсер отвечающий за уровень объекта, чтобы проинициализировать состояние, и добавляет его к массиву.

```javascript
const CREATE_SWITCH = 'CREATE_SWITCH';
const SWITCH = 'SWITCH';

const switchReducer = (state = false, action) => {
  switch (action.type) {
  case (SWITCH): 
    return !state;
  default:
    return state;
  }
};

const switchesReducer = (state = {}, action) => {
  switch (action.type) {
  case (CREATE_SWITCH):
  case (SWITCH):
    return Object.assign({}, state, {[action.id]: switchReducer(state[action.id], action)});  
  default:
    return state;
  }
};

let state = switchesReducer(undefined, {type: 'INIT'});
console.log(state); // {}
state = switchesReducer(state, {type: CREATE_SWITCH, id: 's1'});
state = switchesReducer(state, {type: CREATE_SWITCH, id: 's2'});
state = switchesReducer(state, {type: CREATE_SWITCH, id: 's3'});
console.log(state); // {s1: false, s2: false, s3: false}
state = switchesReducer(state, {type: SWITCH, id: 's2'});
console.log(state); // {s1: false, s2: true, s3: false}
```

Хороший пример из доков редакса [github](https://github.com/reactjs/redux/tree/master/examples/tree-view)

## Action Creators

## Immutability
    * immutability, shallow copy и ... destructuring (immutable performance (https://redux.js.org/docs/recipes/reducers/ImmutableUpdatePatterns.html))
        * Dan советует deepFreeze в тестах, чтобы избежать случайных мутаций
        * https://github.com/kolodny/immutability-helper
        * https://github.com/mweststrate/immer
        * https://redux.js.org/docs/faq/ImmutableData.html#immutability-issues-with-redux


# Middleware

    * redux-logger
    * redux-undo
    * Позволяет перехватывать actions на пути к reducers и реагировать на них
    * middleware principles (http://blog.krawaller.se/posts/exploring-redux-middleware/)
    * thunk middleware (examples/async)
    * requests middleware (OOP, examples/real-world)
    * sagas middleware https://github.com/redux-saga/redux-saga
    * (redux-thunks) позвляет вынести всю логику в action (антипаттерн, на мой взгляд

# Code Structure

https://redux.js.org/docs/faq/CodeStructure.html

# Performance

https://redux.js.org/docs/faq/Performance.html

# Ecosystem

https://redux.js.org/docs/introduction/Ecosystem.html

# Store
    * store enhancers 
    * update batching
    * rxjs integration

# redux hot reloading and time travel
    * [Webpack Hot Reloading and React](https://ctheu.com/2015/12/29/webpack-hot-reloading-and-react-how/)
    * https://github.com/gaearon/react-transform-hmr
    * https://github.com/glenjamin/ultimate-hot-reloading-example
    * [DevTools](https://github.com/gaearon/redux-devtools)
    * https://github.com/chentsulin/electron-react-boilerplate
    * https://github.com/facebook/react-devtools 
    * https://medium.com/the-web-tub/time-travel-in-react-redux-apps-using-the-redux-devtools-5e94eba5e7c0
    * hot reloading (replaceReducer)
    * examples/universal
    * examples/real-world
    * https://daveceddia.com/hot-reloading-create-react-app/

[rest](https://redux.js.org/docs/recipes/StructuringReducers.html)

https://github.com/gaearon/redux-devtools
