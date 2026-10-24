# 김정호의 신용평가 검색기 — 프로젝트 안내 (Claude Code용)

## 무엇을 만드는가
일반인(한국어 사용자)이 나라·기업 이름을 검색하면 S&P·무디스·피치 국제 신용등급과
국내 신평 3사(NICE신용평가·한국신용평가·한국기업평가) 등급을 쉬운 말로 보여주는 공개 웹사이트.
문의: MOJO789@GMAIL.COM (페이지 하단 표기 유지)

## 현재 상태 (2026-10-24 기준, Claude Cowork에서 만든 버전)
- `site/index.html`: 단일 HTML. 시작할 때 `data/ratings.json`을 fetch해서 씀(배포 시 `data/`가 페이지 옆, 저장소에선 `../data/`).
  fetch를 쓰므로 파일을 더블클릭(file://)하면 안 열림 → 저장소 루트에서 `npx http-server -c-1 .` 후 `/site/` 접속.
- `data/ratings.json`: 원본 데이터 144곳. **이 파일이 단일 원본(source of truth)**. 페이지 내장 데이터는 2026-10-24에 제거함.
- 화면 기능: 쉬운 말 판정(매우 안전~부도 7단계), 5년 부도율 체감 문구, 평가사 3곳 한 줄 눈금,
  국내 vs 국제 비교, 비교함(최대 4곳), 초성 검색·자동완성, 분류 필터, 즐겨찾기(localStorage),
  결과 복사, `#id` 딥링크, 최근 12개월 등급 변동, 다크 모드.
- 배포: https://sogogiboy.github.io/credit-rating-search/ (`.github/workflows/pages.yml`이 `site/` + `data/`를 올림).
- 'Claude에게 찾아달라고 하기'는 `worker/`(Cloudflare Worker, `/api/lookup`·`/api/quota`)를 호출.
  `site/index.html`의 `var API=""`에 Worker 주소를 넣어야 버튼이 보임. 무료 한도 IP당 하루 5회·전체 100회(`wrangler.toml` vars),
  처음 쓸 때 하루 한 번 한도 안내 팝업. 같은 이름 결과는 KV에 24시간 캐시(횟수 차감 없음).

## 해야 할 일 (순서대로)
1. **구조 정리**: `site/index.html`의 내장 데이터를 제거하고 `data/ratings.json`을 fetch해서 쓰도록 리팩터링.
   화면·문구·디자인은 그대로 유지. 빌드 도구 없이 정적 파일(HTML/CSS/JS)로 유지해도 됨.
2. **AI 검색 서버**: Cloudflare Workers(무료 플랜)로 `/api/lookup` 엔드포인트 작성.
   - Anthropic API(Claude) + **웹 검색 도구**를 사용해 최신 등급을 찾고, 출처 URL과 기준 날짜를 함께 반환.
   - 응답 JSON: `{name, en, type, sp, moodys, fitch, outlook:[..], domestic:[nice,kis,kr], asOf, sources:[{title,url,date}], confidence}`
   - 확인 못 한 값은 반드시 null. 절대 추정·창작 금지(시스템 프롬프트에 명시).
   - API 키는 Workers secret(`ANTHROPIC_API_KEY`)으로만 보관. 코드·저장소에 절대 넣지 말 것.
   - 남용 방지: IP당 하루 N회 제한(KV 또는 Durable Objects), 전체 일일 상한, 같은 이름 결과 24시간 캐시.
   - 프론트의 'Claude에게 찾기' 버튼이 이 엔드포인트를 호출하도록 교체.
3. **배포**: GitHub 저장소 → GitHub Pages(또는 Cloudflare Pages)로 사이트 배포, Worker는 `wrangler deploy`.
4. **등급 자동 갱신**: GitHub Actions 주 1회 스케줄 작업으로 `data/ratings.json`의 각 항목을
   Worker(또는 스크립트)로 재확인 → 바뀐 값만 PR로 올리기. 사람이 확인 후 병합(자동 병합 금지).
5. **데이터 보강**: `verified:false`이거나 null이 많은 항목, 그리고 아래 '미확인 목록'부터 채우기.

## 데이터 규칙 (중요)
- 등급 표기: S&P·피치 `AAA, AA+, … , D`, 무디스 `Aaa, Aa1, … , C`. 국내 3사는 S&P식 표기.
- 전망: "안정적" / "긍정적" / "부정적" / null.
- 출처 없는 값은 넣지 않는다. 출처는 `source`에 매체·날짜를 적는다. 2024년 이전 자료만 있으면 null.
- 국내 등급은 국제 등급과 같은 자가 아님 → 화면에 이 안내 문구 유지.
- 부도율 문구는 S&P 글로벌 기업 장기 평균(1981년 이후 5년 누적)의 대략값이며 나라에는 그대로 적용 안 됨.
- 투자 권유 아님 안내 문구 유지.

## 조사 팁 (이전 작업에서 확인된 것)
- 접근이 자주 막히는 곳: kisrating.com, nicerating.com, korearatings.com, kind.krx.co.kr, moodys.com, 다수의 *.co.kr 회사 IR.
- 잘 되는 곳: cbonds.com 뉴스("Moody's affirms X at Aa2"), mnimarkets.com, etoday.co.kr, fnnews.com, dealsite.co.kr,
  SEC EDGAR FWP/424B2(미국 회사채 발행 조건에 등급 표기), 회사 영문 IR 신용등급 페이지, Trading Economics(국가).
- 공공데이터포털에 "한국서부발전(주)_국내외 신용등급"(2025.08) 파일 데이터 있음.

## 미확인 목록 (채우면 좋음)
카카오, 삼성SDI, 삼성생명, 현대모비스, 셀트리온, 삼성바이오로직스, 하나금융지주, 우리금융지주, GS칼텍스,
인천국제공항공사, 한국지역난방공사, 한국공항공사, 캠코, BYD, AIA, 핑안, 공상은행, 레노버, HKEX,
버크셔해서웨이, 코카콜라, 비자, 마스터카드, 뱅크오브아메리카, 골드만삭스, 포드, GM, ASML, LVMH, 네슬레,
발전 5사 피치 등급, 국내 3사 평가사별 등급(공기업 다수).
