# 화면 모듈

`hub.html` 안의 업무 현황 모듈(`window.tkHub`) 구조와 포털과의 경계를 정리합니다.


- `hub.html` 6240행 `window.tkHub = (() => { … })();` 즉시 실행 함수 하나로 감싸 이름(`isAdmin`·`people`·`STATUSES` 등)이 포털과 섞이지 않게 했습니다.
- 접두: DOM id 는 `tk-`(모듈 안에서 `T("x")` = `document.getElementById("tk-x")`), CSS 클래스는 `tk`(`.tkRow`·`.tkBtn`·`.tkBack` 등). 보기 컨테이너 `#v-tasks`, 창 3개 `#tk-taskBack`(업무)·`#tk-tpBack`(반복 규칙)·`#tk-bkBack`(엑셀 일괄).
- 서버 호출은 모듈 안 `tapi(method, path, data)`(세션 쿠키, 실패 시 `Error` 에 `status`·`body`). 엑셀 미리보기만 `fetch` 로 xlsx 원문(`application/octet-stream`) 전송. 양식·내보내기는 `<a href download>`.
- 한국 날짜: 화면도 `Intl.DateTimeFormat("sv-SE", {timeZone:"Asia/Seoul"})` 로 '오늘'을 셈(브라우저 시간대 무관).

## 공개 함수(모듈이 돌려주는 객체)

| 이름 | 내부 함수 | 포털이 부르는 곳 | 하는 일 |
|---|---|---|---|
| `allowed()` | `allowed` | 보기 전환(3052행)·일정 달력·`/go` 처리 | 로그인 + builtin 등록 + 업무 현황 권한 |
| `onMe()` | `syncMe` | 계정 문서 갱신·로그인·로그아웃(5498·5514·5549행) | 사람·권한 반영, 사람이 바뀌면 상태 초기화·「내 업무만」 기억 다시 읽기, 메뉴 탭 숨김 |
| `onApps()` | `onApps` | `/api/apps` 받은 뒤(5614행) | `syncMe` + `#tasks` 로 들어온 경우 이동 |
| `onView(name)` | `onView` | `gotoView` 마다(3061행) | 들어오면 첫 로드·미뤄 둔 갱신, 나가면 창·띠·메뉴 닫기 |
| `dirty()` | `anyDirty` | 뒤로가기 처리(3112행) | 고친 내용이 있는가(요약 보기는 늘 거짓) |
| `confirmLeave()` | `confirmLeave` | 뒤로가기 처리(3114행) | 버릴지 묻고 창 닫기 → 참/거짓(Promise) |
| `canCreate()` | `canCreate` | 일정 상세 창 [업무로 만들기] 표시(5199·5223행) | `allowed()` + admin·member |
| `newFrom(pre, {replace})` | `newFrom` | 일정 상세 창(5228행) | 업무 현황으로 옮겨 새 업무 창을 열고 회사·업무명·기한·메모 채움(최대 10초 대기) |
| `open(id)` | `openFrom` | 일정 달력 ◆(4790행) | 업무 현황으로 가서 그 업무 창 열기 + 「현황에서 보기 →」 |
| `dueList(from, to)` | `dueList` | 일정 달력 「업무 기한 표시」(4777행) | 미완료 업무 `[{id,title,dueDate,ownerName,status}]`, 아직 안 받았으면 `[]` + 30초에 한 번 받기 |
| `showCal()` | `showCal` | 일정 달력 「업무 캘린더 →」(4792행) | 업무 현황 캘린더 탭으로 |
| `_state()` | — | 시험용(읽기만) | 내부 상태 요약 |

모듈 → 포털 사건: 목록을 받으면 `window` 에 `tk:tasks` 를 알림(일정 달력 다시 그리기).

## 포털 공용 함수 의존(모듈이 쓰는 것)

`$` · `esc` · `toast` · `ask`(모듈 안에서 `tkAsk` 로 옵션 이름만 바꿔 씀) · `gotoView` · `currentView` · `me`(계정 문서) · `myApps`(`/api/apps` 결과). 원칙: 모듈은 이것들을 **쓰기만** 하고 포털 쪽 코드를 고치지 않으며, 포털은 `window.tkHub.*` 공개 함수로만 모듈을 부릅니다. `shim.js` 는 고치지 않습니다(SSE 는 `EventSource` 감싸기로 엿들음).

## 브라우저 저장(`localStorage`)

| 키 | 값 | 쓰는 곳 |
|---|---|---|
| `tk.mine.<이메일 소문자>` | `"1"`/`"0"` | 현황 「내 업무만」(사람마다 따로) |
| `tk.calHidden` | JSON 배열(숨긴 담당자 이메일) | 캘린더 「담당자 표시」 |
| `tk.introSeen` | `"1"` | 휴대폰(폭 468px 이하)에서 처음 한 번만 소개 문단 펼침 |
| `evTaskTgl` | `"1"`/`"0"` | 포털 일정 달력 「업무 기한 표시」(일정 쪽 코드, 모듈 밖) |

그 밖의 펼침 상태(진행 중 「나머지 n건」, 기간별 「위에 이미 나온 n건」 등)는 저장하지 않고 화면 동안만 기억합니다.

---

**관련 문서**

- [코드 위치·로컬 실행](code.md)
- [실시간(SSE)](sse.md)
- [시험](testing.md)
