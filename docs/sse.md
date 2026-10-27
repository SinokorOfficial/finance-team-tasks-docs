# 실시간(SSE)

변경이 다른 사람 화면에 바로 반영되는 방식과 공개 범위 규칙을 정리합니다.


| 구분 | 규칙 |
|---|---|
| 통로 | 포털 공용 `/api/stream`. 메시지 `{coll, id, data, version, updatedAt}`. 화면은 `shim.js` 가 연 연결에 `EventSource` 를 감싸 귀를 하나 더 담(연결은 하나) |
| 대상 컬렉션 | `TASK_SSE_COLLS` = `tasks`·`taskTemplates`·`taskTrash`(COLLS 밖이지만 `on_change`·`/api/stream` 에서 따로 통과) |
| 권한 없음 | 보내지 않음 |
| 휴지통 | 조회(viewer) 연결에는 보내지 않음. `data` = 휴지통 행 모양(받는 사람 기준 `canRestore`) |
| 팀 문서 저장 | 권한 있는 연결 모두에 `task_out` 모양(이력 없음, `owner` 이름) |
| 개인 문서 저장 | 담당자 본인에게만 내용. 다른 사람에게는 **직전이 팀이었을 때만** `data:null` 한 번(팀→개인 전환), 그 밖엔 id 도 보내지 않음 |
| 삭제(`data:null`) | 직전이 팀 → 권한 있는 연결 모두 / 개인 → 담당자 본인만 / 모름 → 보내지 않음 |
| 업무 현황 meta | `meta/holidays`·`meta/taskSettings` 는 업무 현황 권한자에게만. `meta/taskSkips` 는 **누구에게도** 보내지 않음 |
| 계정 변경 | `accounts` 변경은 포털 공용 규칙으로 감. 화면은 받으면 `GET /api/tasks/people` 만 다시 받음 |

## 직전 공개 상태 기억(`_tk_vis`)

- 레플리카 메모리 사전 `_tk_vis = {(coll, id): "" | 담당자 이메일}`(팀 = `""`, 개인 = 담당자 이메일 소문자). 상한 `_TK_VIS_MAX = 50,000`(넘으면 오래된 것부터 버림), `threading.Lock` 보호.
- 갱신: 모든 변경이 지나는 `broadcast()` 에서 `tk_sse_mark(msg)` 가 직전 값을 메시지 내부 칸 `_tkPrev` 로 달고 기억을 바꿈. `/api/stream` 이 `_tkPrev` 를 떼고 `task_stream_view(u, coll, data, prev)` 로 사람마다 가공(False = 안 보냄, None = 삭제로 알림, dict = 행).
- 기억에 없는 문서의 삭제는 같은 id 짝(`tasks` ↔ `taskTrash`)을 읽어 판정.
- 기동: `_tasks_boot` 가 저장소 연결을 기다린 뒤 `ensure_builtin_apps`·`meta/taskSkips` 빈 문서 생성·`tk_sse_preload`(세 컬렉션 전체 읽어 채움).
- 한계: 상한 밖으로 밀린 문서, PG 리스너 재접속 사이(약 3초) 놓친 NOTIFY, 기동 직후 채우기 전 틈의 변경은 직전 상태를 몰라 **그 삭제 알림을 다른 사람에게 보내지 않음**(새는 쪽은 없음, 새로고침이면 바로잡힘).

## 화면 쪽 수신(`hub.html`)

- `TK_COLLS`(tasks·taskTemplates·taskTrash)·`accounts`·`meta/holidays*` 메시지를 `tk:change` 사건으로 바꿈. 재접속(두 번째 `open`)이면 `{coll:"*"}`.
- 업무·규칙 변경 → 목록 전체 재수신(창·펼친 화면에서 입력 중이면 미룸 — `autoReloadOk`). 휴지통 변경 → 250ms 묶어 휴지통만 재수신(입력 보호 무관). 목록에 없는 id 의 `data:null` 은 무시.
- 보고 있는 업무가 바뀌면: 요약 보기면 250ms 뒤 다시 그림, 고치기 모드면 안내 띠 `#tk-staleBar`.

---

**관련 문서**

- [권한](rbac.md)
- [동시성·충돌](concurrency.md)
- [화면 모듈](frontend.md)
