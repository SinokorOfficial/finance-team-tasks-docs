# 권한

누가 무엇을 할 수 있고, 서버는 권한을 어떤 순서로 판정하는지 정리합니다.

## 권한 3단계

권한은 포털 **계정 관리**에서 사람마다 「업무 현황」 앱 권한을 고르는 방식입니다(계정 문서 `apps.tasks`).

| 권한 | 값 | 할 수 있는 일 |
|---|---|---|
| 관리자 | `admin` | 담당자가 할 수 있는 일 전부 + 공휴일 관리(추가·삭제·제외 되돌리기) + 안내 메일 설정(보관만) + 관리 탭 담당자 표 보기(조회 권한자 포함) + 남의 규칙 `forceIds`·기간 전체 `force` 생성 |
| 담당자 | `member` | 업무 등록·수정·삭제(본인·등록자 범위), 본인 업무 완료 표시, 반복 규칙 등록·수정, 규칙에서 생성, 엑셀 일괄 등록·내보내기, 휴지통 |
| 조회 | `viewer` | 현황·캘린더 보기, 엑셀 양식·내보내기만. **GET 이 아닌 API 는 전부 403**(`require_tasks`), 휴지통 403, 담당자로 지정 불가 |
| 권한 없음 | 키 없음(일반 계정) · `"none"` · 목록 밖 값 | 업무 현황 API 전부 403, 메뉴 안 보임 |

## 업무 한 건 단위 권한(코드 함수 기준)

| 동작 | 누가 | 코드 |
|---|---|---|
| 팀 업무 보기 | 업무 현황 권한이 있는 모든 계정 | `tk_visible` |
| 개인 업무(🔒) 보기 | 담당자 본인만 — **관리자도 못 봄**(목록·내보내기·SSE·휴지통 모두) | `tk_visible` |
| 내용 수정·삭제·휴지통 복원·영구 삭제 | 담당자 본인 · 등록한 사람 · 관리자 | `is_editor` |
| 상태·완료 표시(`status`·`doneAt`·`doneBy`) | 담당자 본인만(관리자도 못 함) | `check_update` |
| 개인 업무로 전환 | 담당자 본인만, 개인 업무의 담당자는 본인이어야 함 | `check_update`·`check_new` |
| 담당자가 될 수 있는 사람 | 승인된 계정 중 업무 현황 `admin`·`member` | `assignable_people` |
| 기한 알림 메일 켜고 끄기 | 본인만, `admin`·`member` 만(조회는 403) | `PATCH /api/auth/me` |

## 권한 판정 흐름


```
계정 문서 accounts/{email}
  ├ status ≠ "approved"                          → 권한 없음
  ├ apps/tasks 문서 없음 또는 enabled = false     → 권한 없음(허브 관리자 포함 — 「사용 중지」)
  ├ 허브 role = "admin" 이고 apps 에 "tasks" 키가 아예 없음 → admin   (처음 설치 때 잠김 방지 기본값)
  └ 그 밖: apps.tasks 값(app_role)
        값이 apps/tasks.roles 목록의 id 이고 admin·member·viewer 중 하나 → 그 값
        "none" · "" · 목록 밖 값 → 권한 없음
```

| 함수 | 쓰는 곳 | 차이 |
|---|---|---|
| `tasks_role(acct, a)` | 모든 업무 현황 라우트·SSE·담당자 목록 | 앱 `enabled` 확인 포함 |
| `eff_app_role(acct, a)` | `/api/apps` 메뉴 `myRole`·`/go` | builtin 앱이면 같은 규칙. `enabled` 는 목록 단계에서 거름 |
| `require_tasks(req)` | 라우트 입구 | 반환 사전의 `role` = 업무 현황 권한, `hubRole` = 허브 권한. viewer 의 비 GET 403 |
| `require_tasks_admin(req)` | 공휴일 쓰기·설정 | 업무 현황 admin 아니면 403 |
| 화면 `roleOf()`·`allowed()` | `hub.html` | `allowed` = 로그인 + `/api/apps` 에 `{id:"tasks", mode:"builtin"}` + `roleOf()` |

- 「없음」 저장: 계정 관리에서 builtin 앱의 「없음」을 고르면 키를 지우지 않고 `"none"` 을 저장합니다(`APP_ROLE_NONE`). 키를 지우면 허브 관리자가 다시 admin 으로 돌아가기 때문입니다. 다른 앱(입장권·링크)의 「없음」은 키 삭제입니다.
- 담당자 지정 시 확인: 담당자를 **새로 정할 때만** 그 계정의 업무 현황 권한을 봅니다(`validate_common`). 권한 없음·조회는 400.

---

**관련 문서**

- [데이터 모델](data-model.md)
- [API](api.md)
- [실시간(SSE)](sse.md)
