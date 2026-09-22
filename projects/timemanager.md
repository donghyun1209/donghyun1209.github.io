---
layout: page
title: "TimeManager (EPITECH)"
permalink: /projects/timemanager/
---

> **EPITECH TimeManager** — 프랑스 EPITECH 교환학생 기간 중 진행 중인 팀 프로젝트. 직원 근무시간(clock-in/out, working time) 관리 시스템을 만드는 과제로, 저는 `users` 스키마/마이그레이션 파트를 담당하고 있습니다.

**학교**: EPITECH (프랑스) — 교환학생 과정\\
**형태**: 팀 프로젝트\\
**상태**: 진행 중 (2026.09~)

---

## 프로젝트 개요

`theme01_project.pdf` 스펙에 따라 진행되는 팀 프로젝트로, 회원(`users`)과 그에 연관된 근무 기록(`clocks`, `workingtime`)을 관리하는 시스템을 만듭니다. 저는 그중 `users` 스키마와 마이그레이션을 담당했습니다.

## 담당 파트 — `users` 스키마 (Ecto / PostgreSQL)

스펙에서 요구하는 필드는 이름·타입을 임의로 바꿀 수 없습니다:

```
users = { username: string(required), email: string(required, X@X.X 형식) }
```

`clocks`/`workingtime`은 `user`에 `belongs_to`하는 연관관계라, 팀원이 해당 스키마를 만들려면 제 `users` 마이그레이션이 먼저 적용돼 있어야 합니다. (`clocks`/`workingtime` 스키마 자체는 팀원 담당)

**기술 스택 (제가 다룬 백엔드 파트)**: Elixir · Phoenix · Ecto · PostgreSQL

## 진행 로그

### 2026-09-21

- [x] `users` 스키마 + 마이그레이션 생성
  - 체크포인트: **설명하기** — username/email에 required 제약을 마이그레이션과 changeset 양쪽에 다 걸어야 하는 이유
- [x] `mix ecto.migrate` 실행 후 DB에서 `users` 테이블/제약조건 직접 확인 (`psql`로 `\d users`)
  - 체크포인트: **예측+설명** — 통과

## 개발 로그

아래 포스트들에서 개발 과정을 기록하고 있습니다.

{% for post in site.posts %}{% if post.categories contains 'TimeManager' %}
- [{{ post.title }}]({{ post.url }}) <small>{{ post.date | date: "%Y-%m-%d" }}</small>
{% endif %}{% endfor %}
