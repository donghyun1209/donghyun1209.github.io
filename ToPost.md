# TimeManager — 진행 기록

작업마다 한 줄로 남긴다: `- [ ] 작업명 — 이해 확인한 핵심 포인트 — 완료 여부`. 할 일 목록은 `Step's_for_me.md` 참고.

---

## 진행 기록 - 1 (theme01: USER API, Phoenix)

- [x] `mix phx.new` 플래그 이해 — 설명+예측 — 통과 (`--no-html`은 뷰만 빼고 JS/CSS는 남김)
- [x] `users` 스키마 + 마이그레이션 — 설명 — 통과 (changeset은 우회 가능, migration `null: false`는 DB 레벨이라 우회 불가 → 둘 다 필요)
- [x] `mix ecto.migrate` + `psql \d users` — 예측+설명 — 통과 (NULL INSERT 시 DB 자체 제약 에러 확인)
- [x] `Users` 컨텍스트 — 직접수정+설명 — 통과 (`%User{}` = 저장 전 구조체, create/update는 changeset 필요, delete는 불필요)
- [x] 3-1. 라우터 `/api` scope + USER 5개 라우트 — 직접수정 — 통과 (`:browser` vs `:api` 파이프라인 구분, `mix phx.routes`로 검증)
- [x] 3-2. 컨트롤러 `index`/`show` — 직접수정+설명+예측 — 통과 (`:userID`를 `:id`로 바꾸면 FunctionClauseError)
- [x] 3-3. 컨트롤러 `create` — 직접수정+설명 — 통과 (`{:ok,_}`/`{:error,_}`는 Elixir 관례, 422 = `:unprocessable_entity`, `_` 접두사는 "사용 여부" 기준)
- [x] 3-4. 컨트롤러 `update`/`delete` — 직접수정+예측+설명 — 통과 (create 201 / update 200 / delete 204, `send_resp`는 파이프에서 `conn` 중복 전달 금지)
- [x] 3-5. `UserJSON` 뷰 — 예측+직접수정+설명 — 통과 (모듈 이름으로 찾음, 함수 머리엔 패턴만, atom 키 ≠ 문자열 키, `Enum.map(users, &data/1)`) — `traverse_errors`는 설명 없이 받음, 필요 시 다시 요청
- [x] 1-A. 경로명 `/workingtimes` → `/workingtime` (router · `api.js` · 테스트 `~p`) — 예측+직접수정+설명 — 통과 (`mix test` 26 passed, `:id`는 문자열 `"7"`로 params에 들어옴) — 브랜치 `Donghyun-workingtime-path`, 웹 실제 호출은 1-B/2-B에서 확인
- [x] 2-A. `AddTime.vue` → `WorkingTime.vue` + 라우트 파라미터 `:userid`/`:workingtimeid` — 예측+직접수정 — 통과 (`npm run build` 성공, 없는 param은 에러 없이 `undefined` → 이름 불일치는 조용히 망가짐) — PR #2

---

## 진행 기록 - 2 (theme02: Vue UI)

- [x] `main.js` 스타일 import — 직접수정 — 완료 (`style.css` → `styles.css`, 실제 파일명과 불일치)
- [x] Vite proxy — 예측 — 통과 1/2 (브라우저는 5173하고만 통신, proxy는 dev 서버 전용이라 build엔 없음)
- [x] `vite: command not found` — 완료 (vite는 프로젝트 로컬 패키지 → `npm install`)
- [x] A-1. 사이드바 로고 — 예측+설명+변형 — 통과 1.5/2 (없는 CSS 규칙은 에러 없이 무시, `grid-template-columns` 첫 열 = 고정 폭)
- [x] A-2. User 자리 — 예측+설명 — 통과 (`gap` = 자식 사이, `padding` = 안쪽 여백)
- [x] A-2+. userId 표시 — 예측+설명 — 통과 1/2 (`ref`는 `.value`로 변경, 그냥 변수는 Vue가 변화를 모름)
- [x] A-3. 사이드바 메뉴 — 예측+설명 — 통과 (`:to` = JS 식, 백틱만 `${}` 치환, ref라 링크 자동 갱신)
- [x] B-1. 탑바 제목 — 설명 — 통과 2.5/3 (`split('/')` 첫 조각으로 제목, computed = 반응형 + 캐시)
- [x] C-1. `WorkingTimes.vue` + 라우트 — 예측+직접수정+설명 — 통과 1/2 (경로 앞 `/` 누락 → 하얀 화면, `:userID` 대소문자 불일치 → `undefined`)
- [x] C-2. `getWorkingTimes()` — 예측+직접수정+설명+변형 — 통과 1/3 (`await` 없으면 Promise가 들어감, `onMounted(fn())` ≠ `onMounted(fn)`, watch로 userId 변경 대응)
- [x] C-3. 목록 테이블 — 예측+직접수정 — 통과 (`v-for` + `:key`, 모르는 태그(`<thread>`)는 에러 없이 무시)
- [x] C-4a. 총 근무 시간 — 예측+직접수정+변형 — 통과 (`reduce`에 시작값 `0` 없으면 객체+숫자 = 문자열)
- [x] C-4b. 기간 필터 — 직접수정+설명 — 통과 0/2 (`:value` 단방향 vs `v-model` 양방향, "바뀌면 실행" = `watch`)
- [x] C-5. 각 행 Edit 링크 — 예측+직접수정+설명 — 통과 1/3 (`entry.id` = DB id, `router-link-active`는 등록된 라우트 기준)
- [x] E-1. 사이드바 → `Sidebar.vue` — 예측+직접수정+변형 — 통과 (변수는 컴포넌트마다 따로, 템플릿이 쓰는 변수도 같이 옮김)
- [x] E-2/E-3. 탑바 → `Topbar.vue` — 예측+직접수정 — 통과 (라우터는 앱에 하나라 `useRoute()`는 어디서나 같음, 태그 빠뜨리면 에러 없이 사라짐)
- [x] D-1/D-2. 백엔드 검증 — curl 확인 — 통과 (팀원이 이미 반영, 둘 다 422, FK 없으면 `assoc_constraint`는 못 막음)
- [x] 3. 검증 (WorkingTimes) — 완료 (조회·빈 목록, 경로/data/method PDF 일치, 날짜 형식) — 팀원 컴포넌트는 병합 후 재검증
- [x] C-6. Timesheet 헤더 디자인 — 예측+직접수정+변형 — 통과 1/3 (flex는 바로 아래 자식만 배치, `margin-top`은 아래 요소도 밀어냄, `space-between`은 자식 1개면 무효)
- 메모: `App.vue`의 `ref('1')`은 User 컴포넌트 연결 시 교체 (User → App 전달 방식은 User 담당과 합의)
- 메모: 미커밋 — `package-lock.json`(커밋 대상), `Project/deps/`(커밋 금지, 확인 필요)

---

## 진행 기록 - 3 (theme03: DevOps)
---

## 진행 기록 - 4 (Skills & Tasks API, `my_work(small).md`)

