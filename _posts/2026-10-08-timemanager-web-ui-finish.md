---
title: "TimeManager: 웹 UI 마무리, id 대신 이름 보여주기"
date: 2026-10-08 09:00:00 +0900
categories: [EPITECH, TimeManager]
tags: [vue, composable, watch, phoenix, ecto, preload, frontend]
---

팀에서 웹은 UI 완성으로 마무리하기로 했다. 남은 일은 관리자가 직원을 고르고, 역할과 스킬을 바꾸고, 태스크를 배정하는 화면을 다듬는 것이었다. 그리고 하다 보니 화면 곳곳에 `#7` 같은 id가 보이는 게 계속 거슬렸다.\\
The team decided to wrap up the web once the UI is complete. What remained was polishing the screens where a manager picks an employee, changes their role and skills, and assigns tasks. And along the way, ids like `#7` showing up all over the screen kept bothering me.

---

## 관리자 화면 다듬기

사이드바를 "Team"과 "My work"로 나누고, Team 쪽에 직원 선택 상자를 넣었다. 하나 기억에 남는 건 URL에서 꺼낸 id는 항상 문자열이라 내 id와 비교할 때 `String()`으로 맞춰야 한다는 점이었다.\\
I split the sidebar into "Team" and "My work" and added an employee picker to the Team side. One thing that stuck with me: ids taken from the URL are always strings, so comparing them with my id needed `String()`.

역할 승급·강등 기능을 넣었다. 선택 상자의 값을 `"employee"`로 썼는데 서버가 아는 역할은 `user`, `manager`, `admin`뿐이라 422가 나고 선택 상자가 빈칸이 됐다. 화면의 라벨과 서버가 받는 값은 따로라는 걸 다시 느꼈다. `$event.target.value`는 고른 option의 `value` 하나이고, 라벨이 아니다.\\
I added promoting and demoting roles. I set the option value to `"employee"`, but the server only knows `user`, `manager`, and `admin`, so it returned 422 and the select went blank. It reminded me again that the label on screen and the value the server accepts are separate. `$event.target.value` is the chosen option's `value`, not its label.

태스크 수정에선 `task`에 바로 `v-model`을 걸지 않고 `reactive` 복사본(`editForm`)을 따로 뒀다. 바로 걸면 Cancel을 눌러도 입력한 값이 화면에 남는다. 배정은 표 안 선택 상자로 바꿨는데, 서버가 이미 배정된 태스크를 다시 배정하면 거부해서 "해제 → 새로 배정" 두 번 요청을 보내야 했다.\\
For task editing, I didn't bind `v-model` directly to `task` but kept a separate `reactive` copy (`editForm`). Binding directly would leave the typed values on screen even after pressing Cancel. I moved assignment into a select inside the table, and because the server refuses to reassign an already assigned task, it has to send two requests: "unassign → assign".

## 이름은 서버가 같이 주는 게 낫다

직원은 `GET /users` 권한이 없어서, 태스크 표에 남의 담당자가 `#id`로만 보였다. 화면에서 이름을 따로 찾아오는 대신 태스크 응답 자체에 `skill_name`과 `user_name`을 넣기로 팀원과 합의하고 백엔드를 고쳤다.\\
Employees don't have permission for `GET /users`, so other people's assignees showed only as `#id` in the task table. Instead of looking up names separately on screen, I agreed with my teammate to add `skill_name` and `user_name` to the task response itself, and changed the backend.

`task_json.ex`의 `data/1` 한 곳에서 `Repo.preload(task, [:skill, :user])`를 하니 태스크를 돌려주는 응답 6개가 전부 바뀌었다. 미배정 태스크는 `user`가 `nil`이라 `task.user && …`로 감싸야 했다. 안 그러면 미배정 하나 때문에 목록 전체가 500이 된다. preload 결과를 `task =`로 다시 받아야 하는 이유는 처음에 틀렸는데, Elixir에선 값이 불변이라 원래 `task`는 바뀌지 않고 새 값이 돌아오기 때문이다.\\
Doing `Repo.preload(task, [:skill, :user])` in one place, `data/1` in `task_json.ex`, changed all six responses that return tasks. An unassigned task has `user` as `nil`, so it had to be wrapped as `task.user && …`. Otherwise a single unassigned task makes the whole list 500. I first got wrong why the preload result has to be reassigned with `task =`: in Elixir values are immutable, so the original `task` doesn't change and a new value is returned.

