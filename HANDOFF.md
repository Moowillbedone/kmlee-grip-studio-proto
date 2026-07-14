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
- **판매 모드:** `mode`='limited'(한정)|'store'(스토어). 좌측 하단 토글. `applyMode(m)`가 제목/안내문구/[UP한 상품 목록] 버튼 가시성/UP 정책/기본 UP 일괄 전환 + `.sales-card`에 `is-store` 클래스 토글. `window.__gripSetMode` 노출.
  - **모드별 툴바(`.select-row .store-toggle`, 2026-07-03):** 공통 `[가격 수정][선택 삭제]`. **한정판매**=+`[스토어 ON/OFF]`(`.mode-limited-only`). **스토어판매**=`[UP 상품만 보기]` 토글(`#btnUpOnly`, `.uponly-toggle`, `.sales-card.is-store`일 때만 노출) — ON 시 UP 안 된 행 `display:none`(`applyUpOnly()`, setUp/resetUpState에서 재적용). 스토어ON/OFF는 store에서 숨김.
  - 한정: UP **1개씩**(다른 상품 UP 시 이전 자동 해제), [UP한 상품 목록] 버튼 O, "1개씩 판매" 문구, 기본 UP 1개.
  - 스토어: UP **최대 5개**(6번째 팝업 `#upMaxModal`), 버튼 숨김, "최대 5개/판매종료·품절 자동해제" 문구, 기본 UP 3개.
- **UP:** `upOrder`(현재 UP, 프리뷰 노출) vs `everUp`(**한 번이라도 UP된 상품 전부** — UP한 상품 목록 모달용, 누적). 삭제 시 둘 다에서 제거. 품절 클릭 시 UP 자동 해제.
- **삭제:** (2026-07-03 변경) 행별 🗑(`.pr-del`) **제거** → **체크박스 선택 + `[선택 삭제]`(`#btnDelSel`)** 방식. `#btnDelSel`은 체크박스 IIFE `actionBtns`에 포함돼 1개+ 선택 시 `.enabled`(코랄 `danger`). 클릭 → `askBulkDelete(window.__gripCheckedRows())` → 확인 모달(`#deleteModal`) → `[제거]` → `pendingRows.forEach(doRemove)` + `window.__gripSelRefresh()`. 체크박스 IIFE는 `liveRows()`(isConnected)로 삭제 후 카운트 보정, `__gripSelRefresh`/`__gripCheckedRows` 노출. UP 목록/프리뷰 자동 반영(한정).
  - **삭제 연동 정책(2026-07-03 정정 — 중요):** 토스트=“N개 상품을 삭제했습니다”(단순). 모달/토스트/PRD에서 **“비즈센터 상품은 유지” 문구 전면 삭제**. 정정된 정책: 스튜디오 삭제 → **비즈센터 ‘라이브 방송 상품 설정’ 영역에서도 함께 제거(연동 O)**, 단 **비즈센터의 실제 상품 목록(상품 마스터)은 삭제 안 됨(연동 X)**. ‘라이브 방송 상품 설정’ ≠ 비즈센터 상품 목록(별개 영역). (이전엔 “라이브 방송 상품 설정 미반영”으로 **틀리게** 적혀 있었음 → 뒤집음.) PRD 명세4 제목 “비즈센터 미반영”→“삭제 연동 범위(라이브 방송 상품 설정)”. **가격 3338의 비즈센터=가격 데이터 연동은 별개·정상(유지).**
  - **판매 모드별 확인 문구:** `setDelDetail()`이 `#delDetail`을 모드별로 세팅. **한정판매**=“스튜디오 방송 리스트와 UP한 상품 목록에서 제거” / **스토어판매**=“스튜디오 방송 리스트에서 제거”(UP 목록 문구 없음). 비즈센터 언급 없음. PRD 명세 **4개**(1선택삭제/2**모드별 문구**/3UP동기화(한정)/4**삭제 연동 범위**), 플로우 “현재 판매 모드는?” 분기, 데모에 스토어 분기+정리 스텝. 컨플루언스 삭제 PRD 2319581187 **v14**(ver0.4).
