# 알려진 한계·2단계

아직 없는 기능, 알고 있는 한계, 기능 규약 문서와 코드가 다른 곳을 정리합니다.


## 2단계(아직 없는 것)

| 항목 | 지금 상태 |
|---|---|
| 기한 임박 안내 메일 발송 | 설정(`meta/taskSettings`)·개인 끄기(`taskRemindOff`)만 저장. 예약·[지금 발송] 없음, `sending:false`. 전산팀 무인 발송 경로 결정 후. 화면 도움말에도 「지금은 설정만 저장」 명시 |
| 메일 확인(담당자 PC 수집 도우미 → `/api/tasks/mails/*`) | 없음. 독립판 `/api/mails/*`·도우미 토큰은 옮기지 않음 |
| 시험 입장(`/dev/login`)·`/sso` | 옮기지 않음 |
| builtin → ticket·link 역전환 | 없음(한 방향 전환만) |
| 2028년 이후 공휴일 | 내장 목록 없음 — 관리자가 직접 추가해야 함. 추가 전에는 반복 규칙이 주말만 피함 |
| 독립판·Artifact 판 자료 이관 | 이 기능에 포함 안 됨(담당자 id → 이메일 매핑, 수동·별건) |

## 알려진 한계·주의

| 항목 | 내용 |
|---|---|
| 예약 작업 없음 | 반복 생성·자동 진행은 누군가 목록을 열거나 내보내기를 할 때만 돎 |
| 규칙 등록 때 생성 잠금이 바쁘면 | 그 규칙의 이번 달 지난 기한 건너뛰기가 적용되지 않을 수 있음([알려진 한계](limits.md) D1) |
| 기존 업무 불변 | 규칙을 고치거나 공휴일을 바꿔도 이미 만들어진 업무의 기한·내용은 그대로 |
| 옛 `taskSkips` | 커밋 `6299ed6` 이전에 쌓인 지난 기한 기록은 `past:` 접두가 없어 `skippedDeleted`(「지워서 건너뜀」)로 셈 |
| 중복 판정 놓침 | 같은 일인데 제목 차이가 커서 겹침으로 못 잡는 경우는 두 건이 함께 생김 — 담당자가 하나를 지움(의도된 보수 규칙) |
| 개인 업무 휴지통 | 담당자가 업무 현황 권한을 잃으면 그 사람의 개인 업무 휴지통 건은 아무도(관리자도) 복원·영구 삭제 못 함 |
| 휴지통 | 자동 비우기 없음. 복원 때 담당자 권한 재확인 없음 |
| SSE 기억 | 레플리카 메모리(상한 50,000) — [실시간(SSE)](sse.md) 한계 참고 |
| 목록 크기 | 변경이 생길 때마다 보고 있는 사람 전원이 `GET /api/tasks`·people·templates 를 다시 받음. 문서 크기 상한 없음 |
| 일정에서 업무 만들기 | 같은 일정으로 이미 만든 업무가 있는지 따지지 않음(사용자 결정) |
| 내보내기 건수 | 화면 「위 n건 제외」와 파일 건수가 다를 수 있음(의도) |
| 선택지 이중 관리 | 구분·회사 목록이 `app.py` 와 `hub.html` 에 따로 있음 |
| 권한 종류 목록 | 서버는 `apps/tasks.roles` 목록을, 화면은 고정 3종을 봄([알려진 한계](limits.md) D3) |
| CI | 업무 현황 시험을 CI 가 돌리지 않음 |
| 이름 | 계정 이름이 없으면 이메일이 이름 자리에 나옴(`name_map`) |

## 문서와 코드 차이


