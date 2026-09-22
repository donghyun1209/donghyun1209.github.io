`Users` 컨텍스트 모듈 작성 — `create_user/1`/`update_user/2`/`delete_user/1`은 직접 작성 `%User{}`가 "id: nil인, 아직 저장 안 된 구조체"라는 걸 확인 → create/update는 changeset이 값 검증 때문에 필요하지만 delete는 검증이 필요 없어 `Repo.delete(user)`를 changeset 없이 바로 쓴다는 것까지 스스로 구분해냄


 라우터 `/api` scope + USER 5개 라우트 (`post/put/delete/get`) 직접 작성


`UserController` 생성, `index`(예시 제공)/`show`(직접 작성) "라우터 `:userID`를 `:id`로 바꾸면?" - FunctionClauseError. 


컨트롤러 `update` — `get_user!(id)`로 기존 user를 먼저 조회한 뒤 `Users.update_user(user, params)` 호출, `case`/`{:ok,_}`/`{:error,_}` 분기 처음엔 `get_user!` 호출 없이 존재하지 않는 `user` 변수를 바로 써서 `undefined variable "user"` 컴파일 에러를 봄 → `show`와 비교해서 스스로 고침. `end` 짝 개수 세다가 문법 에러도 스스로 해결. `put_status(:update)` → `:updated`도 유효하지 않은 status atom이다. `:ok`로 정정. `create`는 없던 리소스를 새로 만드니 201, `update`는 이미 있던 리소스를 고치는 거라 200이라는 걸 POST/PUT 차이부터 시작해 스스로 도출함


컨트롤러 `delete` — `get_user!(id)`로 조회 후 `Users.delete_user(user)`, 성공 시 `send_resp(conn, 204, "")`, 실패 시 `{:error, changeset}` 분기 처음엔 삭제된 데이터를 `render(:show, ...)`로 보여주려다 스스로 "지운 걸 왜 다시 보여주지?"라는 모순을 알아채고 204 No Content로 방향 전환. `put_status(:ok)` 다음에 `send_resp(conn, 204, "")`를 이어붙이면 실제로 죽은 코드(`put_status(:ok)`)라는 걸 깨닫고 제거함. `send_resp(conn, 204, "")`를 파이프 안에서 쓸 때 `conn`을 중복으로 또 넘겨 `send_resp/4` undefined 에러를 봄 → `render`가 파이프에서 `conn` 안 넘기는 것과 비교해서 스스로 고침. unused variable 경고 정리 중 `Users.delete_user(user)`의 `user`(46번 줄, 실제 사용 중)와 `{:ok, user}`의 `user`(48번 줄, case 분기 안에서 미사용)를 헷갈려 엉뚱한 줄을 `_user`로 바꿨다가, 컴파일 경고가 가리키는 정확한 줄 번호를 다시 대조해서 스스로 정정함


`UserJSON` 뷰 (`lib/time_manager_web/controllers/user_json.ex`) 작성 — `data/1` 헬퍼는 예시로 받고, `index`/`show`/`error`는 직접 작성 시도 Phoenix가 `render(conn, :show, user: user)`를 호출하면 실제로 어느 모듈/함수가 실행되는지 예측하는 문제에서, 파일을 controllers 폴더가 아닌 엉뚱한 곳에 둬도 동작하는지 실험 → `elixirc_paths`가 `lib` 전체라 파일 위치가 아니라 `defmodule` 모듈 이름만 본다는 것을 직접 확인함. 이어서 `data(%User{})`처럼 함수 호출을 함수 머리에 쓰려다 "머리에는 패턴만 가능하고 함수 호출은 안 된다"는 규칙을 배움. `%{"userID" => id}`(컨트롤러 params, 문자열 키)와 `%{user: user}`(JSON 뷰 assigns, atom 키)를 헷갈리다가, atom과 문자열은 다른 값이라 키가 안 맞으면 패턴 매칭 자체가 실패한다는 걸 정정받고 이해함. `data/1`은 `%User{}` 구조체 하나만 받는데 `users`는 리스트라서 `Enum.map(users, &data/1)`으로 원소마다 호출해야 한다는 걸 스스로 설명함