- [x] 1. `skills` migration — 예측+직접수정 — 통과 1/2 (`null: false` 빼면 NULL도 저장됨 = 규칙이 사라짐, `unique_index` 중복은 DB가 거부 → schema `unique_constraint`로 422 변환) — 브랜치 `Donghyun-skills-tasks-api`, migrate는 3단계 후
- [x] 2. `tasks` migration — 예측+직접수정 — 통과 1/2 (설계 확정: task당 skill 하나(`skill_id` `:restrict`) + user_id nullable. `:restrict` = 부모 삭제가 거부됨(자식 삭제 아님), `:nilify_all` = 부모 삭제 시 NULL로, `null: false`와 같이 쓰면 부모 삭제가 에러, `type :x`는 함수 호출로 읽혀 파싱은 통과·migrate에서 실패)
- [x] 3. `user_skills` migration + `mix ecto.migrate` — 예측+직접수정 — 통과 0/1 (복합 PK = (user_id, skill_id) 쌍이 이름표라 같은 쌍 재삽입은 거부, `:x` atom vs `x:` keyword 키) — skills·tasks·user_skills 생성 확인
- [x] 4. `Skill` schema + `Skills` context — 예측+직접수정 — 통과 2/3 (`cast` 허용 목록 밖 키는 조용히 버려짐 = mass assignment 방어, `get!`는 없으면 raise → Phoenix가 404로, changeset invalid면 `Repo.insert`가 DB에 안 가고 `{:error, changeset}`)
- [x] 5~11. Skills·Tasks 컨트롤러·JSON·라우트 16개 + 배정 규칙 — **Claude가 작성** (학습 흐름 생략, 사용자 요청) — `mix test` 48 passed (신규 22). 규칙: 스킬 없으면 422, 이미 배정 409, task 쓰는 스킬 삭제 422, user_id는 create/update로 못 바꿈 — 커밋은 사용자가 직접

## 진행 기록 - 5 (theme04 Change resistance, `my_work(small).md`)

