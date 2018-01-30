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


## Нормализация данных

https://redux.js.org/docs/recipes/reducers/NormalizingStateShape.html
https://redux.js.org/docs/recipes/reducers/UpdatingNormalizedData.html

### Селекторы
    * [Querying a Redux Store](https://medium.com/@adamrackis/querying-a-redux-store-37db8c7f3b0f)
    * [Normalizing Redux Stores for Maximum Code Reuse](https://medium.com/@adamrackis/normalizing-redux-stores-for-maximum-code-reuse-ae6e3844ae95)
    * https://github.com/gaearon/redux-devtools
    * redux normalizr
    * [Practical Redux: Redux-ORM Basics](http://blog.isquaredsoftware.com/2016/10/practical-redux-part-1-redux-orm-basics/)
    * [Practical Redux: Redux-ORM Concepts and Techniques](http://blog.isquaredsoftware.com/2016/10/practical-redux-part-2-redux-orm-concepts-and-techniques/)

### Reselect

### Normalizr
    * examples/real-world


## Примеры

redux/examples/tree-view


## Action Creators

## Immutability
    * immutability, shallow copy и ... destructuring (immutable performance (https://redux.js.org/docs/recipes/reducers/ImmutableUpdatePatterns.html))
        * Dan советует deepFreeze в тестах, чтобы избежать случайных мутаций
        * refrential equality (combineReducers supports it)
        * ... destructuring support
        * https://reactjs.org/docs/update.html
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
