# 그립 스튜디오 2.0 라이브 콘솔 프로토타입 — 인수인계 / 컨텍스트 문서

> **목적:** 대화 컨텍스트가 리셋돼도 이 문서만 읽으면 작업을 그대로 이어갈 수 있도록 정리한 맥락 문서. (작성: 이건무 PM ↔ Claude, 2026-07-03)

## 0. 한 줄 요약
그립 스튜디오 2.0 "라이브 커머스 운영 콘솔"을 **역기획**하는 단일 HTML 인터랙티브 프로토타입. 화면 + 기획(PRD 모드) + 자동 데모를 한 파일에 담아 기획 검증·공유·개발 온보딩에 사용. 실제 지라 과제가 생길 때마다 이 프로토타입에 기능을 추가하고, PRD 모드/자동 데모에 등록하고, 필요 시 컨플루언스 PRD도 만든다.

## 1. 링크 · 저장소 · 배포
- **라이브(개인):** https://kmlee-grip-studio-proto.vercel.app  (Vercel 자동배포 — `git push`만 하면 반영)
- **라이브(회사):** https://pages.gripcorp.co/kmlee/grip-studio2-live-dashboard  (내부 GitHub Pages, 조직 로그인 필요)
- **개인 레포:** `Moowillbedone/kmlee-grip-studio-proto` (public). 캐논 소스 = `index.html` (+ `product.png`, `p1~p6.png` 상품 이미지).
- **회사 레포:** `gripcorp/product-kmlee` (kmlee-ai 계정). 배포 파일명 = `grip-studio2-live-dashboard.html` (index.html을 복사).
- **양쪽 동시 배포(항상):** `bash "/Users/grip_0195/Downloads/무제 폴더/deploy-grip-studio.sh" "커밋 메시지"`
  - 개인 push(자동 Vercel) + 회사 kmlee-ai 전환 push + Moowillbedone 복구까지 자동.
  - ⚠️ **회사 단계가 kmlee-ai keyring 인증 불안정으로 실패할 때가 있음.** 그러면 수동:
    `gh auth switch --user kmlee-ai` → `rm -rf /tmp/product-kmlee-deploy && gh repo clone gripcorp/product-kmlee /tmp/product-kmlee-deploy` → 거기로 `index.html→grip-studio2-live-dashboard.html`, `product.png`, `p*.png` 복사 → `git add -A && git -c user.name=kmlee-ai -c user.email=kmlee@gripcorp.co commit -m "..." && git push` → `gh auth switch --user Moowillbedone`.
- **로컬 프리뷰:** preview_start `portfolio-dashboard` (python http.server, `무제 폴더` 루트, 포트 8790) → `/kmlee-grip-studio-proto/index.html`. 리로드는 `?v=Date.now()` 붙여서.

## 2. 파일 구조 (전부 index.html 단일 파일)
- `<style>` 상단 = 디자인 토큰(:root, 브랜드 코랄 `--grip:#f5375b`) + 전 컴포넌트 CSS.
- 레이아웃: 좌측 GNB 사이드바(접기 가능, `.brand-collapse` « → 66px) → 방송 프리뷰 컬럼(`.col-broadcast`, 하단에 **판매 모드 토글** `.mode-toggle` 한정판매/스토어판매) → 판매 리스트(`.col-sales`) → 우측(`.col-right`) 실시간 성과 + **채팅 패널**.
- `<script>` = 순차 IIFE들: ①라이브 타이머 ②판매상태 라디오 ③GNB 접기 ④상품(체크박스/인라인 가격편집/일괄수정 모달/200원 검증) ⑤**UP+삭제+판매모드 IIFE** ⑥**채팅 IIFE** ⑦**PRD 모드 + 자동 데모 컨트롤러 IIFE**.

## 3. 핵심 아키텍처
### PRD 모드 = 기능 레지스트리 (컨트롤러 IIFE)
- `var FEATURES = []` 배열. 각 원소:
  `{ id, demoMode:'limited'|'store', jira:'GRIPPGM-xxxx', name, overviewHTML, flowHTML, prd:[{n,target,title,tag,def,asis?,tobe?,features:[]}], steps:[{tag,t,target,act}] }`
