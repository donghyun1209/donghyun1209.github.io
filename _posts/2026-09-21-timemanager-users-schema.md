---
title: "TimeManager: users 스키마와 마이그레이션"
date: 2026-09-21 09:00:00 +0900
categories: [EPITECH, TimeManager]
tags: [elixir, phoenix, postgresql, migration, database]
---

프랑스 EPITECH 교환학생 기간에 진행하는 첫 팀 프로젝트 TimeManager를 시작했다. \\
I started TimeManager, the first team project of my EPITECH exchange semester in France.

---

## 1. 형식

```
users = { username: string(required), email: string(required, X@X.X 형식) }
```

## 2. 느낀 점

처음 경험해보는 분야라 아직 공부가 더 필요할 것 같다. \\
It's my first time in this field, so I think I still need to study more.

안전 장치를 마이그레이션과 changeset 양쪽에 걸어야 하는 이유: DB에 직접 넣으려고 하는 시도에서도 형식을 지키도록 유지할 수 있기(?)때문.\\
Reason for setting the safeguard on both the migration and the changeset: because it keeps the format enforced even for attempts that try to insert directly into the DB (?)