`/auth/me`에도 내 스킬을 넣었다. 스킬을 preload하지 않은 채로 `Enum.map`을 돌리면 아직 안 불러온 값(NotLoaded)을 돌리려다 500이 나고, 로그인 자체가 실패한다. 스킬이 없는 새 가입자는 `nil`이 아니라 빈 리스트 `[]`가 온다.\\
I also added my skills to `/auth/me`. Running `Enum.map` without preloading skills tries to iterate a not-yet-loaded value (NotLoaded), returns 500, and login itself fails. A new user with no skills gets an empty list `[]`, not `nil`.

그 덕분에 직원 화면에선 "내 태스크 + 내 스킬로 할 수 있는 미배정 태스크"만 보이게 거를 수 있었다. 다만 화면에서만 거르는 거라 API는 여전히 전부 돌려준다.\\
Thanks to that, the employee view could be filtered to show only "my tasks + unassigned tasks I have the skill for". It's only filtered on screen, though, so the API still returns everything.

## 값이 아니라 함수를 넘겨야 따라온다

Live clock, Timesheet, Insights, Add time 제목에도 이름을 보여주려고 `useUserName(getId)` composable을 만들었다. 내 id면 `currentUser`에서, 남의 id면 일단 `User #id`를 보여주고 관리자일 때만 서버에서 이름을 가져온다.\\
To show names in the titles of Live clock, Timesheet, Insights, and Add time, I made a `useUserName(getId)` composable. For my own id it uses `currentUser`; for someone else's it shows `User #id` first and fetches the name from the server only for managers.

여기서 `useUserName(() => props.userid)`처럼 함수를 넘기는 게 핵심이었다. `props.userid` 값을 그대로 넘기면 build 에러는 안 나지만 Vue가 "Invalid watch source" 경고를 띄우고, 직원을 바꿔도 제목이 따라오지 않는다. 그 순간의 값만 복사해서 넘겼기 때문이다.\\
The key was passing a function, as in `useUserName(() => props.userid)`. Passing `props.userid` as a plain value doesn't break the build, but Vue warns "Invalid watch source" and the title doesn't follow when you switch employees, because only the value at that moment was copied.

## 퇴근하면 근무 시간이 생기게

마지막으로 퇴근 버튼을 누르면 출근 시각부터 퇴근 시각까지 근무 기록이 자동으로 생기게 했다. 처음엔 백엔드에서 직전 출근 기록을 찾는 방식으로 시작했다가, 되돌리고 프론트에서 처리했다. 퇴근 요청을 보내기 전에 출근 시각을 잡아두고, 퇴근이 성공하면 그 시각과 퇴근 시각으로 근무 기록을 만든다.\\
Finally, I made pressing clock-out automatically create a working time from clock-in to clock-out. I first started by finding the previous clock-in on the backend, then reverted and handled it on the frontend. It grabs the clock-in time before sending the clock-out request, and if clock-out succeeds, creates a working time with those two times.

---

## 다음 진행할 내용

웹 UI는 이걸로 끝이다. 아직 직원이 API로 남의 출퇴근·근무 기록을 볼 수 있는 문제가 남아 있어서, 권한 쪽을 마무리해야 한다. 그리고 완성된 화면으로 튜토리얼 캡처를 다시 찍을 차례다.\\
That's it for the web UI. There's still the issue that employees can see others' clocks and working times through the API, so permissions need finishing. And it's time to retake the tutorial screenshots with the finished screens.