- 상품 이미지: `p1~p6.png`(다운로드 오늘자 PNG 복사). 프리뷰 가로 카드 `#poStrip`(scroll-snap), UP한 상품 목록 모달 `#upListModal`(전체선택/체크/스토어 ON·OFF).
- 데모 리셋용 `window.__gripRestoreRows`, `window.__gripSetMode` 노출.

### 상품 옵션 관리 모달 IIFE (신규 과제, 2026-07-10 · 진행중)
- **과제:** "[스튜디오] 상품 옵션 관리 탭 내 가격·재고·옵션명 일괄 수정"(지라 미정). **1~3단계 완료**(옵션 버튼→모달→옵션명 수정), **4단계(가격·재고 일괄 수정) 남음**.
- 상품 행 `.pr-actions`에 **`[옵션]` 버튼(`.btn-option`, [UP] 왼쪽)** — 전 행/양 모드. 클릭(위임) → **`#optionModal`**(상품 옵션 관리): 상품명(클릭 행에서), 판매중 옵션 수·총재고, 필터(비주얼), 요약카드(**옵션 상품이면 라이브가만** + 전체판매 토글 / 비옵션이면 상품가·상시할인가·라이브가 3단), 옵션 구성, `[옵션명 수정하기]`, 옵션 그룹(체크박스+표: 옵션값/가격/재고/판매여부 토글/옵션코드), 닫기·적용.
- **다축(옵션 종류 1~3개) 지원(2026-07-10):** 리스트 순서대로 옵션 종류 개수 배정(`data-optaxes`=`(i%3)+1` → 1,2,3,1,2,3). 각 종류당 값 3개. 표 = **카테시안 조합**(1종류=3행 / 2종류=3×3=9행 / 3종류=27행, 마지막 축이 가장 빠르게 변함). thead=옵션1..N + 가격/재고/판매여부/코드. `state={axes:[{type,values[]}], combos:[{price,stock,sale}]}`. `AXIS_TPL`(옵션1/옵션2/옵션3 템플릿), `buildState(n)`, `comboLabels(ci)`(mixed-radix), `renderGroup()`.
- **옵션 일괄 수정 중첩 팝업 `#optNameModal`**(z-index 120, `#optNameBody`, `.opt-bulk-modal` 900px). 버튼명 `옵션 일괄 수정`(구 "옵션명 수정하기"). **4단계 완료(2026-07-10):** 옵션 종류별 블록 = 종류 input + 값 행들. 각 값 행 = **값/가격/재고 input** + 컨트롤(**삭제**=첫 값 제외 모든 행, **추가**=마지막 행). `state.axes[].values`가 `{name,price,stock}` 객체. `draft`(작업 사본)+`readBody`/`renderBody`로 add/del/edit, `[옵션 적용 하기]`(#optNameSave)→검증(빈 종류/값0 방지, 빈 값 필터)→`state.axes` 커밋+`buildCombos` 재생성+`renderGroup`. **값 추가/삭제 시 조합 수 자동 갱신**(4×3=12 등). `초기화`(#optNameReset)=draft를 마지막 적용상태로 복원.
- **⭐ 추가가격 모델(2026-07-10, 상용 파리티 — 최신):** 모델이 바뀜 → `state={base, priceReg, priceDis, axes:[{type,values:[{name}]}], combos:[{addPrice,stock,sale,code}]}`. **값은 name만**(가격/재고 없음), **조합(combo)별로 addPrice/stock/sale/code**. **조합 표시가격 = 판매가(base=라이브가) + addPrice**. `base`/`priceReg`/`priceDis`는 상품 행의 `.pi-live/regular/discount-input`에서 읽어 요약카드(#optPriceLive/Regular/Discount)에 세팅하되, **옵션 상품(축>0)이면 `renderGroup`이 상품가(#optPriceRegular)·상시할인가(#optPriceDiscount) 부모 span을 `display:none` 처리 → 라이브가만 노출**(리스트 행과 동일 정책, 2026-07-10). **일괄 [가격 수정](`pmApply`)도 옵션 인지**: 옵션 상품 행은 정가/상시할인가 대상에서 제외하고 라이브가만 반영(`.pi-opt-live-input`에 `input` 이벤트 디스패치 → base·숨은 라이브가·옵션가 범위 동기화), 200원 검증도 라이브가에만 적용. 비옵션은 기존 3단 그대로. `AXIS_TPL` 값 = "옵션 N번의 옵션값 M번" 네이밍.
  - **옵션 일괄 수정 모달 2섹션:** ①옵션명/값 편집(`.optb-axis2`: 옵션명 input+옵션 삭제, 옵션값 rows `.optb-vrow2`[값+삭제], `옵션값 추가`=`data-addval`, `+ 옵션 추가`) ②**옵션 상세**(`.optd`): 조합별 표 [축 라벨 | 추가 가격 input | 재고 input | 판매여부 toggle | 자체옵션코드 input] + `일괄 입력`(`#optdBulkBtn`→bar→전체 적용). 구조 변경(값/축/추가/삭제) 시 `syncCombos`(resizeCombos, 위치기준 보존)로 조합 재계산.
  - **[옵션] hover** → `.opt-hover` 팝업: 재고 낮은 순 TOP 4(라벨/재고/가격=base+add) + "전체 옵션 관리하기 (n/total)". (스샷3)
  - **상품 행 라이브가 인풋 + 옵션가 범위(2026-07-10 최신, apply30/31):** 옵션 상품(조합>0)은 행에서 3단(`.pi-prices`) 숨기고 `.pi-opt` 2셀 표시 → **①`라이브가` = 단독 상품처럼 편집 가능한 인풋(`.pi-opt-live-input`, =base=판매가)** + **②`옵션가` = 범위 텍스트(`.pi-opt-range`, `optRangeText(st)`= min~max, 각 조합가=base+addPrice)**. **옵션 상품은 정가/상시할인가 없이 라이브가 하나만.** 라이브가 인풋 편집(document 위임 `input` 리스너)→`state.base` 갱신+숨김 `.pi-live-input` 동기화+옵션가 범위 즉시 재계산. 값이 같으면 `12,000원`, 다르면 `12,000원 ~ 13,000원`. 라벨/셀/범위는 `white-space:nowrap`(좁은 폭 세로깨짐 방지). `updateRowPrice(row)`가 토글, 옵션 추가/삭제 적용 시 **동적 전환**. **초기 배정: index0(미니 대표 선크림)=비옵션(3단, 3338 가격수정 데모가 rowN(0) 3단 인라인 편집을 쓰므로 보호), index1~5=옵션 상품(축 1,2,3,1,2 → 라이브가+범위).** ⚠️ 사용자가 rowN(0)에 옵션을 직접 추가하면 3단→라이브가+범위로 바뀌어 3338 데모가 깨질 수 있음(리로드 시 기본 복귀).
- **옵션 종류 추가/삭제(2026-07-10):** 팝업 하단 **`+ 옵션 추가` 점선 버튼**(축 추가, `data-addaxis`, `MAX_AXES=3` — 3개면 숨김. "최대 3개" 안내는 상단 `.opt-note`로 이동). 추가 시 **`newAxis(ai)`로 옵션값 1개(빈 값)만** 추가(기본 상품 옵션은 `tplAxis`=3값, 별개). 각 종류 아래 **`옵션 삭제`**(축 삭제, `data-axdel`, **항상 노출**). **값 행 추가/삭제(`data-add`/`data-del-ax`)와는 별개** — 전자는 옵션 축, 후자는 옵션 값.
- **옵션 전체 삭제(2026-07-10):** 값 삭제는 **첫 행 포함 모든 행**(canDel 제한 제거). 값 전부 삭제 시 해당 축 자동 제거(`draft.splice`). 축이 0개인 상태로 `[옵션 적용 하기]` → **`#optDelAllModal` 확인 팝업**("옵션을 모두 삭제하시겠습니까?", 닫기/적용, z-index 140). `적용`→`state.axes=[]`·`combos=[]`→renderGroup **빈 상태**(`.opt-empty` "등록된 옵션이 없습니다", 옵션 구성="옵션 없음"). `닫기`→팝업만 닫고 일괄수정 유지. (3중 모달 스택: optionModal→optNameModal→optDelAllModal)
- **상품별 상태 유지(2026-07-10 버그수정):** 옵션 상태를 각 행에 `row.__optState`로 저장(`cloneState`/`persist()`). `openModal`이 저장분 있으면 복원, 없으면 `buildState` 후 저장. 적용/전체삭제/판매여부 토글마다 `persist()`. → 재오픈 시 **템플릿으로 롤백되던 문제 해결**(상품별 독립). (이전 버그: openModal이 매번 buildState로 새로 생성.)
- 필터 체크박스는 축별(`data-all`=축idx, `data-ax`/`data-vi`).
- **PRD/데모/컨플루언스 정식 등록 완료(2026-07-13, GRIPPGM-3336):** FEATURES 5번째 `id:'option'`(jira `GRIPPGM-3336`, demoMode `limited`) 추가 → PRD 모드 스위처·명세(4항목)·Jira pill 렌더. **자동 데모 7스텝**(시작→[옵션]진입&요약→옵션정보(조합)→옵션 일괄 수정 옵션명·값→옵션 상세·일괄 입력→적용·전체삭제→리스트 반영). 데모 acts는 **상태 커밋 없이 모달 open+스팟라이트만**(openModal은 clone 동작 → replay 누적 없음). `resetUI` 모달 목록에 `#optionModal/#optNameModal/#optDelAllModal` 추가(데모 리셋 시 옵션 모달도 닫힘). PRD 타깃: 명세1·2=`.btn-option`, 명세3·4=`.pi-opt`(마커 겹침은 qavis 패턴 수준). **컨플루언스 PRD**: 페이지 `2337406981`(부모 2306867212, 제목 "[PRD] [스튜디오] 상품 옵션 관리 탭 내 가격,재고, 옵션명 일괄 수정 기능 추가"), 레퍼런스 2320039942 양식(TOC·info·문서정보·히스토리·배경목적·KPI·용어·ASIS/TOBE·기능요구5·검증규칙 V-1~V-8·플로우+option_flow.png·화면UX·영향범위) + Jira 매크로. 스크립트 `confluence_create_option.py`/`draw_option.py`/`prd_option.xhtml`. 이제 컨플루언스 PRD 5개(가격2306113587/삭제2319581187/채팅2319515671/Q&A2320039942/옵션2337406981).
- **상품 옵션 관리 표 가격·재고 즉시 편집(2026-07-13, apply35):** `#optGroup` renderGroup의 가격/재고 셀을 정적 `.opt-cell-box`→**편집 인풋**(`.opt-cell-box.is-input` + `.opt-cell-num` + 단위). `group`에 `input` 리스너 추가: **가격 인풋(.opt-price-inp)**=표시값이 조합 총액이라 `addPrice = numOf(값) − state.base`로 역산 저장→persist→`updateRowPrice(currentRow)`로 리스트 옵션가 범위 갱신; **재고 인풋(.opt-stock-inp)**=`stock=fmt(numOf)` 저장→persist. 음수 addPrice 저장/복원 위해 `numOf` **부호 인식**으로 보강(`/^\s*-/` → 음수 반환). 상품 옵션 관리 편집 ↔ 옵션 일괄 수정(추가가격 뷰) ↔ 리스트 옵션가 전부 state 통해 동기화. 컨플루언스 V-9·기능요구 No1 반영(v5). 검증: 9,000→9,500(add 100→600)+재고 10→55 재오픈 유지·범위 9,100~9,500·일괄수정 추가가격 600 동기화·콘솔0.
- **⭐ VOC 반영 대개편(2026-07-14, apply36~38 — 최신, 이전 중첩모달 구조 대체):** GRIPPGM-3336 지라 첨부(운영 att_00 + VOC)를 보니 실제 운영은 **상품 옵션 관리 모달 하나**에서 다 처리(별도 [옵션 일괄 수정] 중첩모달 없음)이고, VOC 3요청=①가격·재고 일괄수정 ②옵션 리스트 +아이콘 옵션추가 ③옵션명 클릭 인라인 수정. 우리 v1은 중첩모달(1depth)이 생겨 어긋남 → **모달 하나로 전면 재구성.** 현재 버전 백업: git tag `option-v1-nested-modal`(@4fc89f0), `_archive/index_option_v1_nested-modal.html`, 스크래치패드 동명. **새 구조(renderGroup 전면 교체):** ①`.opt-edit` 인라인 편집 영역(항상 노출, `nameEditOpen`) — 옵션 종류명(`.opt-etype`)·값(`.opt-eval`) 클릭 인라인 수정 + `.opt-evadd`(+옵션 추가=값1개)·`.opt-evdel`(값삭제)·`.opt-eaxadd`(+옵션 종류 추가 최대3)·`.opt-eaxdel`(종류삭제) ②일괄 수정 바 `.opt-bulk`(선택 N개 + 가격/재고 입력 + `.opt-bulk-apply` 선택 적용) ③조합 표: 행 선택 체크박스 `.opt-rowsel`/전체선택 `.opt-selall`(선택은 `c._sel` 트랜지언트, **syncSel로 re-render 없이 부분갱신** — 일괄 입력값·포커스 보존), 가격/재고/자체옵션코드(`.opt-code-inp`) 전부 인라인, 판매여부 토글. 종류/값명 편집은 `syncComboLabels`로 헤더·라벨만 부분 갱신(포커스 보존). **[옵션 일괄 수정] 버튼 → [옵션명 수정하기]로 라벨 변경**(#optEditNameBtn), 클릭 시 `nameEditOpen` 토글(.opt-edit 접기/펼치기, active 클래스). 구 중첩모달(#optNameModal/#optDelAllModal)·renderBody/openName 등은 **미사용 dead code로 잔존**(제거 안 함). 데모 6스텝 새 플로우로 교체(인라인 편집→가격/재고/코드→체크박스 일괄→선택 적용, 커밋 없이 스팟라이트). 인앱 PRD(개요·플로우·명세1~3) 새 구조 반영. **검증:** 2개 선택→일괄 10,000/100 적용(입력값 유지)·옵션값→레드/종류→색상 동기화·+값추가 4조합·+종류추가 2축·코드 A-1·재오픈 저장유지·데모 전스텝·콘솔0. **UI 정리(2026-07-14, apply39~41, "정신없다" 피드백):** ①일괄 수정 = 상시 노출 바 제거 → **`.opt-info-head`에 [일괄 수정] 버튼(#optBulkBtn, 선택 시 활성·"일괄 수정 (N)")** → 클릭 시 **작은 팝업 #optBulkModal**(가격/재고 입력 + 적용, 선택 조합에만, 입력 항목만 반영). 체크박스는 유지(3338 가격수정·3357 선택삭제 패턴과 동일). ②**옵션 종류·값 편집 영역(.opt-edit) 기본 접힘**(nameEditOpen 기본 false, openModal마다 false 리셋) → **[옵션명 수정하기]로 펼침/접기**(운영 att_00처럼 기본은 요약+버튼+표만). updateBulkBtn()이 renderGroup·syncSel 양쪽에서 버튼 상태 갱신(적용 후 선택 해제→비활성). 데모 6스텝 재정렬(진입→가격/재고/코드 인라인→[옵션명 수정하기] 펼침→체크박스 선택→[일괄 수정] 팝업). 인앱 PRD도 반영 필요분 반영. 검증: 기본 접힘·버튼 비활성, 2선택→"일괄 수정(2)"→팝업 12000/88 적용(3,100/88)·적용 후 비활성, 토글 열기/닫기, 데모 팝업까지, 콘솔0. ⚠️ **컨플루언스 PRD 2337406981은 아직 구버전 서술 — 사용자 확인 후 재동기화 예정.**

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
**구현됨(최종 정책 — 본인만 보기 + ⋮→수정하기 사후 수정):** 답변 인라인 박스(`#chatReply`)에 **"본인만 보기" 토글(`#crLiveBox`, 기본 OFF)**. `registerQA(m,a,priv)`에 `priv` 플래그. **OFF(기본)=전체 공개**(모든 유저 Q&A + 방송 프리뷰 띠지 노출), **ON=본인만**(문의한 본인에게만, 다른 유저·방송 미노출). `updateOvQA()`는 **priv=false(전체 공개)인 최신 항목만** 띠지 표시.
- **시청자(모바일 앱) 노출 의미 — 핵심(2026-07-03 명확화):** 전체 공개=모든 시청자 모바일 앱 Q&A+방송 화면 노출. **본인만=문의한 A의 모바일 앱에는 전체 공개와 똑같이 Q&A가 정상 노출**(A는 답변받은 것으로 보임), **다른 시청자 앱·방송엔 미노출**. 즉 A에게 숨기는 게 아니라 A에게만 보여주는 것(1:1 DM 아님). 인앱 개요·명세2·데모 콜아웃 + 컨플루언스 용어·명세2에 반영. (프로토타입은 스튜디오 화면이라 모바일 뷰는 없음 → 문서로만 서술.)
- **사후 수정(2026-07-03 재추가):** Q&A 탭 항목 **⋮ → 수정하기**(`data-a="edit"`, 답변 있으면 노출) → `answering=true` → **인라인 편집 폼**(답변 input + `.qa-edit-live` [본인만 보기] 토글, 첫 답변과 동일 UX) → **저장** 시 `qaList[i].a`+`priv` 갱신 → 배지·띠지 즉시 반영. 배지 `.qa-vis`는 표시용(직접 클릭 토글 아님, 수정은 ⋮→수정하기로). (예전 "배지 직접 클릭 사후 전환"은 제거했고, 대신 이 인라인 편집 방식으로 재구현 — 사용자가 "너무 어렵다"던 것 해결.)
- PRD 기능4 "qavis"(GRIPPGM-3402) — 인앱 기능명 = **"[스튜디오] 채팅내용 답변 시 Q&A 미노출 기능 추가"**(지라 티켓 제목과 일치, 2026-07-03 변경). 컨플루언스 info 스위처 안내 문구도 동일하게 맞춤. **명세 3개**(1:답변 시 노출범위 토글, 2:본인만=다른유저 미노출, 3:**Q&A 탭에서 노출 범위 수정 ⋮→수정하기**). 데모(…→전체공개→본인만→**사후수정(⋮→수정하기)→저장**). 검증 완료(⋮→수정하기→토글→저장→배지 🔒 본인만/띠지 갱신, 콘솔 0).
- (이력: "라이브 노출 on/off"→"전체 공개 기본/본인만 토글"→"사후 전환 제거"→"⋮→수정하기 인라인 편집으로 사후 수정 재추가".)
- **컨플루언스 PRD: page 2320039942**(부모 2306867212, 제목 "채팅내용 답변 시 Q&A 미노출 기능" — 사용자가 편집함). 사후 수정 반영해 **v8 갱신**(명세3·검증·UX·히스토리 ver0.2 + 플로우차트, 이미지 렌더 확인). ⚠️ 이 페이지는 사용자가 Confluence 에디터로 직접 편집(local-id 붙은 verbose storage, 담당자 user-mention/APP 추가) → **전체 덮어쓰기 금지, 타깃 치환으로만 갱신**(라이브 body fetch→치환→resave). (삭제 PRD 2319581187도 판매모드 정책 v9 갱신 완료.)
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
