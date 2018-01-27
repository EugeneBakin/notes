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

Первый запуск редьюсера происходит с (state = undefined, action = { type: @@INIT }). Это внутренний action редакса, который обрабатывать не рекомендуется. А нужен он для того, чтобы редьюсер проинициализировал начальное состояние. 

Вообще говоря редьюсер всего один, но никто не мешает разбивать его на более маленькие функции (особенно, если приложение - крупное).  
Эти функции поменьше также принято называть редьюсерами, если они также принимают первым параметром state и вторым action, и от них ожидается то же самое поведение, что и от родительского редьюсера (т.е. вернуть state по умолчанию, если пришел undefined, referential equality (TODO))

## combineReducers

Из коробки в редаксе доступна утилитка `combineReducers` позволяющая создать редьюсер высшего порядка, который будет разбивать состояние-объект по ключам и для каждого из ключей запускать собственный под-редьюсер. 

(TODO code sample)

Часто combineReducers может быть недостаточно. (Описание проблемы) (Здесь примеры решения проблемы через thunks и через новый редьюсер)

Судя по багтрекеру редакса частым запросом является добавление в дочерние редьюсеры combineReducers общего состояния приложения третьим параметром.
Однако эти запросы отклоняются, кмк, по трем причинам:
1. Нет четкого понимания, что делать, если дочерний редьюсер сам становится родителем. Нужно ли пробрасывать полный стейт в его детей, или только его собственный стейт.
2. Нет понимания какой именно стейт нужно пробрасывать. Оригинальный, или уже видоизмененный прошлыми дочерними редьюсерами.
3. Такое поведение у встроенной утилиты провоцирует писать редьюсеры сильно связанные друг с другом.

В целом совет таков - использовать combineReducers для простых случаев, (а сложных случаев избегать) а если приспичило писать явный дочерний редьюсер, куда явно передается нужный кусок общего состояния.

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
