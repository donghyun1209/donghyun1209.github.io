---
title: "TimeManager: 화면을 API에 연결하고 프로필 페이지 만들기"
date: 2026-10-05 09:00:00 +0900
categories: [EPITECH, TimeManager]
tags: [vue, fetch, rest-api, css, scoped-css, git, merge-conflict]
---

만들어둔 API들을 화면에 연결하고, 내 정보를 보고 고칠 수 있는 프로필 페이지를 만들었다. 둘 다 결국 "서버가 어떤 모양을 기대하는지"를 맞추는 일이었다.\\
I connected APIs to the screens and built a profile page where you can view and edit your own info. Both came down to matching "what shape the server expects".

---

## 감쌀지 말지는 서버가 정한다

`api.js`의 `request()`는 서버 응답에서 `data`를 벗겨서 돌려준다. 그래서 `listRoles()`는 바로 배열을 주고, 습관처럼 `roles.data`라고 쓰면 에러 없이 `undefined`가 된다.\\
`request()` in `api.js` unwraps `data` from the server's response. So `listRoles()` returns the array directly, and writing `roles.data` out of habit gives `undefined` with no error.

더 헷갈린 건 바디였다. 회원가입은 `body: { user }`로 보냈더니 `{"user": {...}}`로 한 겹 감싸져서 서버가 password를 못 찾았다. 그래서 `body: user`로 풀었다. 그런데 태스크 생성은 반대로 서버가 `"task"` 키를 꺼내 읽어서 `{ task }`로 감싸야 했다. 같은 파일 안에서도 함수마다 다르니, 매번 컨트롤러가 어떤 키를 읽는지 확인하는 수밖에 없었다.\\
The body was more confusing. Sign-up sent `body: { user }`, which wrapped it as `{"user": {...}}`, so the server couldn't find the password. I unwrapped it to `body: user`. But task creation was the opposite: the server reads the `"task"` key, so it had to be wrapped as `{ task }`. Even within one file it differs by function, so the only way was to check which key each controller reads.

프로필 수정(`updateUser`)에서도 똑같이 걸렸는데, 이번엔 더 조용했다. 감싸서 보내면 서버는 200을 주는데 아무것도 안 바뀐다. 화면엔 성공한 것처럼 보이니 DB를 직접 봐야 알 수 있었다.\\
Profile update (`updateUser`) hit the same issue, but more quietly. Sent wrapped, the server returns 200 but nothing changes. The screen looks like it succeeded, so I could only tell by checking the DB.

## 프로필: 없는 값 하나에 화면이 통째로 깨진다

프로필은 로그인 때 받아둔 `currentUser`를 그대로 썼다. 처음엔 `v-if`를 안 걸어도 "마지막 사용자 정보"가 남아 있을 줄 알았는데, 로그아웃해서 `currentUser`가 `null`이 되면 `null.username`에서 TypeError가 나서 렌더링 자체가 깨진다.\\
The profile reuses `currentUser` from login. At first I thought that without a `v-if` the "last user info" would just stay, but after logout `currentUser` becomes `null`, and `null.username` throws a TypeError that breaks rendering entirely.

## 폼은 복사본으로

수정 폼은 `currentUser`의 값을 복사해서 채웠다. 그래서 폼에 입력해도 위쪽 카드는 안 바뀐다. 저장에 성공하면 그때 서버 응답을 `currentUser`에 합쳐서 카드와 사이드바 이름이 같이 바뀐다.\\
The edit form is filled with a copy of `currentUser`. So typing in the form doesn't change the card above. Only when saving succeeds does the server response get merged into `currentUser`, and the card and sidebar name update together.

`save()`를 붙이면서 `ref` import, `updateUser` import, `currentUser` 갱신 줄 세 개를 빠뜨렸다. `ref` 하나가 빠지자 페이지 전체가 깨졌는데, build는 통과했다. build는 쓰는 이름이 실제로 있는지까지는 봐주지 않는다.\\
While wiring `save()`, I missed three things: the `ref` import, the `updateUser` import, and the line that updates `currentUser`. Missing just `ref` broke the whole page, yet the build passed. The build doesn't check whether the names you use actually exist.

---

## 다음 진행할 내용

화면과 API는 대부분 연결됐다. 다음은 남은 웹 UI를 정리하고, 직원 이름이 id 대신 보이도록 백엔드 응답도 조금 손볼 생각이다.\\
Most screens are now connected to the API. Next I plan to clean up the remaining web UI and adjust the backend responses a bit so employee names show instead of ids.
