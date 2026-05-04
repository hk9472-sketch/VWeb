# VWeb 스택 결정 검토 — 대화 정리

**일자**: 2026-05-04
**참고 프로젝트**:
- 동천교회 홈페이지 — `D:\Works\Christ\pkistdc_new\dongcheon-church` (Next.js 16 + TS + Prisma + MySQL)
- VMES C# 원본 — `D:\Works\Projects\000_VMES\VMES_Upgrade` (WinForms + DevExpress)
- VMES_KASWIN (현역 진행본) — `D:\Works\Projects\000_VMES\VMES_KASWIN` (Express + EJS + DevExtreme jQuery + MSSQL)
- VWeb 가이드 — `D:\Works\Projects\000_VMES\VWeb\DEVELOPMENT_GUIDE.md` (Express + EJS + Prisma + DevExtreme 노선)

---

## 1. 세 프로젝트 비교 요약

| 항목 | 동천교회 (모던 풀스택) | VMES_KASWIN (현역) | VMES C# (원본) |
|---|---|---|---|
| 프레임워크 | **Next.js 16 (App Router)** | Express 4 + EJS | WinForms |
| 언어 | **TypeScript 5.9** | JavaScript (CJS) | C# |
| ORM/DB | Prisma 6 + MySQL | mssql 9 raw, **다중 DB 풀** | EnterpriseDB + SP |
| 인증 | NextAuth | express-session + 쿠키 | UserInfo singleton |
| UI | React 19 + Tailwind 4 + Tiptap | EJS + **DevExtreme 25** | DevExpress 21.2 |
| 데이터 접근 | Prisma 모델 + `$executeRaw` | **`callProcedure` (SP 직접)** | SP 100% |

VWeb 가이드 노선은 "Express + EJS + Prisma + DevExtreme + JS"로 KASWIN 계승형이지만, 동천교회의 모던 스택을 흡수할지가 핵심 갈림길.

---

## 2. 두 갈림길 — 트랙 A vs 트랙 B

### 트랙 A — KASWIN 계승
- 구성: Express + EJS + DevExtreme jQuery + JavaScript
- 장점: KASWIN 화면(`baseDesign01.ejs`) · SP · `db.js` 그대로 복붙 가능. 학습비용 0
- 단점: TS 없음. SPA 전환 시 재작업. 동천교회 자산 재사용 어려움

### 트랙 B — 모던 풀스택
- 구성: Next.js 16 + TypeScript + Prisma + DevExtreme React
- 장점: 동천교회 노하우 직접 이식. App Router로 API+UI 한 트리. 장기 유지보수 우위
- 단점: KASWIN의 EJS 화면을 React로 재작성. DevExtreme React 라이선스/노하우 추가

### 트랙 C — 분리형 (Express API + Next.js 프론트)
- 거의 비추 (배포 2개, CORS/세션 복잡)

---

## 3. KASWIN 화면이 B로 이어질 수 있는가

KASWIN의 화면은 4개 요소로 구성:

| 요소 | 무엇 | B(React)로 이어지나? |
|---|---|---|
| ① **HTML 골격** (EJS, `condition-area`/`data-form` div) | 빈 컨테이너만 두는 템플릿 | ✅ JSX로 **기계적 변환** 가능 |
| ② **CSS 디자인 시스템** (`themed-grid-col`, `textBox`, `data-form-area`) | 자체 디자인 토큰 | ✅ **100% 재사용** — `globals.css`로 그대로 |
| ③ **DevExtreme 컴포넌트** (`dxDataGrid`, `dxButton`, `dxDateBox`) | UI 위젯 | ✅ **DevExtreme React** 동일 라이선스/동일 옵션 |
| ④ **`class FrmXxx extends BaseDesign01`** (jQuery 호출 패턴) | C# 폼 클래스 답습 | ❌ **재작성 필요** — React 패러다임으로 |

→ **CSS · 레이아웃 · DevExtreme 옵션 객체는 그대로**, **jQuery 호출 코드는 재작성** 필요.

### 자산별 이관률 정량화

| 자산 | 이관률 |
|---|---|
| SP / SQL | 100% |
| `dbConfig.json` (14개 고객 DB) | 100% |
| `db.js`, `serverProcess.js` | 95% (TS 포팅) |
| Express 라우트 → Route Handler | 80% |
| **CSS 디자인 토큰** | **100%** |
| EJS 골격 | 70% |
| **DevExtreme 옵션 객체** | **85~90%** |
| `BaseDesign01.js` | 0% (재설계) |
| 화면별 비즈니스 로직 | 40~70% |
| 직접 jQuery 호출 | 0% |

