# 데이터 모델

업무 현황이 저장하는 문서(컬렉션)와 각 칸의 뜻을 정리합니다.


모두 포털 문서 저장소의 컬렉션입니다(`store.doc_put`/`doc_get`). 업무 현황 컬렉션은 **범용 `/api/db/*` 허용 목록 `COLLS` 에 없습니다**(404). 업무 현황 meta 문서 셋(`taskSkips`·`taskSettings`·`holidays`)도 범용 `/api/db/meta/…` 로 읽기·쓰기·목록 모두 404(허브 관리자 포함, `check_doc_route`).

| 문서 | 내용 |
|---|---|
| `tasks/{id}` | 업무 한 건. 손 등록 id = 무작위 20자(`tk_new_id`), 규칙 생성 id = `g_{규칙id}_{YYYYMM}` |
| `taskTemplates/{id}` | 반복 규칙 |
| `taskTrash/{id}` | 휴지통 — 지운 순간의 업무 원문(같은 id, `history` 포함) + `deletedAt`·`deletedBy`·`deletedByEmail`. 자동 비우기 없음 |
| `meta/taskSkips` | `{생성 id: 값}` — 값 = 지운 시각(ISO) 또는 `"past:" + 시각`(규칙 등록 때 이미 지난 기한이라 건너뜀). 한 칸씩 merge |
| `meta/taskSettings` | `{remind:{enabled,time,days,skipHolidays}, updatedAt, updatedBy}` — **보관만, 발송 없음**. 기본값 `{enabled:true, time:"08:30", days:3, skipHolidays:true}` |
| `meta/holidays` | `{add:{날짜:이름}, del:[날짜], updatedAt}` |
| `apps/tasks` | 앱 등록표 항목 `{id:"tasks", name:"업무 현황", mode:"builtin", enabled:true, roles:[admin·member·viewer], order:0, …}` — 없으면 기동 때·처음 부를 때 생성 |
| `accounts/{email}` 의 `apps.tasks` | 업무 현황 권한(`admin`·`member`·`viewer`·`"none"`) |
| `accounts/{email}` 의 `taskRemindOff` | 본인 기한 안내 메일 끄기(참/거짓) |

## `tasks` 문서 칸

| 칸 | 뜻 | 누가 쓰나 |
|---|---|---|
| `title` | 업무명(필수, 120자) | 사용자 |
| `dueDate` | 기한 `YYYY-MM-DD`(필수) | 사용자 / 생성 때 `due_of` |
| `startDate` | 시작일(선택, 기한 이하) | 사용자 / 생성 때 기한 − `startBefore` |
| `ownerEmail` | 담당자 이메일(소문자, 필수) | 사용자 |
| `status` | `예정`·`진행`·`완료`·`보류`·`취소`(기본 예정) | 담당자 본인 / 서버(자동 진행·담당자 변경) |
| `group` | 구분(목록 값 또는 빈 값) | 사용자 |
| `company` | 회사(목록 값 또는 빈 값) | 사용자 |
| `institution` | 관련 기관, 부서(200자) | 사용자 |
| `project` | 관련 선박·프로젝트(200자) | 사용자 |
| `external` | 외부 기한(참/거짓) | 사용자 |
| `private` | 개인 업무(참/거짓) | 담당자 본인 |
| `note` | 메모(2,000자) | 사용자 |
| `doneAt`·`doneBy` | 완료 일시·완료자 | **서버** |
| `prevStatus` | 완료·취소 직전 상태 | **서버** |
| `autoStarted` | 시작일 도래로 자동 '진행' 됨 | **서버** |
| `createdAt`·`createdBy`·`createdByEmail` | 등록 시각(UTC ISO)·등록자 이름·이메일 | **서버**(규칙 생성 건은 규칙 만든 사람 + 「 (규칙 생성)」) |
| `updatedAt`·`updatedBy` | 최근 수정 | **서버** |
| `history` | 이력 배열 `{at, by, action, changes?, memo?}` — 최대 100건(첫 '등록'은 밀려나지 않음), 값은 300자까지 | **서버** |
| `templateId`·`gen` | 생성한 규칙 id·규칙 생성 건 표시(`gen:true`) | **서버**(생성 때) |

## `taskTemplates` 문서 칸

| 칸 | 뜻 | 검증 |
|---|---|---|
| `title`·`ownerEmail`·`group`·`company`·`institution`·`project`·`external`·`private`·`note` | 업무와 같음(생성 업무로 복사) | 업무와 같음 |
| `months` | 해당 달 목록(정수 1~12, 하나 이상) | 정렬·중복 제거해 저장 |
| `dueDay` | `"1"`~`"31"` 또는 `"말일"` | 문자열로 저장 |
| `shift` | `none`(그대로) · `back`(앞 영업일) · `fwd`(다음 영업일) | 기본 `none` |
| `startBefore` | 시작일 = 기한 − N일(0~60, 선택) | 빈 값이면 칸 삭제 |
| `active` | 사용 여부(기본 참) | 참/거짓 |
| `createdAt`·`createdBy`·`createdByEmail`·`updatedAt`·`updatedBy` | 서버가 찍음 | — |

## 화면으로 나가는 모양(`task_out`)

- 업무·휴지통 행: 저장 문서에서 `history` 를 빼고 `histN`(이력 건수), `startDate`(없으면 `""`), `autoStarted`(참/거짓), `owner`(담당자 이름)를 채움. 이력은 `GET /api/tasks/{id}/history` 로 따로.
- 휴지통 행: 위 + `deletedAt`·`deletedBy`·`deletedByEmail`(빈 값 채움) + `canRestore`.
- 규칙 행: `startBefore`, `owner` 채움. 목록 API 에서 `next`·`made`·`ownerOk` 를 더함.

## 선택지(서버·화면 같은 값)

| 칸 | 값 |
|---|---|
| 구분 `TASK_GROUPS` | 원리금상환 · 사후관리 · 현황자료 · 정기예금 · 재금융 · 결산·감사 · 신용평가 · 공시 · 외국환신고 · 보험 · 기타 |
| 회사 `TASK_COMPANIES` | 장금상선 · 흥아라인 · 한성라인 · 흥아해운 · 장금흥아통합 · 기타 |
| 상태 `TK_STATUSES` | 예정 · 진행 · 완료 · 보류 · 취소(화면 표시만 「취소됨」) |

목록 밖 구분·회사는 400. 빈 값 허용, 고치지 않은 옛 값은 통과(`validate_common`). 화면 `hub.html` 의 `GROUPS`·`COMPANIES` 상수와 **손으로 맞춰야** 합니다(공용 정의 없음).

---

**관련 문서**

- [API](api.md)
- [반복 규칙 생성](generate.md)
- [동시성·충돌](concurrency.md)
