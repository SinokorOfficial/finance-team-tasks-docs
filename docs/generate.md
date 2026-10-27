# 반복 규칙 생성

반복 규칙에서 업무가 어떻게 자동으로 만들어지는지(`generate`·`_generate`) 정리합니다.


| 단계 | 내용 |
|---|---|
| 잠금 | `tasks:gen`(TTL 60초). 목록 조회는 대기 0초(바쁘면 `{busy}` 로 생성 없이 넘어감), 규칙 등록·[규칙에서 생성]·생성 건 복원은 5초 대기 후 503 |
| 대상 규칙 | `active` 가 거짓이 아니고, 담당자가 업무 현황 admin·member 인 승인 계정(`tpl_owner_ok`), [규칙에서 생성]은 요청자가 볼 수 있는 규칙만 |
| 읽기 순서 | **업무 목록 먼저 → `taskSkips` 나중**(삭제가 「skip 기록 → 문서 삭제」 순이라 둘 중 하나에는 반드시 보임) |
| 달 반복 | 한국 오늘의 달부터 n달, 규칙의 `months` 에 든 달만 |
| id | `g_{규칙id}_{YYYYMM}` 고정 → 이미 있으면 `exists`(덮어쓰지 않음) |
| 지운 기록 | `taskSkips` 에 있으면 건너뜀: 값이 `past:` 로 시작하면 `skippedPast`, 아니면 `skippedDeleted` |
| 기한 계산 | `due_of(y, m, dueDay, shift, hol)` — 말일 또는 그 날(달 길이로 자름). `shift` 가 back/fwd 이고 주말·공휴일이면 한 칸씩 이동, **달을 벗어나면 방향을 한 번 뒤집음**. 공휴일 = 내장 − 제외 + 추가 |
| 지난 기한 | `skip_past`(규칙 등록 직후 생성만)이면 이번 달 기한 < 오늘인 건은 만들지 않고 `taskSkips` 에 `past:` 기록 |
| 겹침 판정 | `find_manual_dup` — 같은 담당자·같은 기한 달의 손 등록 업무(`gen` 아님) 중 `tasks_dup`(회사·구분이 둘 다 있으면 같아야 함 + `titles_similar`). 팀 규칙은 팀 업무와만, 개인 규칙은 본인의 팀·개인 업무와 견줌. 겹치면 만들지 않고 `dups` 에 담음(**`taskSkips` 에 넣지 않음**) |
| 겹쳐도 만들기 | `forceIds`(gid 목록, 200개까지, 모양 `g_(.+)_(\d{6})`): 각 gid 의 규칙이 요청자 규칙(담당자·만든 사람)이거나 요청자가 admin 이어야 함 — 아니면 403, 아무것도 안 만듦. `force:true`(=`includeDups`): 그 기간 전부, admin 만 |
| 만들 문서 | 규칙 칸 복사 + `templateId`·`gen:true`·`status:"예정"`·`createdBy`(규칙 만든 사람 + 「 (규칙 생성)」)·이력 「등록 (규칙 생성)」, `startBefore` 가 있으면 `startDate` |
| 응답 가공 | `dups` 는 `gen_dups_for` 로 요청자에게 알릴 것만(본인 규칙 또는 admin, 규칙·겹친 업무 둘 다 볼 수 있을 때). `rules` = `rule_counts`(`total`·`mine`·`noOwner`·`inactive`) |

## 제목 비슷함(`norm_title_parts`·`titles_similar`)

- 정규화(비교에만 사용): NFKC·소문자 → 구두점·`_` 를 띄어쓰기로 → **앞머리만** 연도(`2026년`·`26년`·`2026 `+N월)·`N월`(+분/말/별, 띄어쓰기 앞의 차)·`월별/월말/월간/매월/매달`·`N분기/N반기`의 숫자 제거 → `분기별/반기별` → `분기/반기` → 공백 제거. 걷어낸 연·월·분기 숫자는 따로 기억.
- 비슷함: 양쪽 다 날짜 표지가 있으면 같아야 함 / 짧은 쪽 열쇠 6자 이상 / 같거나, 짧은 쪽이 긴 쪽에 들어 있고 나머지가 날짜 조각(`[0-9년월일분말]`)뿐이며 나머지 길이 ≤ 짧은 쪽의 절반.
- 「놓치는 쪽」을 택한 보수 규칙입니다. 사례 30쌍이 `test_tasks.py` 19-9 에 기대값으로 고정되어 있습니다.

## 미리보기·규칙 목록 칸

- `tpl_upcoming`: 한국 오늘부터 앞으로 올 기한 n건(기한 = 오늘이면 포함, 해 넘김 포함). `tpl_occurrence` 가 `due`·`base`(옮기기 전)·`shifted`·`weekday`·`reason`(「M.D 토요일이라 뒤로」, 공휴일이면 이름, 방향이 규칙과 반대면 「(달을 넘기지 않게)」).
- `tpl_skipped_this_month`: 이번 달이 규칙 달이고 기한이 지났으면 `{base, due, reason:"이미 지난 기한"}`.
- 규칙 목록 `next` = `tpl_upcoming(…,1)[0].due`(사용 중지면 null), `made` = 그 규칙 `templateId` 를 가진 볼 수 있는 업무 수(휴지통 제외), `ownerOk` = `tpl_owner_ok`.

## 시작일 자동 진행(`auto_start`)

- 조건(`_auto_due`): `startDate` 있음 · `startDate ≤ 한국 오늘` · 상태 '예정' · `autoStarted` 아님.
- 문서마다 잠금을 대기 0초로 잡고(못 잡으면 건너뜀) **다시 읽어** 조건 확인 → merge 로 `status:"진행"`·`autoStarted:true`·이력(by 「시스템」).
- 도는 때: `GET /api/tasks`(생성 직후)·`GET /api/tasks/export.xlsx`. 예약 작업 없음.
- 담당자 변경 때: `_tasks_edit` 가 같은 조건이면 그 저장에서 바로 '진행'(이력 한 줄).

---

**관련 문서**

- [데이터 모델](data-model.md)
- [동시성·충돌](concurrency.md)
- [알려진 한계](limits.md)