**평균 이관률: 60~75%** (가드레일 없는 일반 시나리오)

---

## 4. 트랙 A → B 100% 이관 가능성

**결론: 100%는 불가능. 잘해도 90~95%, 평범하게 짜면 60~75%.**

### 100%를 막는 5가지 원인
1. A 운영 중에 화면이 늘어남 (1년 운영 → 견적 2배)
2. "동작하는 화면 다시 만들기"의 ROI가 낮아 정치적으로 통과 어려움 → **legacy island 영구화**
3. A로 짤 때 B 고려 안 하면 부채 누적 (jQuery hack은 1:1 매핑 불가)
4. 두 패러다임 동시 운영의 인지 부하 → "그냥 A로 추가하자"가 됨
5. 픽셀/동작 회귀 검증 비용

### 95%+ 이관률 보장 가드레일

A 시작 시 처음부터 지켜야 할 5가지:

1. **비즈니스 로직과 UI 코드 분리** — SP 호출 등은 `services/` 별도 모듈
2. **DevExtreme 옵션을 별도 모듈로 추출** — `gridOptions/workOrderGrid.js` 식으로
3. **SP 호출 레이어 단일화** — 화면에서 `fetch('/callProc')` 직접 호출 금지
4. **직접 DOM 조작 최소화** — 명령형(`$('#x').show()`)보다 상태→렌더링 패턴
5. **TypeScript는 처음부터** — 최소 SP 응답/도메인 타입은 `.d.ts`로

| 시나리오 | 이관률 |
|---|---|
| A 자유롭게 + 손 마이그 | 50~65% |
| A 가드레일 + 손 마이그 | 75~85% |
| **A 가드레일 + Claude 마이그** | **92~98%** |
| A 자유롭게 + Claude 마이그 | 80~90% |

---

## 5. Claude 자동 마이그의 영향

이전 견적의 **"마이그 5개월"은 사람이 손으로 한다는 가정** 위에 있었음. Claude 협업으로 8~10배 단축 가능.

### Claude가 잘하는 것

| 작업 | 정확도 | 화면당 시간 |
|---|---|---|
| EJS 골격 → JSX 변환 | 95% | 5분 |
| `class FrmXxx extends BaseDesign01` → React 함수형 | 85% | 10분 |
| `$('#x').dxDataGrid({...})` → `<DataGrid {...} />` | 90% | 5분 |
| jQuery 이벤트 → React 핸들러 | 80% | 10분 |
| `this.dataStore.*` → `useState` | 75% | 10분 |
| SP 호출 라인 그대로 유지 | 100% | 0분 |
| TypeScript 타입 추가 | 80% | 10분 |

→ **화면당 사람 4시간 → Claude 30분~1시간** (8~10배 단축)

### 사람이 반드시 해야 하는 것
- 베이스 컴포넌트 첫 설계 (`<BaseFormPage>`)
- 실행 검증 (브라우저 클릭/탭/저장)
- 미묘한 jQuery 부작용 발견 (DOM 타이밍, 이벤트 버블링)
- 비즈니스 의도 모호한 코드 결정
- 회귀 테스트

### 100개 화면 마이그 비용 재추정
- 사람 손: 5개월 1FTE
- **Claude + 사람 검증: 3~5주 (변환 1주 + 검증 2~4주)**

→ 정치적/기술적으로 "마이그 영구 보류" 함정에서 벗어남.

---

## 6. Walking Skeleton 필요성

화면 1개만 변환하는 PoC는 띄워볼 수 없음 — 운영환경 인프라가 필요:

### KASWIN 구동환경 → B 대응

| KASWIN 구성요소 | B(Next.js) 대응 작업 |
|---|---|
| `template/header.ejs` (jQuery, Bootstrap, DevExtreme CDN, fonts) | `app/layout.tsx` + `globals.css` + DevExtreme React + theme |
| `views/QMeSystem/index.ejs` (로그인 + DB선택 + 사업장/공장) | `app/(auth)/login/page.tsx` + NextAuth Credentials |
| `views/QMeSystem/main_tile.ejs` (사이드바 + iframe) | `app/(app)/layout.tsx` + sidebar + **iframe? 또는 라우팅?** |
| `simple-treeview` (메뉴) | DevExtreme `<TreeView>` + `usp_User_menu` SP 호출 |
| `tabManager.js` | React Context 또는 Zustand 탭 상태 |
| `baseFactor.js`, `PsLibs.js`, `PsSQL.js` | TS 모듈 (`src/lib/PsLibs.ts` 등) |
| `db.js`, `serverProcess.js` | `src/lib/db.ts`, `src/lib/sp.ts` |
| `dbConfigPopup.js` | `<DbSelectModal>` 컴포넌트 |
| `routes/index.js` 등 | `app/api/.../route.ts` |
| `baseDesign01.ejs/.js` | `<BaseFormPage>` + 베이스 훅 |

