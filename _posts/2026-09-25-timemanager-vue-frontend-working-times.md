---
title: "TimeManager: Vue 프론트 뼈대와 근무 시간 페이지"
date: 2026-09-25 09:00:00 +0900
categories: [EPITECH, TimeManager]
tags: [vue, vite, vue-router, proxy, computed, ref, v-for, frontend]
---

백엔드에 이어 이번엔 Vue로 프론트 뼈대를 잡고, 근무 시간(WorkingTimes) 페이지까지 만들어봤다.\\
Following the backend, this time I set up the frontend skeleton with Vue and built the WorkingTimes page.

---

## 1. 기본적인 실수들

처음 띄웠을 때 스타일이 하나도 안 먹길래 봤더니, `main.js`는 `style.css`를 불러오는데 실제 파일 이름은 `styles.css`였다.\\
When I first ran it, none of the styles applied. It turned out `main.js` was importing `style.css`, while the actual file was `styles.css`.

그다음엔 `vite: command not found`가 떴다. vite를 컴퓨터에 깔린 프로그램처럼 생각했는데, 사실은 프로젝트 안에 설치되는 도구라서 `npm install`부터 해야 했다.\\
Next I got `vite: command not found`. I had thought of vite as a program installed on my machine, but it's actually a tool installed inside the project, so I had to run `npm install` first.

## 2. 백엔드, 프론트 엔드 이해

프론트는 5173번, 백엔드는 4000번 포트에서 돈다. 브라우저는 5173에만 요청을 보내고, `/api`로 시작하는 요청은 Vite가 4000번으로 대신 전달해 준다. `curl localhost:5173/api/users`로 200이 오는 것까지 확인했다.\\
The frontend runs on port 5173 and the backend on 4000. The browser only talks to 5173, and Vite forwards any request starting with `/api` to 4000. I confirmed it with `curl localhost:5173/api/users` returning 200.

## 3. 헷갈렸던 것들

사이드바에는 선택된 userId만 보여주기로 팀원들과 정했다. 임시로 `ref('1')`을 두었다. 바꾸고 싶으면 `ref`는 `userId.value = '7'`처럼 바꿔야 하고, 그래야 Vue가 알아채고 화면도 다시 그린다. (`const`라서 생기는 문제.)\\
We agreed as a team to show only the selected userId in the sidebar. I put a temporary `ref('1')`. If someone wants to change it, they need to change `userId.value = '7'`, and that's what lets Vue notice and re-render. (Because it is `const`)

메뉴 링크를 만들면서는 `:to`처럼 `:`를 붙여야 코드로 계산된 값이 들어가고, `${}`는 백틱 문자열에서만 된다는 걸 다시 확인했다. \\
While building the menu links, I re-confirmed that `:to` needs the `:` to take a computed value, and that `${}` only works inside backtick strings. 

## 4. 제목은 주소에서

위쪽 바 제목은 `useRoute()`로 지금 주소를 읽고, `split('/')`로 나눈 첫 조각을 보고 정했다. `'/workingTimes/1'`을 나누면 맨 앞에 빈 문자열이 생겨서 `['', 'workingTimes', '1']`이 된다. 여기에 `computed`를 썼는데, 그냥 함수로 써도 화면은 바뀐다. 차이는 `computed`가 결과를 기억해 두고 필요할 때만 다시 계산한다는 점이었다.\\
The top bar title reads the current path with `useRoute()` and uses the first segment after `split('/')`. Splitting `'/workingTimes/1'` gives an empty string at the front: `['', 'workingTimes', '1']`. I used `computed` here, though a plain function would also update the screen. The difference is that `computed` caches the result and only recalculates when needed.

## 하얀 화면

WorkingTimes 페이지와 라우트를 붙였더니 앱 전체가 하얗게 됐다. F12 Console을 열어보니 라우트 경로 앞에 `/`를 빠뜨려서 라우터가 만들어지다가 에러가 난 거였다. 고치고 나니 이번엔 userId 자리가 비어 있었는데, 라우트에는 `:userID`, 코드에서는 `params.userId`로 읽고 있었다. 대소문자 하나 때문에 `undefined`였다.\\
After adding the WorkingTimes page and route, the whole app went white. The F12 Console showed I had left out the leading `/` in the route path, so the router errored while being created. Once that was fixed, the userId spot was empty: the route said `:userID` but the code read `params.userId`. One letter's case made it `undefined`.

`<RouterView />`는 페이지가 들어갈 빈자리다. 이걸 빼보면 위쪽 바 제목은 계속 바뀌는데 본문만 사라진다. 라우터는 주소를 알고 있지만, 페이지를 그릴 자리가 없는 것이다. 에러도 안 나서 처음엔 헷갈렸다.\\
`<RouterView />` is the empty slot where the page goes. Without it, the top bar title still changes but the body disappears. The router knows the path, but there's nowhere to render the page. No error shows up either, which confused me at first.

## 컴포넌트로 쪼개기

`App.vue`에 몰려 있던 사이드바와 위쪽 바를 `Sidebar.vue`, `Topbar.vue`로 옮겼다. 이제 `App.vue`는 `<Sidebar />`, `<Topbar />`, `<RouterView />`를 배치하는 일만 한다.\\
I moved the sidebar and top bar out of `App.vue` into `Sidebar.vue` and `Topbar.vue`. Now `App.vue` only lays out `<Sidebar />`, `<Topbar />`, and `<RouterView />`.