| # | 문서 | 코드 | 비고 |
|---|---|---|---|
| D1 | `docs/tasks.md` 「반복 규칙」: 「규칙을 새로 만든 날 이미 지난 기한(이번 달)은 만들지 않습니다」(예외 없음) | `tasks_tpl_add` 의 생성이 잠금 `tasks:gen` 을 5초 안에 못 잡으면 `made:0` 으로 넘어가고, 그 뒤 목록 조회의 생성은 `skip_past=False` 라 **이번 달 지난 기한 업무가 만들어질 수 있음**. 코드 주석(3110행)에는 「지난 기한 건너뛰기는 그때 안 됨: docs/tasks.md」라고 적혀 있으나 tasks.md 에 해당 문장 없음 | 문서 보완 필요 |
| D2 | `docs/tasks.md` 「API」: `POST /api/tasks/templates` → `{ok, id, made, dups}`, `dups` 모양은 `{gid, templateId, title, dueDate, taskId, taskTitle, taskDueDate}` | 규칙 등록 응답의 `dups` 는 `gen_dups_for` 를 거치지 않은 원본이라 **`ownerEmail` 칸이 더 실림** | 노출 범위상 문제는 확인 못 함(같은 담당자의 팀 업무만 견줌) — 확인 필요 |
| D3 | `docs/tasks.md` 「권한 규칙」: `"none"`·**권한 목록에 없는 값**·빈 문자열은 접근 없음 | 서버 `app_role` 은 값이 **앱 문서 `apps/tasks` 의 `roles` 목록**에 있어야 인정. 화면 `roleOf` 는 고정 `ROLE_LABEL`(admin·member·viewer)로만 판정 | 관리자가 하위 메뉴에서 「권한 종류」를 바꾸면 서버·화면 판정이 갈릴 수 있음 — 확인 필요 |
| D4 | `docs/portal-apps.md` 「builtin」: 「이 전환은 내장 기능 id(`tasks`)만 됩니다」 | `PATCH /api/apps/admin/{id}` 는 `BUILTIN_APPS` 의 id(`tasks`·`news`·`invest`·`mailbox`)면 모두 builtin 전환 허용(2115행) | 문서가 옛 상태 |
| D5 | `docs/architecture.md` 「배포」: 「ax-fnc 의 main 에 push」 | 현재 운영 절차는 개인 가지 → develop PR → develop→main PR(deploy.yml 머리 주석이 develop→main PR 을 전제) | 문서가 옛 상태 |
| D6 | `app.py` 2180행 주석: 「실시간(SSE)은 `TASK_COLLS` 허용 목록으로 따로 통과」 | 실제 허용 목록은 `TASK_SSE_COLLS`(= tasks·taskTemplates·taskTrash) | 코드 주석이 옛 상태 |
| D7 | `hub.html` 6242행 주석: 「남의 개인 업무는 `data:null`(id 만)로 보낸다」 | 서버는 26-10-13 부터 남의 개인 업무 변경을 **보내지 않음**(팀→개인 전환 한 번만 `data:null`) | 코드 주석이 옛 상태. 동작 영향 없음 |
| D8 | `docs/tasks.md` 「저장」 표: `meta/taskSettings` = `{remind:{…}}`, `meta/holidays` = `{add, del}` | 실제로는 `updatedAt`(설정은 `updatedBy` 도) 칸이 함께 저장됨 | 사소 |
| D9 | `docs/tasks.md` 「API」: 오류 일반 설명에 415·413 없음 | `tk_body` 가 JSON 아님 415, 1MB 초과 413 | 사소 |
| D10 | `docs/tasks.md` 「엑셀」: 「5MB·500행까지」 | 5MB 초과는 413 이 아니라 **400**(「파일이 비었거나 5MB를 넘습니다」). 시트 행은 머리글 탐색용으로 520행까지 읽고 결과는 500행에서 자름 | 사소 |

API 표 자체(경로·권한·응답 모양)는 `docs/tasks.md` 「API」 절과 코드가 위 항목 말고는 일치했습니다(라우트 27개 대조).

---

**관련 문서**

- [반복 규칙 생성](generate.md)
- [실시간(SSE)](sse.md)
- [변경 이력](changelog.md)
