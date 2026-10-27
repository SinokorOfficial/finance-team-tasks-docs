# 시험

서버·화면 시험 파일과 실행 방법, 주의 사항을 정리합니다.


| 파일 | 종류 | 실행(server/ 에서) | 규모 |
|---|---|---|---|
| `test_tasks.py` | 서버. FastAPI `TestClient` 로 앱을 그 자리에서 부름. 저장소 = 임시 폴더 SQLite(`HUB_DB` 를 임시 경로로 지정), 세션은 `store.session_create` 로 직접 | `python test_tasks.py`(항목마다 PASS/FAIL) 또는 `python -m pytest -q test_tasks.py` | 시험 함수 25개 · 항목 478개 |
| `test_ui_tasks.py` | 화면. Playwright 로 정적 사본(`static/`)을 띄우고 `/api/**` 를 가짜로 응답(서버 안 띄움) | 별도 창에서 `python -m http.server 8079 --directory <static 사본>` → `python test_ui_tasks.py --base http://127.0.0.1:8079 [--shots <폴더>]`. pytest 로는 `HUB_UI_TASKS_BASE` 환경 변수가 있어야 돎(없으면 건너뜀) | 559항목(docs/tasks.md 기재 — 이번에 실행 안 함, **확인 필요**) |
| `test_ui_events.py` 묶음 14 | 화면. 일정에서 업무 만들기 | 위와 같은 방식 | 14-16~14-30 |

## 실측(2026-10-26, 이 문서 작성 때)

`server/` 에서 `python test_tasks.py`(파이썬 3.12, `PYTHONIOENCODING=utf-8`) → **「항목 478개 · 시험 함수 25개 · 모두 통과」**.

## `test_tasks.py` 시험 함수 목록

| 함수 | 다루는 것 |
|---|---|
| `test_builtin_app_and_isolation` | 앱 등록표 builtin, 범용 `/api/db` 격리 |
| `test_roles` · `test_hub_admin_explicit_role` | 권한 3단계, 허브 admin 명시값(`member`·`"none"`·`viewer`·`""`) |
| `test_team_task_rules` · `test_private` | 팀 업무 수정·삭제 권한, 개인 업무 |
| `test_owner_change_and_history` · `test_ux1_owner_change_status_and_viewers` | 담당자 변경 상태 규칙·이력, `viewers` |
| `test_generate` · `test_gen_lock` · `test_gen_dups` | 반복 생성, 생성 잠금, 손 등록 겹침·forceIds |
| `test_due_of_and_kst` · `test_holidays_api` | 기한 계산·한국 시각, 공휴일 API |
| `test_start_date_and_auto_start` | 시작일·자동 진행 |
| `test_excel` | 양식·미리보기 codes·일괄·내보내기 |
| `test_settings_and_me` | 설정·알림 끄기·`PATCH /api/auth/me` |
| `test_sse_view` · `test_sse_stream_live` | SSE 가공·실제 스트림 |
| `test_task_meta_guard` | meta 셋 범용 경로 차단 |
| `test_release_lock_holder` | 잠금 holder |
| `test_history_split` | 목록 `histN`·이력 분리 |
| `test_base_conflict` | `_base` 칸 단위 409 |
| `test_trash` · `test_trash_races` | 휴지통, 복원·생성 경합 |
| `test_title_norm` | 제목 정규화(사례 30쌍) |
| `test_ux3_server` | 편의성 3묶음 서버(미리보기·규칙 목록 칸 등) |

## 주의

- **시험 파일을 한 프로세스에서 같이 돌리지 마십시오.** 각 파일이 import 때 `HUB_DB` 를 임시 경로로 정하는데(`"app" not in sys.modules` 일 때만), 여러 파일을 한 pytest 프로세스로 돌리면 먼저 import 한 파일의 DB 를 같이 쓰게 되어 엉킵니다 → **파일별로 따로 실행**.
- 화면 시험은 콘솔 한글 출력 때문에 `PYTHONIOENCODING=utf-8` 을 붙여 실행합니다(Windows).
- 운영 DB·실제 `hub.db` 에 대고 돌리지 마십시오(`AX_PG_URL` 은 시험이 지움).
- 화면 시험의 가짜 API 는 서버 응답 모양을 흉내 낸 것입니다. 서버 응답 모양을 바꾸면 `test_ui_tasks.py` 의 가짜 응답도 같이 고쳐야 합니다.
- 기본 `python` 에 `sqlalchemy` 등 `requirements.txt` 의존성이 있어야 `app` import 가 됩니다(없으면 `ModuleNotFoundError`).

---

**관련 문서**

- [배포·운영](deployment.md)
- [코드 위치·로컬 실행](code.md)