- 현재 **4개**: **price(GRIPPGM-3338)**, **delete(GRIPPGM-3357)**, **chat(GRIPPGM-3224)**, **qavis(GRIPPGM-3402, Q&A 라이브 노출 제어)**.
- PRD 패널 상단 **기능 스위처**(`#pfsBtn`/`#pfsMenu`)로 전환 → 개요·검증플로우·명세·화면마커(`.prd-marker`)·자동데모(steps)가 통째로 교체(`renderFeature()`).
- **새 과제 추가 = FEATURES 배열에 항목 하나 push** 하면 목록·마커·데모 자동 편입.
- 자동 데모(`#tourOverlay`): 스포트라이트(box-shadow) + 콜아웃. **rebuild(idx): 매 렌더 0단계부터 재구성**(좌/우 화살표·자동진행 어디서 오든 깨끗). 스텝 target은 **항상 보이는 요소**여야 함(숨겨지는 요소면 콜아웃이 0,0으로 날아감 — 실제 겪은 버그). placeSpot은 240/460/700ms 다중 배치(모달 트랜지션 대비).
- 마커 target은 PRD 모드에서 보이는 요소만(#poStrip·.mode-toggle는 방송컬럼이라 PRD모드에서 숨김 → 마커 X, 데모(풀스크린)에서만 사용).

### UP + 삭제 + 판매모드 IIFE
- **판매 모드:** `mode`='limited'(한정)|'store'(스토어). 좌측 하단 토글. `applyMode(m)`가 제목/안내문구/[UP한 상품 목록] 버튼 가시성/UP 정책/기본 UP 일괄 전환. `window.__gripSetMode` 노출.
  - 한정: UP **1개씩**(다른 상품 UP 시 이전 자동 해제), [UP한 상품 목록] 버튼 O, "1개씩 판매" 문구, 기본 UP 1개.
  - 스토어: UP **최대 5개**(6번째 팝업 `#upMaxModal`), 버튼 숨김, "최대 5개/판매종료·품절 자동해제" 문구, 기본 UP 3개.
- **UP:** `upOrder`(현재 UP, 프리뷰 노출) vs `everUp`(**한 번이라도 UP된 상품 전부** — UP한 상품 목록 모달용, 누적). 삭제 시 둘 다에서 제거. 품절 클릭 시 UP 자동 해제.
- **삭제:** 상품 행 🗑(`.pr-del`) → 확인 모달(`#deleteModal`) → 제거 + 토스트. UP 목록/프리뷰 자동 반영(한정). 비즈센터 미반영.
  - **판매 모드별 확인 문구(2026-07-03 추가):** `askDelete()`가 `#delDetail`을 모드별로 세팅. **한정판매**=“UP한 상품 목록에서도 제거” 문구 포함 / **스토어판매**=제외(스토어엔 [UP한 상품 목록] 버튼 자체가 없음 — `applyMode`가 store에서 `btnUpList` 숨김). “비즈센터 유지” 문구는 두 모드 공통. PRD 명세 **4개**(1삭제/2**모드별 문구**/3UP동기화(한정)/4비즈센터), 플로우 “현재 판매 모드는?” 분기, 데모에 **스토어 분기+정리 스텝** 추가.
- 상품 이미지: `p1~p6.png`(다운로드 오늘자 PNG 복사). 프리뷰 가로 카드 `#poStrip`(scroll-snap), UP한 상품 목록 모달 `#upListModal`(전체선택/체크/스토어 ON·OFF).
- 데모 리셋용 `window.__gripRestoreRows`, `window.__gripSetMode` 노출.

### 채팅 IIFE (실제 운영 화면 수준)
- 실시간 스트림(유저10+매니저"모니터를좋아해", 2.4초, 결정적 유사난수). 파란 유저닉/레드 매니저. 메시지 박스형.
- 좌측 프리뷰 채팅 오버레이 `.ov-chat`(최근 4줄) + **Q&A 띠지 `#ovQa`**. `.ov-bottom` 세로 스택으로 프리뷰 하단 요소 겹침 방지(채팅→Q&A띠지→상품카드).
- **닉네임 검색:** 채팅탭 상단 🔍(`#chatSearchBtn`) → like 자동완성(`#chatAC`, 건수) → 선택/Enter → 해당 유저만 + "메시지 N건"(`#chatFilterStrip`) → 검색 닫기 시 실시간 복귀(최신 자동 스크롤). 닉네임 클릭으로도 필터. (**'전체 채팅 +N' 배너는 삭제함**.)
- **⋮ 메뉴(유저 메시지):** 방송 끝까지 음소거 / 5분 음소거 / 답변하기.
  - 음소거: `muted` 맵 → 해당 유저 메시지 숨김 + 토스트.
  - **답변하기 → 인라인 답변 박스(`#chatReply`)**: 대상 질문 + 답변 입력 + [답변하기] → `registerQA(m,a)`로 질문+답변 동시에 Q&A 등록.
- **Q&A 탭:** `data-tab=qa` `#qaPanel`. Q/A 표시, ⋮로 답변하기(인라인)/삭제하기, 탭 알림점 `#qaDot`. Q&A 등록 시 프리뷰 `#ovQa` 띠지 노출.
- ⚠️ **채팅 IIFE는 `toast(msg,ok)` 헬퍼를 자체 정의**해야 함(`window.__gripToast` 래핑). 없으면 음소거 등 핸들러가 ReferenceError로 중단(실제 겪은 버그).
- `window.__gripChatReset`로 데모 리셋(음소거·Q&A·필터·검색 초기화).

## 4. 컨플루언스 PRD
- 공간 spaceId **4030468**, 부모 페이지 **2306867212**("[스튜디오] 과제 모음").
- 페이지: 가격 **2306113587**(3338) / 삭제 **2319581187**(3357) / 채팅 **2319515671**(3224) / **Q&A 노출범위 2320039942**(3402).
- 양식(가격 PRD 기준): TOC 매크로 → info 패널(프로토타입 링크) → 문서정보(Jira 매크로 포함) → 문서 히스토리 → 1.배경목적 2.KPI 3.용어 4.ASIS/TOBE 5.기능요구 6.검증규칙·수용기준 7.검증플로우(**플로우차트 이미지**) 8.화면UX 9.영향범위.
- **Jira 매크로:** `<ac:structured-macro ac:name="jira"><ac:parameter ac:name="server">System Jira</ac:parameter><ac:parameter ac:name="serverId">174ac8ba-9990-3d59-be10-643a4aa0d945</ac:parameter><ac:parameter ac:name="key">GRIPPGM-xxxx</ac:parameter></ac:structured-macro>`
- **플로우차트 이미지:** 로컬에 SVG변환기 없음 → **PIL로 PNG 렌더**(`draw_flow.py`, 한글폰트 `/System/Library/Fonts/AppleSDGothicNeo.ttc`, 시작/과정/결정/분기/최종 노드+화살표, 브랜드 코랄). 페이지에 `flow.png` 첨부 → 본문 `<ac:image ac:width="780"><ri:attachment ri:filename="flow.png"/></ac:image>`.
  - 첨부 신규: `POST /wiki/rest/api/content/{id}/child/attachment` (multipart, 헤더 `X-Atlassian-Token: no-check`).
  - **기존 첨부 갱신(같은 파일명 신규 POST는 400): `POST /wiki/rest/api/content/{id}/child/attachment/{attId}/data`**.
  - 이미지 "Preview unavailable" 뜨면 → 첨부 존재 상태로 **본문 재저장(PUT version+1)** 하면 재렌더됨.
- 페이지 생성: v2 `POST /wiki/api/v2/pages` {spaceId, status:current, title, parentId, body:{representation:storage, value}}.
- **Atlassian API 토큰:** 사용자(이건무, kmlee@gripcorp.co)가 발급해 제공. Basic 인증(email:token base64). JIRA·Confluence 공통. **세션마다 사용자에게 다시 받아야 함**(시크릿, 저장 안 함). 스크립트들은 scratchpad에 있고 `ATLASSIAN_API_TOKEN` env로 받음.

## 5. 지금까지 만든 기능 (시간순)
가격 3단(정가/상시할인가/라이브가) 인라인+일괄 수정 & 200원 검증(3338) → 상품 삭제 + UP목록 동기화(3357) → UP 노출(최대5·순서·프리뷰 카드) → 판매모드 토글(한정/스토어) → 채팅(스트림·검색·음소거·답변/Q&A) → UP목록=한번이라도 UP된 상품 누적 → 채팅 답변 인라인화·프리뷰 겹침수정·배너삭제 → 컨플루언스 PRD 2건(삭제·채팅) + 플로우차트.
(상세는 메모리 `project_grip_studio.md` 참조 — ~/.claude/projects/.../memory/)

## 6. GRIPPGM-3402 — 채팅 답변 시 Q&A 미노출 ✅ (구현·배포 완료 2026-07-03)
**구현됨(최종 정책 — 본인만 보기, 사후 전환 제거):** 답변 인라인 박스(`#chatReply`)에 **"본인만 보기" 토글(`#crLiveBox`, 기본 OFF)**. `registerQA(m,a,priv)`에 `priv` 플래그. **OFF(기본)=전체 공개**(모든 유저 Q&A + 방송 프리뷰 띠지 노출), **ON=본인만**(문의한 본인에게만, 다른 유저·방송 미노출). **노출 범위는 답변 시점에만 결정** — Q&A 탭 항목의 범위 배지는 **정적 표시** `.qa-vis`(🌐 전체 공개 / 🔒 본인만), 클릭 토글 아님(사후 전환 기능은 사용자 요청으로 제거). `updateOvQA()`는 **priv=false(전체 공개)인 최신 항목만** 띠지 표시. PRD 기능4 "qavis"(GRIPPGM-3402, 이름 "…Q&A 노출 범위(전체/본인만)") — **명세 1·2번만**(1:답변 시 노출범위 토글, 2:본인만=다른유저 미노출), **3번 "Q&A 탭 사후 전환" 삭제**. 데모(시작→답변작성→노출범위 토글→전체공개 기본→본인만). 검증: 기본→띠지 ON·"🌐 전체 공개", 본인만→띠지 X·"🔒 본인만", 배지는 정적(클릭 무반응). (이력: "라이브 노출 on/off"→"전체 공개 기본/본인만 토글"→"사후 전환 제거·답변 시점 결정"으로 단순화.) **컨플루언스 PRD 생성 완료(2026-07-03): page 2320039942**(부모 2306867212, 발행 전 4관점 적대적 검증 통과, 플로우차트 이미지 렌더 확인). (삭제 PRD 2319581187도 판매모드 정책 반영해 v9로 갱신 완료 — 본문+플로우차트.)
<details><summary>(원래 계획 — 참고용)</summary>
- **ASIS:** 클릭메이트 등 그립 무관한 문의에 매니저가 실수로 답변하면 그 Q&A가 **라이브 방송에 고정 노출**됨.
- **사용자 TOBE 아이디어:** 1) 답변 시 "Q&A 노출 안하기" 토글(작성 시점부터 노출 X) 2) "본인만 보기"(문의자에게만).
- **Claude 권장안(더 나음 — 적용 예정):** "답변 작성"과 "라이브 노출"을 **분리**. 답변 인라인 박스(`#chatReply`)에 **"라이브에 노출" 토글(기본 OFF)** 추가.
  - OFF(기본): 답변이 **Q&A 탭(매니저 내부 기록)에만** 저장, 라이브 방송 띠지엔 노출 X → 실수해도 라이브에 안 뜸(default-safe).
  - ON: 라이브 Q&A 띠지(`#ovQa`)에 노출.
  - Q&A 탭 각 항목에 **라이브 노출 뱃지/토글** → 검토 후 게시/게시취소 가능.
  - 프리뷰 `#ovQa` 띠지는 **라이브 노출된 항목만**(가장 최근) 표시. 없으면 숨김.
  - 근거: 사용자 아이디어1은 기본이 노출ON이라 실수 여지 남음 → **기본 OFF(노출은 의도적 선택)**가 실수 원천차단 + 게시취소까지 되어 UIUX 우위. (아이디어2 "본인만 보기"=1:1 답장은 옵션으로 언급)
- **구현 위치:** 채팅 IIFE의 `registerQA(m,a)` → `registerQA(m,a,live)`. 인라인 박스에 토글 추가. Q&A 항목 렌더에 라이브 토글. `updateOvQA()`는 live===true인 최신 항목만.
- **추가:** PRD 모드 4번째 기능 "[스튜디오][채팅] Q&A 라이브 노출 제어"(GRIPPGM-3402) + 자동 데모. (컨플루언스 PRD는 요청 시)

</details>

## 7. 자주 쓰는 검증/배포 루틴
1. index.html 편집(문자열 치환 스크립트 or Edit).
2. `python3 -c` 로 `<script>` 추출 후 `node --check` (JS 문법).
3. preview_eval로 실제 동작 검증(리로드 `?v=`), preview_console_logs 에러 0 확인.
4. `deploy-grip-studio.sh` 로 양쪽 배포(회사 실패 시 수동 폴백).
5. 라이브 curl로 반영 확인.
