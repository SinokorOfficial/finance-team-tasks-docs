# API

업무 현황 라우트 전체(경로·권한·본문·응답)를 정리합니다.


공통: 로그인 없음 401, 업무 현황 권한 없음 403, 조회(viewer)의 GET·HEAD 외 요청 403(`require_tasks`).
JSON 본문 라우트(`tk_body`)는 `Content-Type` 이 JSON 이 아니면 415, 1MB 초과 413, 객체가 아니면 400. 오류 응답은 `{"detail": "문구"}`.
id 모양(`^[A-Za-z0-9_-]{1,80}$`)이 아니면 404.

| 메서드·경로 | 권한 | 본문/질의 | 응답 요지 | 비고 |
|---|---|---|---|---|
| `GET /api/tasks` | viewer+ | — | `{tasks, autoMade, genDups}` | 호출 때 생성(2달, 잠금 대기 0초 — 바쁘면 건너뜀) + `auto_start`. 행에 `history` 없음 |
| `GET /api/tasks/{id}/history` | viewer+ | — | `{history:[…]}`(저장 순서) | 없음·남의 개인 업무 404 |
| `POST /api/tasks` | member+ | 업무 칸 | `{ok, id}` | 개인 업무는 본인 담당만, 남의 업무 상태는 '예정'만(403). '완료'로 등록하면 `doneAt`·`doneBy` 찍음 |
| `PATCH /api/tasks/{id}` | member+(수정 = 담당자·등록자·admin, 상태 = 담당자) | 바꾼 칸 + 선택 `_base:{칸:연 때 값}` | `{ok}` / 409 `{detail, conflicts, task}` | [동시성·충돌](concurrency.md) 참고. 담당자 변경 시 상태 규칙 |
| `DELETE /api/tasks/{id}` | member+(담당자·등록자·admin) | — | `{ok, trashed:true}` | 휴지통으로 옮김. 생성 건은 `taskSkips` 기록 |
| `GET /api/tasks/people` | viewer+ | — | `{people:[{email,name,role}]}` + 관리자에게만 `viewers` | 이름순 |
| `GET /api/tasks/settings` | admin | — | `{settings, sending:false, remindOff:[{email,name}]}` | 화면은 더 부르지 않음(라우트 유지) |
| `PATCH /api/tasks/settings` | admin | `enabled`·`skipHolidays`(참/거짓)·`time`(HH:MM)·`days`(0~30) | `{ok, settings}` | 보관만 |
| `PATCH /api/tasks/settings/remind/{email}` | member+ 본인만 | `{off: bool}` | `{ok}` | 남의 이메일이면 관리자도 403. 호환용 |
| `GET /api/tasks/template.xlsx` | viewer+ | — | xlsx | 담당자 드롭다운은 이름 합계 250자 미만일 때만 |
| `GET /api/tasks/export.xlsx` | viewer+ | `status`·`owner`·`group`·`q`·`from_`·`to` | xlsx 19열 | 호출 때 `auto_start`. 기한·업무명 순 정렬 |
| `POST /api/tasks/import-preview` | member+ | 본문 = xlsx 원문(5MB) | `{rows:[{row,data,errors,codes}], ok, bad}` | 저장 안 함. 시트 「업무등록」 없으면 첫 시트 |
| `POST /api/tasks/bulk` | member+ | `{tasks:[…]}`(1~500) | `{ok, made, ids}` | 하나라도 위반이면 400·전부 미등록. 완료·취소 불가 |
| `GET /api/tasks/templates` | viewer+ | — | `{templates:[…+next,made,ownerOk]}` | `made` 는 휴지통 제외·볼 수 있는 업무만 |
| `POST /api/tasks/templates/preview` | member+ | 규칙 키 + 선택 `n`(1~12) | `{next:[{due,weekday,base,shifted,reason}], skippedThisMonth}` | 저장 안 함. 검증 문구는 규칙 저장과 같음 |
| `POST /api/tasks/templates` | member+ | 규칙 칸 | `{ok, id, made, dups}` | 저장 직후 그 규칙만 2달 생성(`skip_past`, 잠금 5초 대기) |
| `POST /api/tasks/generate` | member+ (`forceIds` 는 본인 규칙 gid 만, `force` 는 admin 만) | `{months:1~12, forceIds?, force?/includeDups?}` | `{ok, made, exists, skippedDeleted, skippedPast, skipped, dups, rules}` | 잠금 5초 못 잡으면 503 |
| `PATCH /api/tasks/templates/{id}` | member+(담당자·등록자·admin) | 규칙 칸 | `{ok}` | 이미 만든 업무는 바뀌지 않음 |
| `DELETE /api/tasks/templates/{id}` | member+(담당자·등록자·admin) | — | `{ok}` | 규칙 삭제(휴지통 없음) |
| `GET /api/tasks/trash` | member+(viewer 403) | — | `{trash:[…+deleted*,canRestore]}` | 최근 삭제 순, 남의 개인 업무 제외 |
| `POST /api/tasks/trash/{id}/restore` | member+(담당자·등록자·admin) | — | `{ok, id, unskipped, dupOf}` | 같은 id 있으면 409, 생성 건은 잠금 `tasks:gen` 못 잡으면 503 |
| `DELETE /api/tasks/trash/{id}` | member+(담당자·등록자·admin) | — | `{ok}` | 영구 삭제, `taskSkips` 는 유지 |
| `GET /api/holidays` | viewer+ | `year`(2000~2100, 기본 올해) | `{year, holidays, items, builtinYears, years}` | `years` = 내장·저장된 해 + 올해·내년 |
| `POST /api/holidays` | admin | `{date, name}`(이름 30자) | `{ok}` | 잠금 `tasks:holidays` |
| `DELETE /api/holidays/{date}` | admin | — | `{ok}` | 추가한 날은 삭제, 내장은 '제외' |
| `POST /api/holidays/{date}/restore` | admin | — | `{ok}` | 제외한 내장 되돌리기 |
| `PATCH /api/auth/me` | 로그인 + 업무 현황 member·admin | `{taskRemindOff: bool}` | `{ok, account}` | 형식 400 먼저, viewer·권한 없음 403 |

라우트 순서 주의: 고정 주소(`people`·`settings`·`template.xlsx`·`templates`·`trash` 등)는 `/api/tasks/{id}` 보다 **먼저** 선언되어 있습니다(같은 메서드면 먼저 선언된 쪽이 잡음). 새 고정 주소를 더할 때도 `PATCH`·`DELETE /api/tasks/{id}` 위에 두어야 합니다.

## 앱 등록표 관련(포털 공용 라우트)

| 메서드·경로 | 업무 현황(`builtin`)에서 |
|---|---|
| `GET /api/apps` | `url:null`, `myRole` = `eff_app_role`(허브 admin 은 키 없을 때만 admin) |
| `POST /api/apps/tasks/ticket` | 입장권 없이 `{"mode":"builtin"}` |
| `GET /go/tasks` | 권한 있으면 `302 /#tasks` |
| `PATCH /api/apps/admin/tasks {"mode":"builtin"}` | 옛 ticket·link 등록을 builtin 으로 한 방향 전환(주소·비밀키·안 쓴 입장권 삭제, 계정별 권한 유지) |

---

**관련 문서**

- [권한](rbac.md)
- [데이터 모델](data-model.md)
- [알려진 한계](limits.md)
