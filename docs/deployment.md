# 배포·운영

고친 코드를 운영에 올리는 순서와 CI 검사 항목, 운영 중 점검을 정리합니다.


| 단계 | 내용 |
|---|---|
| 1 | 개인 가지에서 작업·커밋 |
| 2 | 개인 가지 → `develop` PR. **develop 에 push 만으로는 CI 가 돌지 않음**(deploy.yml 트리거는 main push·main 대상 PR·수동) |
| 3 | 독립 검증(작업자와 다른 검증자가 시험·코드 확인) |
| 4 | `develop` → `main` PR — 이때 CI 검증 단계가 돎 |
| 5 | main 병합 = **운영 배포**(GitHub Actions, 이미지 태그 = 커밋 SHA) |
| 6 | 배포 확인: 화면은 **Ctrl+F5**(강력 새로고침)로 새 `index.html` 받기. 상태는 `GET /api/health` |

- 「독립 검증」(팀 운영 규칙, 저장소 코드에는 없음): develop → main 전에 작업한 사람과 **다른 검증자**가 깨끗한 `git clone` 사본에서 전체 시험을 돌리고 「개인 PC 경로·API 키·실명 0건」을 확인한 뒤 「배포 가능」 판정을 남깁니다(보고서는 작업 기록 폴더에 verify 번호로). 판정 전에는 main 에 합치지 않습니다.
- main 병합(= 운영 배포)은 자동이 아니라 사람이 GitHub 에서 「Merge pull request」를 누릅니다. 합쳐진 개인 가지는 바로 지우고, 상시 가지는 `develop`·`main` 둘만 둡니다.
- `.github/workflows/*` 는 배포 파이프라인이므로 기능 PR 에 섞지 않습니다.
- `/api/health` 는 옛 리비전도 200 을 주므로 배포 반영 여부 판정에는 부족합니다. 워크플로 「배포 검증」 단계가 서빙 중인 리비전의 이미지 태그를 커밋 SHA 와 대조합니다(architecture.md).

## CI(`.github/workflows/deploy.yml`)가 검사하는 것

| 단계 | 내용 | PR | main push |
|---|---|---|---|
| 파이썬 문법 검사 | `python -m py_compile server/*.py`(파이썬 3.12) | 돎 | 돎 |
| 화면↔생성본 대조 | `python server/build_static.py` 를 다시 돌려 커밋된 `server/static/index.html` 과 다르면 실패 | 돎 | 돎 |
| 이미지 빌드 | `docker build` | 돎 | 돎 |
| 기동·헬스체크 | 컨테이너를 띄워 30초 안에 `/api/health` 응답 | 돎 | 돎 |
| Azure 로그인(OIDC)·이미지 푸시·리비전 교체·배포 검증 | 저장소 변수 5개가 모두 있을 때 | **안 돎** | 돎 |

- **CI 는 `test_tasks.py`·`test_ui_tasks.py` 를 돌리지 않습니다.** 시험은 사람이 직접 돌려야 합니다([시험](testing.md)).
- `hub.html` 을 고치면 반드시 `python server/build_static.py` 를 돌려 `server/static/index.html` 을 **함께 커밋**해야 합니다. 안 하면 CI 의 대조 단계에서 실패합니다. `static/index.html` 을 직접 고치지 마십시오(빌드가 덮어씀).

## 운영 중 점검

| 점검 | 방법 |
|---|---|
| 서버 상태 | `GET /api/health` → `{ok, store:"postgresql", …}` |
| 앱 등록 | 계정 관리 > 하위 메뉴에 「업무 현황(허브 안 기능)」이 있고 사용 중인지 |
| 기동 로그 | `[tasks] 앱 등록표에 내장 항목을 만들었습니다` / `[tasks] 기동 준비 실패(처음 부를 때 다시)` / `[tasks] 잠금 풀기 실패(TTL 로 만료됨)` |
| 공휴일 | 매년 말 다음 해 공휴일 입력(관리 탭) — 2028년부터 내장 없음 |

---

**관련 문서**

- [시험](testing.md)
- [코드 위치·로컬 실행](code.md)
- [알려진 한계](limits.md)