- [x] 1. 메시지 읽기 — 26건을 누가/불만 한 줄 지도로 정리, 주제별 7그룹 — 결과물은 표 1개 + 소통물 3~4개 (26개 다 만드는 게 아님)
- [x] 2. 이해관계자 표 `docs/stakeholders.md` — 예측+직접작성 — 통과 1/2 (Comm만인 줄 2개 맞음. Office employees의 핵심은 규칙보다 "감시 불신" → 투명성 이메일. `Comm + Feature` = 설명으로 풀리는 걱정 + 앱에 없는 기능 요청이 섞임 → Feature는 소통물에서 약속하면 안 됨) — 브랜치 `Donghyun-theme04-docs`
- [x] 3. 숨은 요구 `docs/stakeholders.md` "Underlying needs" — 예측+직접작성 — 통과 2/2 (Office employees의 Trust는 정보 부족이라 소통물만으로 해결 가능, Fairness·Accuracy는 기능이 있어야 함. Union의 Protection은 안내문이 아니라 실제 교육·기기 지원이 있어야 채워짐 → 소통물은 실제 조치를 알리는 것)
- [x] 5. 소통 계획 `docs/communication-plan.md` — 예측+직접작성 — 통과 (소통물 4개: 투명성 이메일 · 튜토리얼+FAQ · 현장직 전단 · 매니저 가이드. 수단은 받는 사람이 실제로 볼 수 있는 곳. Must not say = 없는 기능 약속 금지. 이메일은 사용법이 아니라 약속이라 이미지 불필요) — 순서 변경: 글 먼저(6), 캡처는 UI 완성 후(4). Stitch 시안은 실제와 달라질 수 있어 안 씀
- [x] 6-1. 투명성 이메일 `docs/communication/01-transparency-email.md` — 예측+직접작성 — 통과 2/2 (걱정 인정 → 사실 → 누가 보나 → 다음 할 일 순서. 기록 목록은 실제 DB 기준(clocks · workingtime · 프로필 · 스킬·태스크). 매니저 "own team only"는 5-7 권한 전엔 거짓 → Soon. 문의처에 노조 = 경영진을 못 믿는 사람의 안전한 창구. 마크다운은 빈 줄이 문단 구분)
- [x] 6-2. 규칙 FAQ `docs/communication/02-rules-faq.md` — **Claude가 작성** (사용자 요청) — 확인된 규칙만(26번: 야간 ×1.5, 주 40h 초과 ×2, 대체휴가), 점심·이동시간은 `[HR to confirm]`. 튜토리얼은 UI 의존이라 UI 완성 후로 미룸
- [x] 4-1 · 4-3 · 6-1~6-3 main 병합 (PR #4)
- [x] 6-3. 현장직 전단 `docs/communication/03-leaflet.md` — **Claude가 작성** (사용자 요청) — 3가지 메시지: 휴대폰 출근(Soon) · 야간 ×1.5 별도 기록(Soon) · 준비될 때까지 바뀌는 것 없음. 출시일은 안 씀. 인쇄해서 현장에 게시

## 진행 기록 - 6 (theme05 Login UI, `my_work(small).md`)

- [x] 1. `main`에서 `Donghyun-login-ui` 브랜치 — 예측 — 통과 1/1 (브랜치 = 커밋을 가리키는 이름표라 새 브랜치도 main 커밋이 보임. `pull` 빼먹으면 옛날 main에서 출발 → 이번엔 PR #4가 빠졌을 것)
- [x] 2. `api.js`에 `login(identifier, password)` (맨 끝) — 예측+직접수정 — 통과 1.5/2 (POST 맞음, 바디는 키 포함 `{"identifier":"kim","password":"1234"}` — 줄임 표기는 변수 이름이 키. 반환값은 Promise라 `await` 필요. 파일 끝 줄바꿈 없으면 다음 추가 시 conflict 원인)
- [x] 3. `stores/auth.js` (`currentUser` · `csrfToken` · `setSession`) — 예측+직접수정 — 통과 1/2 (Sidebar 링크는 `activeUserId`만 읽어서 `/workingTimes/3` — username·token은 안 씀. 함수 이름 오타 `setActiveUser`는 아무도 import 안 해서 build 통과 → build 통과 ≠ 맞음. 별도 파일에서 export한 ref는 앱 전체가 공유)
- [x] 4. `Login.vue` (폼 · `login()` · `setSession` · 401 메시지) — 예측+직접수정 — 통과 1/2 (401이면 throw → `try` 나머지 건너뛰고 `catch`, `finally`로 버튼 복구. `v-model.trim`이 앞뒤 공백 제거 — 비밀번호엔 안 붙임. 가짜 fetch로 mount 테스트 통과) — 라우트 없어서 화면 확인은 5단계 후
- [x] 5. `/login` 라우트 + 로그인 화면에서 Sidebar·Topbar 숨기기 (`App.vue` `v-if`/`v-else`) + `Login.vue` 가운데 정렬(`<style scoped>` flex) — 예측+직접수정 — 통과 2/2 (로그인 화면은 위쪽 `RouterView`가 그림, `v-else`는 바로 앞 형제의 `v-if`가 거짓일 때만. `.page-grid` 12칸 중 `.form-panel` 5칸이라 왼쪽에 붙었음 → 공용 `styles.css`는 안 건드리고 scoped로. `min(400px, 100%)` = 좁은 화면에선 부모 폭(padding 16px 뺀 폭)에 맞춤)
- [x] 6. 로그인 성공 → 역할별 이동 (`useRouter` · `router.push`, 가정: `user.role` 소문자, admin·manager → `/`, employee → `/clock/:id`) — 예측+직접수정 — 통과 1/2 (`"Employee" === "employee"`는 에러가 아니라 `false` → 직원이 관리자 화면으로 조용히 감. 앞 `/` 빠진 `clock/7`은 상대 경로라 `/workingTime/3`에서 호출하면 `/workingTime/clock/7` — build 통과 ≠ 맞음. `setSession` 먼저 = 이동한 화면이 새 id를 읽게. Sidebar는 `computed`라 나중에 바뀌어도 따라가지만 마운트 때 한 번만 읽는 코드는 옛 값을 씀) — 7단계(실제 로그인 검증)는 Jay API 나온 뒤로
- [x] 7-1. PR #6(jay-users) conflict 해결 — `router.ex` 양쪽 라우트 다 남김. `HEAD` = merge를 실행한 브랜치. 표시만 지우고 `git add` 안 하면 아직 미해결(`UU`). 첫 커밋에 router.ex만 들어가 main 파일 26개 빠짐 → push 전이라 `git add` + `commit --amend`로 복구. amend = 새 커밋(해시 변경)이라 push 전에만 — 완료
- [x] 7-2. `api.js` `login()` → `/auth/sign_in`, body `{ login: identifier, password }` — 예측+직접수정 — 통과 (`{ identifier }`는 키 이름 = 변수 이름이라 처음엔 키가 안 바뀜. 인자 이름은 그대로라 `Login.vue` 수정 불필요) — 실제 로그인 테스트는 role 수정 후
- [x] 7-3. `Login.vue:18` → `result.user.role_id === 2` (Jay 응답엔 `role` 없이 `role_id`만. seed 순서상 User=2, 번호는 보장 안 됨 → 문제 생기면 Jay에게 role 이름 요청) — 예측+직접수정 — 통과 (`"2" === 2`는 에러 없이 `false` → 직원이 `/`로 조용히 감) — 실제 로그인 테스트 남음
- [x] 7-4. 실제 로그인 검증 — Jay migration 수정(빈 중복 `update_users_table` 삭제, 이미 있는 `users_email_index` 중복 생성 줄 삭제) → `ecto.reset`/`migrate`/seed → `emp1`(role 2) `/clock/1`, `mgr1`(role 1) `/` — 예측 통과 2/2 (Postgres는 실패한 migration을 통째로 취소 → 고치고 `migrate`만 다시. `sign_up`은 항상 User라 Manager는 DB에서 role 변경, 안 바꾸면 `/clock/2`) — 완료

## 진행 기록 - 7 (theme06 Mobile, `my_work(small).md`)

- [x] 1. `main`에서 `Donghyun-mobile` 브랜치 — 완료
- [x] 2. 6-1 스택 결정 — Capacitor + Ionic Vue (아는 Vue 그대로 · 남은 기간 대비 학습 최소 · 탭 바/오프라인/진동·알림 공식 지원 · 웹뷰라 쿠키 인증이 웹과 비슷할 가능성). 타겟은 Android 패드 실기기, iOS는 시뮬레이터 확인만 — 완료
- (10-02 재개: 패드 없음 → 3단계 패드 실행은 보류, 4단계 탭 바부터 브라우저로 진행)
- [x] 3. 6-2 세팅 — Android Studio + SDK(cmdline-tools 포함), `ionic start mobile tabs --type=vue --capacitor`, `appId` 변경, `cap add android` 완료 (`export default config` 빠지면 Capacitor가 설정을 못 읽어 "Missing appId") — 10-02 패드(SM-T733) 실행 성공 (팝업이 안 뜬 건 adb 데몬이 꺼져 있어서 → `adb devices`로 켜니 `unauthorized` → 허용 후 `device`) — 완료
- [ ] 4-1. `TabsPage.vue` 탭 4개(Clock · Times · Team · Profile) — 코드 제시만, 미입력. **10-02 보류: 팀 결정으로 웹 완성 먼저**

## 진행 기록 - 8 (기존 웹 화면 API 바인딩, 브랜치 `Donghyun-ui-binding`)

모바일(진행 기록 - 7)은 3단계(패드 실행)에서 잠시 멈춤 → 바인딩 후 4단계(탭 바)부터 재개

- [x] 1. `main`에서 `Donghyun-ui-binding` 브랜치 (`clients/mobile/` · `deps/` untracked 남아 있음 → `git add .` 금지) — 완료
- [x] 2. `api.js`에 `signUp` · `signOut` · `listRoles` — 예측+직접수정 — 통과 1.5/2 (`listRoles()`는 `request()`가 `data`를 벗겨서 배열 `[{id, label}]` 반환 — `roles.data`는 에러 없이 `undefined`. 키는 `label`. `body: { user }`는 줄임 표기라 `{"user": {...}}`로 감싸져 서버가 password를 못 찾음 → `body: user`. body 없으면 `undefined` → fetch가 바디 없이 보냄)
- 10-01 Jay와 합의: 디자인은 나중, **task 바인딩 먼저 → user API에 skill 추가**. `user_controller.ex` 버그 6개(username 없음 · role preload 없음 · `"id"` vs `:user_id` · `get_user` 반환형 · create 500 · 검색 무시)는 **Jay 담당**. Register · 로그아웃 · 실제 역할은 보류
- [x] 3. `api.js`에 tasks 함수 (`listTasks` · `createTask` · `deleteTask` · `toggleTaskStatus` · `assignTaskToUser`) + `listSkills` — 예측+직접수정 — 통과 (DELETE는 204 → `request()`가 `null` 반환. `createTask`는 서버가 `"task"` 키를 꺼내서 `{ task }`로 감쌈 — `signUp`과 반대)
- [x] 4-1. `Tasks.vue` (목록 · 생성 · 상태 토글 · 삭제) — **Claude가 작성** (사용자 요청) — `run(() => ...)` = 나중에 실행할 함수 전달, `?.` = 없으면 에러 대신 undefined. 성공 후 `load()`로 표 새로 고침
- [x] 4-2. `/tasks` 라우트 + Sidebar `Tasks` 링크 — **Claude가 작성** (사용자 요청)
- [x] 4-3. 사용자 배정 버튼 (`activeUserId`에게, 이미 배정이면 `#id` 표시. 스킬 없으면 서버 422 메시지 표시) — **Claude가 작성** (사용자 요청) — build 통과. `App.spec.js` 실패는 main에서도 동일(기존 문제)
- [x] 4-5. Skills 페이지 (`Skills.vue` 목록 · 생성 · 삭제, `/skills` 라우트, Sidebar `Skills`, `api.js` `createSkill` · `deleteSkill`) — **Claude가 작성** (사용자 요청) — build 통과. 태스크가 쓰는 스킬 삭제는 DB `on_delete: :restrict`라 서버 500 → 화면엔 "Request failed"
- [x] 4-4. 브라우저 확인 (사용자) — Tasks · Skills — 완료 (10-04 확인)
- [x] 커밋 + push (사용자, 10-02 `5395548`) — 5개 파일: `api.js` · `Tasks.vue` · `Skills.vue` · `router/index.js` · `Sidebar.vue` (`git add .` 금지)
- [ ] 5. user API에 skill 추가 (Jay의 `user_controller.ex` 수정 후 — 같은 파일) — **대기**

→ **10-02부터 모바일(진행 기록 - 7) 4단계 탭 바로 복귀**

## 진행 기록 - 9 (웹 인증 UI: 로그아웃 · 회원가입, 브랜치 `Donghyun-ui-binding`)

- [x] 0. `git merge origin/jay-users` (`49f6523`, 충돌 없음) — 완료
- [x] 1. `api.js` `X-CSRF-Token` 헤더(조건부 spread) + `readError()`가 `error`(단수) 읽기 — 예측 반쯤 (401은 맞음, 화면엔 `Unauthorized`) · 이해 확인 0.5/2 (로그인 전 `csrfToken.value`는 `null` → `...{}` → 헤더 없음. `return` 있는 `if`는 위에서 먼저 걸리는 쪽이 이김 → `error` 체크를 아래로 내리면 "Request failed") — build 통과, 실제 요청은 서버 꺼져 있어 미확인
- [x] 2. `Login.vue:18` `role_id === 2` → `role === "user"` — 예측 1/2 (`"User" === "user"`는 `false` → `/`로 감. 서버 `normalize_role`이 소문자로 보내서 실제론 안전) · 처음엔 `role_id === "user"`로 침 → `undefined === "user"` 항상 `false`, build 통과 ≠ 맞음 → 수정 · 이해 확인 통과 (`setSession` 먼저 = 이동한 화면이 새 유저를 읽음) — 완료
- [x] 3-1. `stores/auth.js` `clearSession()` (`currentUser` · `csrfToken` → `null`) — 예측 통과 (clear 후 `X-CSRF-Token` 안 보냄) · 이해 확인 통과 (`api.js`와 같은 ref 공유 + `request()`가 호출될 때마다 `.value`를 새로 읽음) — 완료
- [x] 3-2. Sidebar `Log out` 버튼 (`signOut()` → `catch {}` → `clearSession()` → `/login`) — build 통과 · 예측 반쯤 (새로고침 후엔 서버 `403 Invalid CSRF`가 맞지만 `catch`가 삼켜서 화면은 `/login`으로 감. **쿠키(JWT)는 안 지워짐** → 후속: 새로고침 때 `GET /auth/me`로 `csrf_token` 복구) — 완료
- [x] 4-1. `Register.vue` (username · first_name · last_name · email · password → `signUp(form)` → `/login`) — 예측 거의 (`readError`가 필드명 붙여 `email has already been taken`) · 오타 `userRouter`(import가 `undefined` → 페이지 열 때 에러, 아직 import 안 돼서 build 통과) · 비밀번호 `.trim` 제거 (Login은 trim 안 함 → 앞뒤 공백 있으면 로그인 불가) — 완료
- [x] 4-2. `/register` 라우트 + Login에 링크 — 예측 거의 (라우트 없으면 본문 빈칸 + 콘솔 `No match found`, Sidebar는 보임) — 완료
- [x] 4-3. `App.vue` `/register`도 Sidebar 없이 — 예측 통과 (`&&`면 경로가 동시에 둘일 수 없어 항상 `false` → Login 폼 + Sidebar 둘 다 보임) — 완료
- [x] 5. 브라우저 확인 (서버 필요) → 커밋 + push — 완료 (10-04 확인)
- [x] 4-4. 링크·Log out 스타일 (`styles.css` `.auth-link`, Sidebar 버튼 `class="button"`) — 예측 통과 (`button primary` = 하늘색) — build 통과, 브라우저 확인(사용자) — 완료

## 진행 기록 - 10 (라우터 가드 + 세션 복구, 5-11, 브랜치 `Donghyun-ui-binding`)

- [x] 1. `api.js` `getMe()` (`GET /auth/me`, method 없으면 GET → CSRF 검사 없음) — 예측 통과 (쿠키 없으면 401 → `request()`가 throw) — 완료
- [x] 2. `stores/auth.js` `restoreSession()` (`getMe()` → `setSession`, 실패 → `clearSession`) — 예측 틀림 (새로고침하면 `ref`가 통째로 초기화 → "전 정보"는 없음. `catch` 없으면 401이 가드까지 올라가 이동 취소 → 빈 화면) · 키 오타 2번 (`csrfToken` → `csrf_Token` → `csrf_token`. 키는 대소문자·밑줄까지 서버와 같아야 함, 틀리면 `undefined` → 헤더 빠짐 → 403, build 통과 ≠ 맞음) — 완료
- [x] 3. 라우트 `meta` (`public: true` = 로그인/회원가입, `role: "manager"` = `/` · `/users` · `/workingTime` 2개 · `/skills`, 표시 없음 = 로그인 필요) — 예측 통과 (가드 없으면 `meta`는 메모일 뿐 → 들어가짐) · `component` 키 중복 제거 (같은 키 두 번 = 에러 없이 뒤가 덮어씀) · 이해 확인 반쯤 (`/tasks`는 "누구나"가 아니라 **로그인한** 누구나) — 완료
- [x] 4. `router.beforeEach` 가드 (첫 이동 때 `restoreSession()` 한 번 → 공개 통과 → 비로그인 `/login` → 역할 부족 `/clock/:id`, `roleLevels`로 숫자 비교) — 예측 모름 → 설명 (`user`가 `/skills` 직접 입력 = 새로고침 → 복구 1번 → `/clock/id`로 redirect, redirect도 가드를 다시 타지만 `restored`라 복구 생략) · 기존 `export default createRouter(...)`를 안 지워서 `export default` 2개 → 에디터 빨간 줄 → 삭제 — 완료
- [x] 5-1. `stores/auth.js` `hasRole(minRole)` (`>=` 비교, 비로그인은 `undefined >= n` → `false`) + 라우터가 사용 — 예측 틀림 (`admin`(3) >= `manager`(2) → `true`, 같은지가 아니라 이상인지) · 고친 곳 3개 (옛 `roleLevels` 미삭제 · 공개 검사 줄을 잘못 바꿈 · 실제 바꿀 역할 검사 줄은 그대로) — 완료
- [x] 5-2. Sidebar `navItems`에 `role` + `.filter((item) => !item.role || hasRole(item.role))` — 예측 통과 (`user` → Timesheet · Live clock · Insights · Tasks 4개) — 완료
- [x] 6. 브라우저 확인 → 커밋 + push (4파일: `api.js` · `stores/auth.js` · `router/index.js` · `Sidebar.vue`)  — 완료 (10-04 확인)
- [x] 7. `Tasks.vue` 역할별 (user: `/skills` 요청 안 함 · 생성 폼·Delete 숨김 · 에러는 표 위로) — **Claude가 작성** (사용자 요청). 사용자가 일부 친 상태에 겹쳐 중복 생김 → 정리 — build 통과. user는 Skill 칸이 `-` (백엔드 `/skills`가 manager 전용)
- [x] 8. `Topbar.vue` `pageTitles`에 `tasks` · `skills` (없으면 `undefined || 'Dashboard'`) — 쉼표 빠짐 → `expected "," (13:2)` (에러 줄의 바로 위 줄 끝을 볼 것) · 예측 틀림 (`/register`는 `App.vue`가 app-shell을 안 그려서 Topbar 자체가 없음) — 완료
- [x] 9. 가드: 로그인한 사람이 `/login`·`/register`(뒤로가기·직접 입력) → 홈으로 (manager 이상 `/`, user `/clock/id`) — 예측 통과 (Log out은 `clearSession()`이 `push("/login")`보다 먼저라 가드 시점엔 `user`가 `null` → 통과) — build 통과, 커밋 남음

## 진행 기록 - 11 (프로필 화면 UI, 5-9, 브랜치 `Donghyun-ui-binding` — main `0b30156` fast-forward 후)

- 범위: UI만 (10-05 Jay: 수정·삭제 API는 Jay가 테스트 후 올림)
- [x] 1. `Profile.vue` 보기 (`currentUser`를 그대로 사용, `getMe()` 재호출 X. template에선 import한 `ref`도 `.value` 없이) — 예측 반쯤 (`v-if` 없으면 "마지막 user 정보"가 아니라 로그아웃 시 `null.username` → TypeError로 렌더 깨짐) · 키 오타 `firstname`/`lastname` → 에러 없이 빈칸 (`first_name`/`last_name`) — 완료
- [x] 2. `/profile` 라우트(`meta` 없음 = 로그인한 누구나) · Sidebar `Profile` · Topbar `profile` — 예측 통과 (비로그인 → 로그인 화면) · Sidebar `Skills` 줄 중복 → 삭제 (쉼표만 붙이면 될 걸 줄을 새로 침) — 브라우저 확인(사용자) — 완료
- [x] 3. 프로필 카드 디자인 (이니셜 아바타 `first_name[0]` · 이름 + `@username` · 역할 배지 · `info-grid` `repeat(auto-fit, minmax(160px, 1fr))`) — 예측 통과 (`EO`) · class 오타 3개 (`into-grid` · `info-valeue` · `avator`) → 에러 없이 스타일만 안 먹음 · 역할 배지 대비 낮음(초록 배경 + 연회색 글자) → `role-pill` (하늘색) — 예측 통과 (scoped는 `[data-v-…]`가 붙어 더 구체적 → `role-pill` 색이 이김). 변형: `status-pill` 빼면 둥근 모양·여백 사라짐 (색만 덮어쓰고 모양은 `status-pill`에서 옴) — 완료
- [x] 4. 수정 폼 UI (`reactive` 폼을 `currentUser`로 미리 채움 · 비밀번호 비워 두면 유지 · Role은 `:value` + `disabled` · Save 버튼 `disabled` · `@submit.prevent`만) — 예측 통과 (폼에 입력해도 위 카드 안 바뀜 — 만들 때 값 복사) · 비밀번호 `.trim` 제거 · `type="email"` 추가 · 안 쓰는 `.hint` 삭제 · 이해 확인 틀림 (`v-model`도 값은 보임 — `:value` + `@input`. 차이는 `disabled`가 빠지면 store의 `currentUser.role`이 바뀌어 화면이 manager용으로 열림) — 완료
- [x] 5. 계정 삭제 버튼 UI (`.danger-zone` · `button danger` `disabled`, 폼 밖) — 안내 문구는 사용자가 뺌 · 예측 통과 (`flex` 빼면 `div`가 block이라 버튼이 다음 줄, 둘 다 왼쪽 정렬) — 완료
- [x] 6. skills 표시 (10-05 Jay와 합의: 백엔드 Jay가 `user_response`에 `skills: [{id, name}]` → sign_up·sign_in·me 모두) — `currentUser.skills?.length` → 있으면 `status-pill` 목록, 없으면 `No skills yet` — 예측 통과 (API 전엔 `undefined` → `v-else`) · class 오타 `noskills` (CSS는 `.no-skills`) → 회색·여백 안 먹음 — build 통과
- [x] 7. 폼 정리 — 비밀번호 칸 삭제 (Jay: 비번 변경 API 안 만듦, `project.pdf`엔 비번 변경 명시 없음) · `form.password`도 같이 삭제 (남기면 저장 때 빈 값이 같이 감) · Role 칸 삭제 (헤더 배지로 이미 보임) → 폼 3칸 — build 통과 — 완료

### 연결 (10-05, Jay `jay-users` `0a50f0f` merge → `0dffbe1`, 충돌 없음 · `mix compile` 통과)

- [x] 8. `api.js` `updateUser` `body: { user }` → `body: user` (Jay `update`는 바깥 키를 읽음. 감싸면 200인데 아무것도 안 바뀜) — 예측 반쯤 (DB 안 바뀜은 맞음, 화면엔 에러 없이 성공처럼 보임) — 완료
- [x] 9. 저장 버튼 `save()` (`updateUser(currentUser.value.id, form)` → `currentUser.value = { ...currentUser.value, ...user }` → 카드·Sidebar 즉시 반영, PUT 응답에 `skills` 포함) · `form-success` / `form-error` — 예측 통과 (중복 이메일 → 422 → `readError` → "email has already been taken") · 빠진 것 3개 (`ref` import → 화면 전체 깨짐 · `updateUser` import · `currentUser` 갱신 줄. build는 통과 — build는 이름이 실제로 있는지 안 봄) — build 통과
- [x] 10. 계정 삭제 `removeAccount()` (`window.confirm` → 취소면 `return` · `deleteUser` → `clearSession()` → `/login`, 실패 시만 `deleting` 복구) — 예측 생략 · `error.value = ""` 초기화 줄 빠짐 (사소: 전 에러가 잠깐 남음) — build 통과, 브라우저 확인 남음 (테스트용 새 계정으로)
- [x] 11. PR #9 — GitHub 충돌 (Jay PR #8이 먼저 main 병합, 공통 조상 2개라 GitHub만 충돌로 봄) → `git checkout Donghyun-ui-binding` → `git merge main` (자동 해결, `api/`는 main과 동일) → push → `CLEAN` → **10-05 병합 `aef4415`** — 완료

## 진행 기록 - 12 (theme04 04 튜토리얼 + 화면 캡처, 10-05)

- [x] `docs/communication/04-tutorial.md` — **Claude가 작성** (사용자 선택) — 직원 기준 8단계 (가입 · 로그인 · 출퇴근 · Timesheet · Insights · Tasks · Profile · 로그아웃). 버튼 문구는 코드와 일치. Timesheet `Edit`는 직원에겐 manager 화면으로 튕겨서 안내 안 함 ("매니저에게 요청") → 2-B에서 숨길 것. 출퇴근이 근무시간을 자동 생성하지 않으므로 약속 안 함
- [x] `docs/mockups/01~07-*.png` — **Claude가 캡처** (headless Chrome + CDP, 1440×900, Vue devtools 버튼 숨김). 튜토리얼 계정 `alex.martin` / `Tutorial2026!` (dev DB id 7) + 샘플 데이터 (skills 2 · 근무시간 5 · 출퇴근 5 · tasks 3) — 로컬 dev DB에만 있음
- 남은 것: `07-profile.png`는 skills가 "No skills yet" (`/auth/me`에 skills 없음, Jay 대기) → Jay 수정 후 다시 캡처. 커밋은 사용자

## 진행 기록 - 13 (2-B 병합 후 전체 재검증, 10-05)

- [x] 2-B — **Claude가 작업** (사용자 요청) — 브랜치 `Donghyun-verify`. Docker DB 볼륨 초기화 (옛 users가 `first_name` NOT NULL 마이그레이션을 막음, orphan `project-postgres-1`이 볼륨 점유) + 시드 + 테스트 계정 `mgr1`(manager) · `emp1`(user), 비번은 시드와 같음
- 고친 것: User.vue 추가 400 (`{user:{}}` → 평평한 바디 + 이름·비번 필드) · `GET /users?email=&username=` 무시 → 백엔드 필터 (`user_controller.ex`, Jay 영역이라 공유 필요) · role 열이 `demoRoleFor` 가짜값 → API `role` · 출퇴근 시각 UTC가 그대로 보임 → `api.js`에서 로컬 변환 · 데모 데이터 버튼 · 직원 Timesheet `Edit` 숨김
- 확인만: ClockManager · WorkingTime 폼(생성·조회·수정) · 필터 · ChartManager 정상. `App.spec.js`는 Vite 기본 테스트라 원래 실패 (손 안 댐)
- 10-06 PR #10으로 main 병합. 남은 것: 직원이 남의 clocks·workingtime 조회 가능 (5-7)

## 진행 기록 - 14 (모바일 이관 · 웹 마무리 계획, 10-07)

- [x] `Donghyun-mobile` push — 옛 main(`6d85ffe`) 기준이었지만 `clients/mobile/`만 바꿔서 main과 겹치는 파일 없음 → `git merge origin/main`(충돌 없음) → push `56351b8`. main에 남아 있던 untracked `clients/mobile/`이 브랜치 이동을 막아 잠깐 옮겼다가 되돌림 — 완료
- [x] 팀 방침 정리 — 팀 기능(5-8 · 5-13 · P-1) 안 함, 웹은 UI 완성으로 끝 → 2-C (디자인 4개 + 기존 API 화면 4개), theme04 캡처·튜토리얼은 2-C 후 다시. 목록은 `my_work(small).md` — 완료

## 진행 기록 - 15 (2-C 웹 UI 마무리, 10-07)

- [x] 1. Sidebar — **Claude가 넣음** (사용자 요청) — "My work"(`currentUser.id`) / "Team"(`activeUserId`, `hasRole('manager')`라 admin도 보임). "Selected user ID" 상자 삭제. `watch`는 URL id가 내 id면 `activeUserId`를 안 바꿈 (`String()` 비교: params는 문자열) · `styles.css`에 `.nav-list small` — build 통과, 브라우저 확인 남음
- [x] 2. Overview — **Claude가 넣음** — Manager demo · Report status(사용자 선택, 통계 카드와 중복) · Demo flow 삭제, 에러 줄만 남김. 카드가 세로였던 원인: 3칸 `.metrics-row` 안에서 `.home-grid .metric-panel { span 4 }` → 규칙 삭제. 예측(좁은 화면이면 세로) 맞음. `services/demoData.js`는 이제 안 쓰임 (안 지움) — build 통과, 브라우저 확인 남음
- [x] 3. Team (`User.vue`) — **Claude가 넣음** — "Team roster" 소개 패널 삭제. 남은 폼 2개(span 5씩)가 한 줄에 나란히 — build 통과
- [x] 4. Tasks — **Claude가 넣음** — `userName()`: 내 id면 `currentUser.first_name`, 아니면 `listUsers()`(manager만)에서 `first_name`, 못 찾으면 `#id`. 직원은 `GET /users` 권한이 없어서 남의 이름은 `#id`로 보임 (백엔드 안 건드림). "Assign to" 버튼도 이름 — build 통과
- [x] 1-보완. Sidebar 중복 active — **Claude가 넣음** — 원인: 로그인 시 `activeUserId` = 내 id → My work · Team의 Timesheet/Insights가 같은 주소. `hasSelection` + `perUser` 표시로 직원 고르기 전엔 Team의 Timesheet · Add time · Insights 숨김 — build 통과, 브라우저 확인 남음
- [x] 1-보완. Sidebar 순서 — **Claude가 넣음** — Team 묶음을 위로, My work를 아래로 — build 통과
- [x] 1-보완. Sidebar 선택한 사용자 이름 — 사용자가 script 입력, Claude가 template 위치(`<aside>` 밖 → Team `<nav>` 안) · CSS(`@media` 안 + `},` → 빈 선택자 build 에러) 수정. `getUser`(`GET /users/:id`)는 컨트롤러 `user_json`이 first_name·last_name 반환 (`user_json.ex` 모듈은 안 쓰이는 코드) — build 통과
- [x] 1-보완. Sidebar 직원 선택 상자 — **Claude가 넣음** — "Selected: 이름" → `<select>` (`listUsers()`, 나 제외). 고르면 `setActiveUserId` + 직원별 화면이면 `/<첫 경로>/<새 id>`로 이동. `routeUserId` computed로 뽑아 watch와 공유. `getUser`·`selectedName`·`.nav-selected` CSS 제거 (대체됨). 사이드바 목록은 매니저 판정 시 1번만 불러옴 (새 직원은 새로고침해야 보임) — build 통과, 브라우저 확인 남음
- [x] 1-보완. 사이드바 선택 상자 스타일 — **Claude가 넣음** — `.nav-list select`: `appearance: none` + SVG 화살표(오른쪽 11px), 폰트를 메뉴와 같게(0.82rem · 700) — build 통과
- [x] 5. 승급·강등 — 사용자 입력 — `api.js` `updateUserRole` (`body: { role }`, 바깥 키 `role`) · `User.vue` `changeRole()` (`run()` 안 → 403이면 아래 줄 건너뜀, 예측 통과) · role 칸 `hasRole('admin')`이면 `<select :value @change>` 아니면 `<span>` (manager는 403 날 상황을 아예 안 만듦). 실수: `value="employee"` → 서버 role은 `user`/`manager`/`admin`만 (422 + 빈 선택) → `value="user"` 수정. 이해 확인: `$event.target.value`는 고른 값 하나(option `value`, 라벨 아님) · manager 화면은 `v-else` 글자만 — 반쯤. 로컬 DB에 admin 없어서 시드 재실행 → admin id 8 · mgr1 비번 `Password123`으로 재설정(Claude, 사용자 요청). mgr1 강등→승급 브라우저 확인 — build 통과 — 완료
- [x] 6. 사용자에게 스킬 주기·빼기 — 1단계 사용자 입력 (`api.js` `addUserSkill` · `removeUserSkill`, body 없음, 중복은 422 예측 통과) · 2단계 사용자 입력 + Claude 보완 (`listSkills` import · `onMounted` 스킬 로드 · `giveSkill`/`takeSkill` 빠져 있었음) · 3단계 **Claude가 넣음** (사용자 요청) — Team 폼에서 직원 선택 시 Skills 줄: 가진 스킬 버튼(누르면 빼기) + `<select v-model>`(가진 건 `availableSkills`에서 제외) + Add skill. `GET /users`에 skills가 preload돼 있어서 `selectedSkills`는 `users`에서 computed — build 통과, 브라우저 확인 남음
- [x] 7. 스킬 이름 수정 — 1단계 사용자 입력 (`api.js` `updateSkill`, `body: { skill }` → `{"skill":{"name":…}}`, 예측은 한 겹 빠짐) · 2단계 script 사용자 입력 + Claude 오타 수정 (`startEdit(skill)` 인자 · `editiongId` · `saveEdit`에 `async` 빠짐) · template **Claude가 넣음** (사용자 요청) — 줄마다 Edit → 입력칸 + Save/Cancel (`<template v-if>`). 다른 줄 Edit 누르면 그 줄로 바뀜 (예측 맞음) — build 통과, 브라우저 확인 남음
- [x] 7-보완. 간격 정리 — **Claude가 넣음** (사용자 요청) — Team: 폼 2개가 span 5씩(10/12칸)이라 표보다 짧음 → `.users-grid .form-panel { span 6 }` (좁은 화면 `@media` 목록에도 추가, 안 넣으면 선택자가 더 구체적이라 모바일에서도 반반) · Skills: Edit/Delete 사이 간격 없음 → 기존 `.button-row`로 감쌈 — build 통과, 브라우저 확인 남음
- [x] 7-보완. Skills 버튼 오른쪽 정렬 — **Claude가 넣음** (사용자 요청) — 기존 `.row-actions`(Team 표에서 쓰던 것) 재사용 + `.row-actions .button-row { justify-content: flex-end }` (flex라서 `text-align: right`로는 안 움직임) — build 통과
- [x] 8. 태스크 수정 · 배정 해제 · 스킬 변경 — `PUT /tasks/:id`가 `skill_id`도 받아서 스킬 변경 API는 안 씀 (수정 줄에서 같이) · 1단계 사용자 입력 (`api.js` `updateTask` · `unassignTask`, 남의 id로 해제하면 404 예측 맞음) · 2단계 사용자 입력 (`editForm` reactive 복사본 — `task`에 바로 v-model하면 Cancel 해도 화면에 남음, 예측 맞음) · 3단계 Title·Skill·User 칸 사용자 입력 + Claude 수정 (`<section>` → `<select>`), 마지막 칸 **Claude가 넣음** (사용자 요청). 직원은 마지막 칸 빈칸 (예측 맞음) — build 통과, 브라우저 확인 남음
- [x] 8-보완. 태스크 배정을 표 안 선택 상자로 — **Claude가 넣음** (사용자 요청, import 삭제만 사용자). 사이드바 직원(`activeUserId`) 버튼 → 매니저: 그 스킬 가진 사람만 `<select>` (`usersWithSkill`, `GET /users`의 skills 사용, 백엔드가 스킬 없으면 `missing_skill`), 없으면 "No one has this skill" · 직원: "Assign to me"(`currentUser.id`, `GET /users` 권한 없음). 패널 방식도 검토했지만 클릭 수 때문에 표 안 방식 선택. 예측(배정 후 이름으로 바뀜) 맞음 — build 통과, 브라우저 확인 남음
- [x] 8-보완. Unassign 버튼 매니저만 — **Claude가 넣음** (사용자 요청) — 아예 삭제하면 재배정 불가 (`already_assigned`라 먼저 해제 필요) → `v-if="hasRole('manager')"`. 백엔드는 직원도 해제 가능 (7-6에서 다룰 것) — build 통과
- [x] 8-보완. Unassign 버튼 삭제 → 선택 상자로 — **Claude가 넣음** (사용자 요청) — 매니저: 배정된 태스크도 `<select :value="task.user_id">` (지금 담당자 선택됨), 첫 항목 "Unassign"/"Assign to…". `reassign()`: 담당자 있으면 해제 → 새 사람 있으면 배정 (백엔드 `already_assigned` 때문에 2번 요청, 배정만 실패하면 미배정 상태로 남음) · 직원: 이름 또는 "Assign to me" — build 통과, 브라우저 확인 남음
- [x] 8-보완. 선택 상자 화살표 통일 — **Claude가 넣음** (사용자 요청) — 표 안 select는 브라우저 기본 화살표(오른쪽 끝에 붙음). 사이드바·Team 스킬 상자에만 있던 SVG 화살표(오른쪽 11px)를 전역 `select` 규칙으로 옮기고 두 곳의 중복 제거 → New task · Tasks 표 · 차트 select도 같은 모양 — build 통과, 브라우저 확인 남음
- [x] 8-보완. 직원은 내 태스크 + 미배정만 — **Claude가 넣음** (사용자 요청) — 직원은 `GET /users` 권한이 없어 남의 담당자가 `#id`로 보였음 → `visibleTasks` computed로 화면에서만 거름 (API는 여전히 전부 반환, 7-6). 예측(스킬 있으면 남고 없으면 에러) 맞음. 직원 화면 Skill 칸 `-`는 백엔드 현재 설계 (`GET /skills` 매니저 전용 · `task_json`에 skill_id만 · `/auth/me`에 skills 없음) → Jay에게 공유할 것 — build 통과, 브라우저 확인 남음
- [x] 정리 — Jay 합의: 백엔드 최소 변경 2개(태스크 응답에 skill_name·user_name **추가**, `/auth/me`에 skills)는 내가 함. `my_work(small).md`를 끝난 것 / 내일 할 것(A 브라우저 확인 · B 백엔드 · C 웹 반영 · D Jay 공유 · E 커밋)으로 재정리 — 완료
- [x] A. 브라우저 확인 (6·7·8 + 보완) — 사용자가 직접 확인 — 완료
- [x] 웹 UI 커밋 · push — `6423c93` "Finish web Ui" (`Donghyun-ui-binding`, 사용자). PR은 B·C 후 — 완료
- [x] B-1. 태스크 응답에 `skill_name`·`user_name` 추가 — 사용자 입력 (`task_json.ex` 한 곳: `data/1` 안에서 `Repo.preload(task, [:skill, :user])` → 컨트롤러 6개 응답 전부 적용). `user_name`은 `first_name`만 (예측 맞음), 미배정은 `task.user && …`로 `null` (없으면 목록 전체 500). `mix compile` 통과 · `mix test` 2/48 (10-05와 동일, 기존 fixture·401 문제) · curl `GET /tasks` (mgr1) 이름 확인. 이해 확인: preload 결과를 `task =`로 다시 받는 이유(불변) 틀림 · data/1 한 곳 = 6개 응답 반쯤 — 완료
- [x] B-2. `/auth/me`에 내 skills 추가 — 사용자 입력 (`auth_controller.ex` `user_response`: `with_role` → `with_details` + `skills: Enum.map(..., fn skill -> %{id, name} end)`, `GET /users`와 같은 모양). 회원가입·로그인·me 3곳 공통. 예측(with_role이면 문제) 방향 맞음 → 정확히는 NotLoaded에 Enum.map → 500, 로그인 자체 실패. `mix compile` 통과 · curl alex.martin 로그인·`/auth/me`에 skills [Archives, Front desk]. 이해 확인: 구조체 그대로 → JSON 변환 에러 맞음 · 새 가입자 skills는 nil 아니라 `[]` 틀림 — 완료
- [x] C-1. `Tasks.vue` 서버 이름 사용 — 사용자 입력 (Skill 칸 `task.skill_name` · User 칸 `task.user_name`, `skillName()`·`userName()` 삭제, `usersWithSkill`은 매니저 선택 상자용으로 남김). 예측: 매니저 화면 그대로 맞음 · "직원 화면에 다른 직원 full name" 틀림 (`visibleTasks`가 내 것+미배정만, `user_name`은 first_name만) → 바뀌는 건 Skill 칸 `-` → 이름. 이해 확인: 직원은 Edit 선택 상자 못 봄 (Edit 버튼이 `hasRole(manager)` 안 → `editingId` 안 바뀜) 맞음 — build 통과, 브라우저 확인 남음
- [x] C-2. 직원에게 내 스킬로 할 수 있는 미배정 태스크만 — 사용자 입력 (`visibleTasks`: 배정됨 → 내 것인지, 미배정 → `currentUser.skills.some(skill.id === task.skill_id)`). 예측(alex.martin → 2·3번) 맞음 — build 통과, 브라우저 확인 남음
- [x] C-3. Profile 스킬 — 코드 변경 없음: `Profile.vue`가 이미 `currentUser.skills` (`{id, name}`) 표시, 저장 후 병합도 `PUT /users` 응답이 `get_user`(with_details)라 skills 유지 — 브라우저 확인 남음
- [x] 태스크 description 표시 (사용자 제안: 입력만 되고 안 보였음) — 사용자 입력 (`Tasks.vue` 제목 칸 `<template v-else>` 대신 `<span v-else>`로 감쌈 — 동작 같음 · `styles.css` `.task-description` display block·회색·0.76rem). 예측(`display: block` 빼면 흰 작은 글씨) 틀림 → 색은 그대로 회색, 제목 **옆 같은 줄**에 붙음 — build 통과, 브라우저 확인 남음
- [x] ②-1. `utils/userName.js` `useUserName(getId)` composable — 사용자 입력 (내 id → `currentUser` 이름, 남의 id → `User #id` 후 매니저만 `getUser`로 이름). 예측(`immediate` 빼면 처음엔 안 보임) 맞음. 실수: import 경로 `"../stores/auth "` 끝 공백 → 수정. 이해 확인 `return` 빼면: 반쯤 — 직원은 `User #id`로 덮임 맞음, 매니저는 깜빡 후 이름 + 불필요한 요청 — 완료 (화면 적용 남음)
- [x] ②-2. Live clock 제목 이름 — 사용자 입력 (`ClockManager.vue` import · `useUserName(() => props.userid)` · `<h1>{{ userName }}</h1>`, 세미콜론 빠졌지만 동작 같음). 예측(화살표 없이 `props.userid` → 오류) 반쯤: 빌드 에러는 아님, 문자열 값만 넘어가 Vue 경고(Invalid watch source) + 직원 바꿔도 제목 안 따라감 — build 통과, 브라우저 확인 남음
- [x] ②-3. Timesheet 디자인 + 이름 — **Claude가 넣음** (사용자 요청: 혼자 늘어나 보임). 원인: `page-grid` 없이 패널 하나가 전체 폭, 여백 없음, 입력칸 100%. → `page-grid timesheet-grid`: 왼쪽 intro(eyebrow·이름·Total) / 오른쪽 Filter 폼(From·Until 2칸 `.filter-row`, Apply primary, `@submit.prevent`) 각 span 6 + 아래 표 패널. 좁은 화면 `@media` 목록 추가. `useUserName(userId)` (computed ref도 watch 가능) — build 통과, 브라우저 확인 남음
- [x] ②-4. Insights 제목 이름 — 사용자 입력 (`ChartManager.vue` 3줄). 예측 모름 → 직원 바꿀 때 서버 응답 전까지 잠깐 `User #B id` (14번째 줄) — build 통과, 브라우저 확인 남음
- [x] ②-5. Add time 제목 이름 — 사용자 입력 (`WorkingTime.vue` `useUserName(userid)` computed 그대로, `Entry #`는 근무 기록 id라 유지). 예측(`.value` 넘기면 오류) 반쯤 — Live clock과 같음: 빌드 에러 아님, 경고 + 안 따라감 — build 통과, 브라우저 확인 남음. ② 전체 완료
- [x] 브라우저 확인 (10-08: B·C·description·②·Timesheet 레이아웃) — 사용자가 직접 확인 — 완료
- [x] 10-08 오전 커밋 — `893a816` "Show names instead of ids" (사용자, 9개 파일, push 전). 남은 것: 오후 clock → working time — 완료
- [x] 2. 퇴근 → working time 생성 — 백엔드(`clocks.ex` `create_clock`에 `with` + 직전 clock 찾기)로 시작했다가 사용자 결정으로 되돌리고 프론트로 처리. 사용자 입력 (`ClockManager.vue` `toggleClock`: `createClock` 전에 `start = startDateTime.value`, 퇴근이고 `start` 있으면 `createWorkingTime(userid, { start, end: clock.time })`. 두 시간 모두 `withLocalTime`을 거친 로컬 문자열이라 Add time과 같은 형식). 예측(같은 초 → 에러 + 다시 누르면 퇴근 2번) 맞음 — 422 뒤 `refresh()`가 건너뛰어져서 화면이 그대로임. 이해 확인: `start`를 `createClock` 아래로 옮겨도 동작함(`refresh()` 아래면 깨짐)은 반쯤 · `!nextStatus` 빼도 `start`가 null이라 출근 때 실행 안 됨은 틀림. Claude가 처음에 "createClock 전에 잡아야 한다"고 잘못 설명했다가 정정. 예전 clock 기록은 소급 안 함 — build 통과, 브라우저 확인 — 완료
- [x] 3. 커밋 · PR — 사용자: `ee452f7` "Create working time on clock out" → push → PR #11 main 병합 `f39a2a0` (6423c93 · 893a816 · ee452f7, main의 e385c7b와 충돌 없음). Claude 확인: 로컬 main = origin/main, 빠진 커밋 없음, build 통과 — 완료
- [x] theme04 4-2 · 4-4 (04 튜토리얼 · 05 매니저 가이드) — **Claude가 진행** (사용자 요청). 배포 DB에 데모 데이터 추가 (admin으로: 스킬 Frontend·Backend → donghyun_hwang, 태스크 3개, 근무 기록 8개 09-29~10-09). 캡처 12장 `docs/mockups/` (빨간 테두리는 화면 CSS만). 발견: Overview "weekly hours"·days가 전체 기간 합계 · 배포판에서 근무 기록 1개 사라짐 + 15:26→15:31 퇴근 때 근무 기록 안 생김 (재배포 확인 필요) — 커밋 남음
- [x] Overview 이번 주만 — **Claude가 넣음** (사용자 요청) — `DashboardHome.vue` `startOfWeek()` (이번 주 월요일 00:00 로컬) → `listWorkingTimes(user.id, { start })` (API의 기존 start 필터 사용, UTC 변환은 api.js가 처리). 일요일도 그 주 월요일로 계산 확인 — build 통과. 병합·재배포 후 `08-overview.png` 다시 캡처
