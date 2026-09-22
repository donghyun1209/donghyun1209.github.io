---
title: "TimeManager: Users 컨트롤러와 CRUD"
date: 2026-09-22 09:00:00 +0900
categories: [EPITECH, TimeManager]
tags: [elixir, phoenix, controller, router, changeset, crud, json]
---

Users 컨텍스트, 라우터, 컨트롤러, JSON 뷰까지 직접 만들어서 CRUD를 끝까지 이어봤다.\\
I built the Users context, router, controller, and JSON view myself, wiring up a full CRUD flow.

---

## 1. create/update엔 changeset, delete엔 필요 없다

`create_user`/`update_user`는 값을 검증해야 해서 changeset이 필요한데, `delete_user`는 검증할 게 없어서 changeset 없이 `Repo.delete`만 쓰면 된다는 걸 알게 됐다.\\
`create_user`/`update_user` need a changeset for validation, but `delete_user` doesn't need any validation, so it can just call `Repo.delete` directly.

## 2. 상태 코드부터 순서까지, 자잘한 실수들

`update`에서 기존 유저를 먼저 조회해야 하는 걸 깜빡해서 없는 변수를 쓰다 에러를 봤고, `:update`라는 상태 코드도 없어서 `:ok`로 고쳤다. 새로 만들면 201, 고치면 200이라는 것도 이번에 정리됐다.\\
I forgot to fetch the existing user first in `update` and hit an error using a variable that didn't exist yet, and fixed a made-up `:update` status to `:ok`. I also finally got clear on why creating returns 201 and updating returns 200.

## 3. 지운 걸 왜 다시 보여주지

삭제 성공 후 데이터를 다시 보여주려다가 그럴 필요가 없다는 걸 깨닫고 204 No Content로 바꿨다. `send_resp`를 파이프 안에서 쓸 때 `conn`을 중복으로 넘겨서 에러가 났던 것도 고쳤다.\\
I was about to render the deleted data back after a successful delete, then realized there was no need to, and switched to 204 No Content. I also fixed an error from passing `conn` twice while piping into `send_resp`.

## 4. 파일 위치보다 모듈 이름

JSON 뷰 파일을 엉뚱한 폴더에 둬도 동작하길래, Phoenix는 파일 위치가 아니라 모듈 이름만 본다는 걸 확인했다. 컨트롤러 params는 문자열 키, JSON 뷰 assigns는 atom 키라는 것도 헷갈리다가 다시 배웠다.\\
I found that Phoenix only cares about the module name, not the file's location, after a JSON view file worked even in a random folder. I also relearned that controller params use string keys while JSON view assigns use atom keys.

## 5. 느낀 점

막힐 때마다 이미 되는 코드(`show`)랑 비교하는 게 제일 빨랐다.\\
Comparing against code that already worked (`show`) was the fastest way through every roadblock.