### 핵심 결정 — iframe vs SPA 라우팅

KASWIN은 iframe 기반 탭 시스템(`<iframe id="mainFrame">`).

| 옵션 | 장점 | 단점 |
|---|---|---|
| **iframe 유지** | 미변환 KASWIN 화면을 그대로 띄울 수 있음 → 점진 마이그가 자연스러움 | SPA 부드러움 사라짐, URL 직링크/뒤로가기 어려움 |
| **Next.js 라우팅 + 탭** | 진짜 SPA, URL 직링크 가능 | 마이그 완료 전까지 운영 불가 |
| **하이브리드** | Next.js 라우팅 메인, 미변환 화면은 iframe wrapper | 두 패턴 공존 |

→ **권장: 하이브리드.** 운영 가능 상태를 유지하며 점진 전환.

### Walking Skeleton 작업 견적 (B 인프라 1세트)

| 작업 | 사람 단독 | Claude 협업 |
|---|---|---|
| Next.js 셋업 + TS + DevExtreme React + 디자인 토큰 CSS 이식 | 1일 | 2시간 |
| `db.ts` + `sp.ts` (KASWIN 포팅) + `dbConfig.json` 이전 | 0.5일 | 1시간 |
| NextAuth + 로그인 + DB/사업장/공장 선택 + 세션 | 1.5일 | 3~4시간 |
| 메인 레이아웃(사이드바 + 트리뷰 + 탭) + iframe wrapper | 2일 | 4~6시간 |
| `<BaseFormPage>` + 변환 룰 매핑표 | 1일 | 2~3시간 |
| 샘플 화면 1개(`frmMF12020M`) 변환 | 0.5일 | 1~2시간 |
| 통합 테스트/디버깅 | 1일 | 2~3시간 |
| **합계** | **7.5일** | **약 2일 (15~20시간)** |

---

## 7. 인력 역량 진단 (트랙 결정의 전제)

기존 인력이 B를 다룰 수 있는지 확인 필요.

### B 트랙 진입 시 6가지 격차

| # | 영역 | KASWIN | B에서 필요 | 격차 |
|---|---|---|---|---|
| 1 | 언어 | JS (CJS) | TypeScript | ★★ |
| 2 | UI 패러다임 | jQuery + DOM 직조작 | React 함수형 + Hooks | ★★★ |
| 3 | 상태 관리 | 클래스 인스턴스 변수 | useState/useReducer/Context | ★★★ |
| 4 | 빌드/툴체인 | nodemon | Next.js, ESM, Turbopack | ★ |
| 5 | 라우팅 | Express router | App Router | ★★ |
| 6 | DevExtreme | jQuery widget | React component | ★ |

→ **2·3이 진짜 병목** (사고방식 변환, 1개월+ 학습)

### 진단 방법 (반나절~1일)

#### 옵션 A — 샌드박스 미니 과제 (반나절)
> "MSSQL `usp_User_menu`를 호출해서 결과를 그리드로 뿌리는 화면을 Next.js + TS + DevExtreme React로 만드시오."
> 시간: 4시간

채점 (Pass/Fail):
- 프로젝트 셋업 (`create-next-app --typescript`)
- Route Handler에서 SP JSON 반환
- Server/Client 컴포넌트 구분 (`"use client"`)
- TS 인터페이스 작성
- DataGrid columns/dataSource props
- 자력 디버깅

**4/6 이상 통과** = B 가능. **2/6 이하** = A 권장.

#### 옵션 B — 인터뷰 체크리스트 (30분/사람)
- TypeScript 인터페이스/제네릭 작성?
- React useState/useEffect 차이 설명?
- Promise/async-await?
- npm 의존성 관리?
- git PR 리뷰?
- 모던 프레임워크(React/Vue/Angular) 1년+?

**4개 이상** = B 적응 가능. **2개 이하** = A 권장.

