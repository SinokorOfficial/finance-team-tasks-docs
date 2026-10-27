# 코드 위치·로컬 실행

어느 파일을 고치면 되는지, 내 PC에서 어떻게 띄워 보는지 정리합니다.


## 저장소·파일

저장소: `SinokorOfficial/ax-fnc`(비공개, 유일한 원본). 운영 접속 주소는 사내 인수인계 자료를 참고하십시오(공개판에는 싣지 않음).

| 파일 | 위치 | 역할 |
|---|---|---|
| `server/app.py` | 2176~3460행 「업무 현황(내장 기능)」 절 | 상수·권한 함수·검증·잠금·반복 생성·라우트 `/api/tasks*`·`/api/holidays*`·SSE 가공·기동 준비(`_tasks_boot`) |
| `server/app.py` | 그 밖 | `COLLS`(94행, 업무 현황 컬렉션 없음), `TASK_META_IDS`(1092행), `/api/stream` 의 업무 현황 분기(1392행 부근), `broadcast`→`tk_sse_mark`(436행), `PATCH /api/auth/me`(524행), `app_role`·`eff_app_role`·`app_role_unset`(1786~1820행), `LOCK_RESERVED_PREFIX`(1338행), `BUILTIN_APPS`(2204행) |
| `server/holidays.py` | 125줄 | 내장 공휴일 `BUILTIN`, 한국 시각 `KST`·`today_kst()`, 공휴일 합성 `holiday_set()`, 추가·삭제·되돌리기 |
| `server/store.py` | — | 문서 저장소(PostgreSQL/SQLite), `doc_put`(merge)·`acquire_lock`·`release_lock(key, holder)`·`session_create` |
| `hub.html` | 859행~ CSS(`.tk*`), 1913행 `<div id="v-tasks">` 보기, 2718·2747·2762행 창 3개(`tk-taskBack`·`tk-tpBack`·`tk-bkBack`), 6240~8058행 `window.tkHub` 모듈 | 화면 전체. **화면 수정은 이 파일에만** |
| `server/build_static.py` | 33줄 | `hub.html` → `server/static/index.html` 생성(SheetJS CDN → `/vendor/xlsx.full.min.js`, `shim.js` 삽입) |
| `server/static/index.html` | 생성본 | 직접 고치지 않음. 커밋은 함 |
| `server/test_tasks.py` | 1569줄 | 서버 시험(TestClient + 임시 SQLite) |
| `server/test_ui_tasks.py` | 2476줄 | 화면 시험(Playwright + 가짜 API) |
| `server/test_ui_events.py` 묶음 14 | — | 「일정에서 업무 만들기」 화면 시험(14-16~14-30) |
| `docs/tasks.md` | 406줄 | 기능 규약(권한·상태·반복·휴지통·동시성·SSE·API·엑셀) |
| `docs/portal-apps.md` | 「허브 안 기능 — builtin」 절 | 앱 등록표의 `builtin` 방식 |

## 로컬 실행

| 항목 | 내용 | 근거 |
|---|---|---|
| 의존성 | `server/requirements.txt`(fastapi·uvicorn·httpx·psycopg·openpyxl·sqlalchemy 등) | requirements.txt |
| 실행 | `server/` 에서 `python -m uvicorn app:app --port 8002`(머리말 원문은 `--host 0.0.0.0 --port 8002`) | app.py 머리말 |
| 저장소 | 환경 변수 `AX_PG_URL` 이 **없으면 SQLite**. 파일은 `HUB_DB`(기본 `server/hub.db`) | app.py 머리말, store.py 26행 |
| 로그인 | `AX_AUTH_URL`·`AX_AUTH_CLIENT_ID`(·`AX_AUTH_REDIRECT_URI`)를 비우면 ax 로그인이 꺼짐(`ax_login_enabled()` 거짓). 앱 자체 비밀번호 로그인은 없음(2026-09-29 삭제, 되살리지 말 것) | app.py 731행·머리말 |
| 세션 만들기 | 로그인이 꺼진 로컬에서는 `store.session_create(토큰, 이메일, 만료ISO)` 로 세션을 직접 만들고 브라우저(또는 시험 클라이언트) 쿠키 `hub_sid` 에 그 토큰을 넣음. 계정 문서는 `store.doc_put("accounts", 이메일, {...status:"approved", apps:{tasks:"member"}})` | test_tasks.py `cli()`·`reset()` |
| 주의 | 운영 DB(PostgreSQL)나 실제 `hub.db` 원본에 대고 시험·실험하지 말 것 | test_tasks.py 머리말 |

예시(로컬 SQLite, 임의 경로):

```
cd server
HUB_DB=/tmp/hub-local.db python -m uvicorn app:app --port 8002
```

세션 직접 만들기 예시(파이썬, 같은 `HUB_DB` 로):

```python
import os; os.environ["HUB_DB"] = "/tmp/hub-local.db"
import store, secrets; store.init()
store.doc_put("accounts", "user-a@company.example",
              {"email": "user-a@company.example", "name": "담당자 A", "role": "member",
               "status": "approved", "apps": {"tasks": "member"}}, who="dev")
tok = secrets.token_urlsafe(32)
store.session_create(tok, "user-a@company.example", "2099-01-01T00:00:00.000Z")
print(tok)   # 브라우저 개발자 도구에서 쿠키 hub_sid 에 넣음
```

- 위 예시의 쿠키 수동 주입 절차는 코드(시험)의 방식을 옮긴 것이고, 브라우저에서 실제로 해 본 기록은 **확인 필요**입니다.
- `hub.html` 을 고쳤으면 `python server/build_static.py` 로 `static/index.html` 을 다시 만들어야 로컬 서버에 반영됩니다.

---

**관련 문서**

- [화면 모듈](frontend.md)
- [시험](testing.md)
- [배포·운영](deployment.md)
