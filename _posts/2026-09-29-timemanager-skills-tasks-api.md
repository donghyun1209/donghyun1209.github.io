---
title: "TimeManager: Skills와 Tasks API, DB가 지켜주는 규칙들"
date: 2026-09-29 09:00:00 +0900
categories: [EPITECH, TimeManager]
tags: [elixir, phoenix, ecto, migration, postgresql, foreign-key, changeset]
---

Users에 이어 이번엔 직원에게 스킬을 붙이고, 스킬이 필요한 태스크를 배정하는 API를 만들었다. 테이블은 `skills`, `tasks`, 그리고 둘을 잇는 `user_skills` 세 개다.\\
After Users, this time I built an API for giving employees skills and assigning tasks that require a skill. There are three tables: `skills`, `tasks`, and `user_skills` connecting them.

---

## 부모를 지우면 자식은 어떻게 되나

`tasks`는 태스크 하나에 스킬 하나를 두고, `user_id`는 비워둘 수 있게 했다(아직 아무에게도 배정 안 된 태스크). 여기서 `skill_id`에 `on_delete: :restrict`를 걸었는데, 자식이 남아 있으면 **부모 삭제 자체가 거부**된다.\\
In `tasks`, each task has one skill, and `user_id` can be empty (a task not yet assigned to anyone). I put `on_delete: :restrict` on `skill_id` as long as children remain, **deleting the parent itself is refused**.

`:nilify_all`은 부모가 지워지면 자식 쪽 값을 NULL로 바꾼다. 그런데 그 컬럼에 `null: false`까지 걸려 있으면 NULL로 바꿀 수가 없으니, 결국 부모 삭제가 에러가 된다. 옵션 두 개가 서로 부딪히는 경우였다.\\
`:nilify_all` sets the child's value to NULL when the parent is deleted. But if that column also has `null: false`, it can't become NULL, so deleting the parent ends up as an error. The two options collide.

하나 더 이상했던 건 `type :x` 같은 잘못된 줄을 써도 미리 알려주는 곳이 없었다는 점이다. Elixir는 괄호를 생략할 수 있어서 이 줄을 `type(:x)`라는 함수 호출로 읽고, 모양만 보면 문제가 없다. 원래는 컴파일할 때 "그런 함수는 없다"는 에러가 나야 하는데, migration은 `.exs` 스크립트라 `mix compile` 때는 아예 컴파일되지 않는다. 그래서 `mix ecto.migrate`를 실행하는 순간에야 컴파일되면서 실패한다.\\
Another odd thing was that nothing warned me in advance about a wrong line like `type :x`. Elixir lets you drop parentheses, so it reads the line as a function call `type(:x)`, and its shape is fine. Normally compiling would fail with "no such function", but migrations are `.exs` scripts that `mix compile` doesn't compile at all. So it only gets compiled, and fails, the moment `mix ecto.migrate` runs.

## `cast`가 조용히 버리는 것들

`Skill` schema와 `Skills` context를 만들면서 `cast`의 허용 목록 밖에 있는 키는 에러 없이 조용히 버려진다는 걸 확인했다. 사용자가 보내면 안 되는 필드를 몰래 끼워 넣어도 무시되니, 이게 mass assignment 방어가 된다.\\
While building the `Skill` schema and `Skills` context, I confirmed that keys outside `cast`'s allow list are silently dropped. Even if a user sneaks in a field they shouldn't send, it's ignored, which is what defends against mass assignment.

그 밖에 `get!`는 없으면 예외를 던지고 Phoenix가 그걸 404로 바꿔준다는 것, changeset이 유효하지 않으면 `Repo.insert`는 DB까지 가지도 않고 `{:error, changeset}`을 돌려준다는 것도 정리했다.\\
I also noted that `get!` raises when nothing is found and Phoenix turns that into a 404, and that when a changeset is invalid, `Repo.insert` doesn't even reach the DB and just returns `{:error, changeset}`.
---

## 다음 진행할 내용

API는 생겼지만 아직 화면에서 부를 곳이 없다. 다음은 로그인부터 붙이고, 그다음 Tasks와 Skills 화면을 이 API에 연결할 생각이다.\\
The API exists, but nothing on screen calls it yet. Next I plan to add login first, then connect the Tasks and Skills pages to this API.