#### 옵션 C — 코드 리딩 테스트 (1시간)
동천교회 임의 페이지(`.tsx` + `route.ts`)를 보여주고 설명시키기.
- 70% 이해 = B 가능
- 절반 이상 막힘 = A 권장

### 결과별 트랙

| 인력 상태 | 권장 트랙 |
|---|---|
| 전원 B 가능 | B 직행 |
| 절반 B, 절반 jQuery만 | A → B 점진 |
| 다수 jQuery만 | A 시작 + 1~2명 B 파일럿 |
| 모두 jQuery만, 학습 의지 낮음 | A 유지 |

---

## 8. 최종 검토 — 세 가지 시작 옵션

### 옵션 X — 트랙 A 즉시 시작
- KASWIN을 그대로 복사, 폴더명만 VWeb로
- 1주 안에 운영 가능
- 가드레일은 처음부터 적용
- B 가능성은 미래에 재평가

### 옵션 Y — B Walking Skeleton 먼저 (2~3주)
- 견적대로 인프라 구축 + 화면 1개 변환
- 패턴 검증 후 트랙 결정
- 잘 됐다 → B 직행 또는 A→B 점진
- 안 됐다 → A로 회귀 (학습 자산은 남음)

### 옵션 Z — 하이브리드 즉시 시작 ⭐
- Next.js 메인 레이아웃 + 인증 + 메뉴만 B로
- iframe 안에 KASWIN 화면을 **변환 없이** 띄움
- 신규 화면만 React로
- 기존 화면은 천천히 변환 (또는 안 함)

#### 옵션 Z의 강점
1. KASWIN 모든 화면을 변환 없이 B 환경에서 띄울 수 있음 (iframe wrapper)
2. 신규 화면은 React/TS로 짜며 인력 학습 동시 진행
3. 마이그 압력 0 → 정치적/기술적 위험 0
4. 진짜 마이그 필요 시 Claude로 뒷정리

→ **트랙 A의 단기 이익 + 트랙 B의 장기 이익을 모두**.

---

## 9. 다음 액션 후보

1. **인력 역량 진단 먼저 진행** — 옵션 A/B/C 중 선택
2. **옵션 Z의 Walking Skeleton 설계도 작성** (코드 X, 30분)
3. **옵션 Z Walking Skeleton 실제 코드** (Phase 0~1까지 한 세션)
4. **코딩 표준 문서 먼저 작성** — 가드레일 1~5 + iframe wrapper 패턴
5. **결정 보류** — 검토 후 다음 세션에서 이어가기

---

## 10. 핵심 미결 항목

- [ ] 인력 역량 진단 결과 (트랙 결정의 전제)
- [ ] iframe vs SPA 라우팅 (옵션 Z 채택 시 자동으로 하이브리드)
- [ ] V1에서 Oracle/Tibero 필요 여부 (없으면 MSSQL-first)
- [ ] DevExtreme React 라이선스 확인 (KASWIN과 동일 라이선스에 포함되는지)
- [ ] 옵션 X / Y / Z 중 시작 노선 결정

---

## 부록 — 참조 코드 위치

| 파일 | 용도 |
|---|---|
| `D:\Works\Projects\000_VMES\VMES_KASWIN\src\db.js` | 다중 DB 풀 + 자동 재연결 (TS 포팅 대상) |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\src\serverProcess.js` | `callProcedure` / `callTransactionProcedure` (TS 포팅) |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\config\dbConfig.json` | DB 접속 정보 |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\views\PsControls\baseDesign01.ejs` | 화면 베이스 템플릿 |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\modules\PsControls\public\js\Basedesign01.js` | 화면 베이스 클래스 |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\views\QMeSystem\index.ejs` | 로그인 화면 |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\views\QMeSystem\main_tile.ejs` | 메인 레이아웃 (iframe 탭) |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\template\header.ejs` | 공통 헤더 (CDN/CSS) |
| `D:\Works\Projects\000_VMES\VMES_KASWIN\views\PsManufacturing\frmMF12020M.ejs` | 화면 샘플 (변환 PoC 대상) |
| `D:\Works\Christ\pkistdc_new\dongcheon-church\src\app\` | Next.js App Router 패턴 참고 |
| `D:\Works\Christ\pkistdc_new\dongcheon-church\prisma\schema.prisma` | Prisma 스키마 패턴 참고 |
| `D:\Works\Projects\000_VMES\VWeb\DEVELOPMENT_GUIDE.md` | 기존 가이드 (Express+EJS+Prisma 노선) |
