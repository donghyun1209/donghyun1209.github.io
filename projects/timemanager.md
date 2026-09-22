---
layout: page
title: "TimeManager (EPITECH)"
permalink: /projects/timemanager/
---

> **EPITECH TimeManager** — 프랑스 EPITECH 교환학생 기간 중 진행 중인 팀 프로젝트.

**학교**: EPITECH (프랑스) — 교환학생 과정\\
**형태**: 팀 프로젝트\\
**상태**: 진행 중 (2026.09~)

---

**기술 스택 (제가 다룬 백엔드 파트)**: Elixir · Phoenix · PostgreSQL

## 개발 로그

아래 포스트들에서 개발 과정을 기록하고 있습니다.

{% for post in site.posts %}{% if post.categories contains 'TimeManager' %}
- [{{ post.title }}]({{ post.url }}) <small>{{ post.date | date: "%Y-%m-%d" }}</small>
{% endif %}{% endfor %}
