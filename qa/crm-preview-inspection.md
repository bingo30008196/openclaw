# CRM Preview 로컬 Chrome 검수 지시서

로컬 Claude Code + `chrome-devtools-mcp`로, **이미 열려 있는 CRM Preview 탭**을 검수하기 위한 절차서.
이 리포 루트의 `.mcp.json`이 `chrome-devtools` 서버를 자동 등록하며, `--browserUrl=http://127.0.0.1:9222`로 **기존 Chrome에 attach**한다 (새 브라우저를 띄우지 않음).

---

## 0. 사전 준비 (사람이 직접 수행)

1. Chrome **완전 종료** (작업 표시줄/트레이 잔여 프로세스 포함. 이미 켜진 프로세스에는 나중에 포트를 열 수 없음).
2. 원격 디버깅 포트로 재실행:
   - Windows: `"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222`
   - macOS: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --remote-debugging-port=9222`
3. 그 창에서 **CRM Preview 탭을 다시 열기**.
4. 이 리포 루트에서 로컬 Claude Code 실행 → `.mcp.json` 로드 확인 (`/mcp`에서 `chrome-devtools` connected).
   - 프로젝트 설정 대신 사용자 전역 등록을 원하면: `claude mcp add chrome-devtools -s user -- npx -y chrome-devtools-mcp@latest --browserUrl=http://127.0.0.1:9222`

## 1. 연결 확인 (검수 시작 게이트)

- `list_pages`로 탭 목록 조회 → CRM Preview 탭이 보이는지 확인 후 `select_page`.
- 탭이 안 보이면 **검수 불가로 중단**하고 0번 절차 재확인을 요청한다. `new_page`로 새 탭/새 브라우저를 띄워 대체하지 않는다 (검수 대상은 기존 탭).

## 2. 검수 매트릭스

두 뷰포트에서 아래 A–F를 각각 수행: `resize_page`로 **1440×900 (데스크톱)** → 전체 수행 → **390×844 (모바일)** → 전체 수행.

- **A. 렌더링**: 풀페이지 스크린샷. 레이아웃 깨짐, 요소 겹침, 잘림, 의도치 않은 가로 스크롤 여부.
- **B. 포인터 클릭**: 주요 인터랙티브 요소(내비게이션, 리스트 행, 버튼, 탭)를 `click`. 반응(화면 전환/상태 변경)과 hover 상태를 확인.
- **C. 휠 스크롤**: 페이지 최상단→최하단→최상단 휠 스크롤. 스크롤 중 스티키 헤더/고정 요소 유지, 스크롤 끊김·점프 여부.
- **D. 키보드**: `Tab` 순회(포커스 링이 시각적으로 보이는지, 순서가 논리적인지), `Enter`로 포커스된 액션 활성화, `Home`/`End`로 문서 점프. 전용 키 입력 툴이 없으면 `evaluate_script`로 KeyboardEvent를 디스패치해 확인하고, 그 사실을 증거에 명시.
- **E. 콘솔/예외**: `list_console_messages`로 error/warning 수집. uncaught exception(pageerror) 여부 확인.
- **F. 네트워크**: `list_network_requests`로 실패 요청(4xx/5xx/blocked/aborted) 수집. **쓰기 요청(POST/PUT/PATCH/DELETE) 발생 여부**를 반드시 확인 — Preview는 읽기 전용이어야 하므로, 쓰기 요청이 실제 서버로 전송되면 그 자체가 결함. 차단(blocked/미발생)되는지 확인.

주의: 검수 중 실데이터를 변경할 수 있는 액션(저장/삭제/전송 버튼)은 **클릭 직전에 멈추고** 네트워크 로그로 사전 요청 여부만 확인한다.

## 3. 증거 수집

- 스크린샷: `crm-<뷰포트>-<항목>-<순번>.png` (예: `crm-1440-A-01.png`). 결함 발견 시 해당 상태 스크린샷 필수.
- 콘솔/네트워크: 메시지·요청 원문(메서드, URL, 상태코드)을 그대로 기록. 요약만 남기지 않는다.
- 결과 테이블: 항목(A–F × 뷰포트 2) 별 PASS / WARN / FAIL + 증거 파일/로그 참조.

## 4. 판정 규칙

- **FAIL**: uncaught pageerror · `console.error` · 자사 요청 5xx 또는 실패 · 쓰기 요청이 서버로 실제 전송됨 · 레이아웃 깨짐/주요 콘텐츠 잘림 · 키보드만으로 주요 액션 불가 또는 포커스 링 없음.
- **WARN**: `console.warning` · 예상된 4xx(예: 미인증 401) · 서드파티(analytics/광고) 요청 실패 · 시각적 minor 이슈.
- **PASS**: 위 어디에도 해당 없음. 항목별 판정이며, FAIL 1건이라도 있으면 전체 판정 FAIL.
- 판정 불가(연결 실패, 탭 소실 등)는 FAIL이 아니라 **BLOCKED**로 구분해 보고.

## 5. 로컬 세션에 붙여넣을 지시문

```text
qa/crm-preview-inspection.md의 검수 지시서를 따라 CRM Preview를 검수해줘.
- chrome-devtools MCP로 이미 열린 CRM Preview 탭에 attach (list_pages → select_page). 새 탭/새 브라우저 금지, 실패 시 BLOCKED로 중단 보고.
- 1440×900과 390×844 두 뷰포트에서 A(렌더링)–F(네트워크) 전 항목 수행.
- 포인터 click, 휠 스크롤, Tab/Enter/Home/End 키보드 확인 포함.
- console error/warning, pageerror, 실패 요청, 쓰기(POST/PUT/PATCH/DELETE) 요청 차단 여부를 원문 증거와 함께 수집.
- 데이터를 변경하는 버튼은 클릭하지 않는다.
- 결과는 항목×뷰포트 PASS/WARN/FAIL 테이블 + 증거(스크린샷 경로, 로그 원문) + 최종 판정으로 보고.
```
