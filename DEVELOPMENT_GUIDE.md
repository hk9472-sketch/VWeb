# VWeb - MES Web Application 개발 가이드

## 1. 프로젝트 개요

### 목적
기존 C# 기반 VMES 데스크톱 애플리케이션을 웹 기반으로 전환하는 프로젝트.
VMES_KASWIN(Node.js 진행중 버전)의 경험을 반영하여 현대적 아키텍처로 재설계한다.

### 핵심 목표
- **웹 기반**: 브라우저에서 동작하는 MES 시스템
- **다중 DB 지원**: MSSQL, Oracle, Tibero, PostgreSQL 등 다양한 DB 연결
- **모듈화**: 도메인별 독립적 모듈 구조
- **기존 호환**: 기존 저장 프로시저(Stored Procedure) 기반 데이터 접근 유지

---

## 2. 기술 스택

| 계층 | 기술 | 비고 |
|------|------|------|
| **Runtime** | Node.js (LTS) | v20+ 권장 |
| **웹 프레임워크** | Express.js | REST API 서버 |
| **ORM** | Prisma | 다중 DB 지원, 타입 안전성 |
| **DB** | MSSQL / Oracle / PostgreSQL / Tibero | Prisma provider 교체로 대응 |
| **인증** | express-session + JWT | 세션 기반 + 토큰 기반 하이브리드 |
| **API 문서** | Swagger (OpenAPI) | 자동 API 문서 생성 |
| **프론트엔드** | EJS + DevExtreme | 기존 KASWIN 방식 유지 가능 |
| **테스트** | Jest | 단위/통합 테스트 |

---

## 3. 프로젝트 구조

```
VWeb/
├── prisma/
│   ├── schema.prisma          # Prisma 스키마 정의
│   └── migrations/            # DB 마이그레이션 파일
├── src/
│   ├── server.js              # Express 서버 진입점
│   ├── app.js                 # Express 앱 설정 (미들웨어 등)
│   ├── config/
│   │   ├── database.js        # DB 연결 설정
│   │   ├── dbConfig.json      # DB 접속 정보 (gitignore 대상)
│   │   └── app.js             # 앱 환경설정
│   ├── middleware/
│   │   ├── auth.js            # 인증 미들웨어
│   │   ├── errorHandler.js    # 에러 핸들링
│   │   └── validation.js      # 입력 검증
│   ├── routes/
│   │   ├── index.js           # 라우트 통합
│   │   ├── auth.js            # 인증 라우트
│   │   ├── procedure.js       # 저장 프로시저 실행 라우트
│   │   ├── dbConfig.js        # DB 설정 관리 라우트
│   │   └── modules/           # 모듈별 라우트
│   │       ├── manufacturing.js
│   │       ├── quality.js
│   │       ├── inventory.js
│   │       ├── sales.js
│   │       ├── purchase.js
│   │       ├── maintenance.js
│   │       ├── kpi.js
│   │       └── system.js
│   ├── services/
│   │   ├── procedureService.js  # SP 실행 서비스
│   │   ├── transactionService.js # 트랜잭션 서비스
│   │   └── modules/             # 모듈별 비즈니스 로직
│   ├── models/                  # Prisma 모델 활용 레이어
│   ├── utils/
│   │   ├── dbPool.js           # 다중 DB 커넥션 풀 관리
│   │   ├── logger.js           # 로깅 유틸
│   │   └── helpers.js          # 공통 유틸
│   └── views/                   # EJS 템플릿
│       ├── layouts/
│       │   └── main.ejs
│       ├── pages/
│       │   ├── login.ejs
│       │   └── dashboard.ejs
│       └── modules/
│           ├── manufacturing/
│           ├── quality/
│           ├── inventory/
│           └── system/
├── public/
│   ├── js/
│   ├── css/
│   └── images/
├── docs/                        # 프로젝트 문서
├── tests/                       # 테스트 파일
├── .env                         # 환경변수 (gitignore 대상)
├── .env.example                 # 환경변수 예시
├── .gitignore
├── package.json
└── DEVELOPMENT_GUIDE.md         # 이 문서
```

---

## 4. 참고 프로젝트 분석 요약

### 4.1 VMES (C# 데스크톱) - 원본 시스템

**아키텍처**: Windows Forms + WCF Web Service + Enterprise Library
**DB 지원**: MSSQL, Oracle, Tibero (EnterpriseDB 추상화 레이어)

**핵심 모듈 (그대로 웹으로 이식해야 할 대상)**:

| 코드 접두사 | 모듈명 | 설명 |
|------------|--------|------|
| BA | System | 사용자/권한/코드/조직 관리 |
| MF | Manufacturing | 생산계획, 작업지시, 생산실적, 외주, 비가동 |
| QM/CT/QA | Quality | 품질검사, 부적합, 공정관리계획 |
| WH/LT | Inventory | 재고현황, 입출고, LOT 추적 |
| SA | Sales | 수주관리 |
| PU | Purchase | 구매관리 |
| MT | Maintenance | 설비보전 |
| KPI | KPI | 핵심성과지표 |
| FMB | Monitoring | 공장모니터링보드 |

**데이터 접근 패턴**:
- 모든 DB 접근은 저장 프로시저(SP) 기반
- SP 네이밍: `[PREFIX]_[TABLE]_[OP][NUM]` (예: `WIP_LOTSTS_SEL01`)
- Oracle은 패키지 단위: `PKG_COMMON`, `PKG_BAS`, `PKG_WIP` 등

**주요 엔티티 코드 체계**:
- `BIZ_CD` (사업장), `PLANT_CD` (공장), `WC_CD` (작업장)
- `LINE_CD` (라인), `DEPT_CD` (부서), `ITEM_CD` (품목)
- `LOT_NO` (LOT번호), `PR_CD` (공정), `MACH_CD` (설비)

### 4.2 VMES_KASWIN (Node.js 진행중) - 직접 참고 대상

**아키텍처**: Express + EJS + MSSQL + DevExtreme

**잘된 점 (계승해야 할 것)**:
- 모듈별 폴더 구조 (PsManufacturing, PsQuality 등)
- 저장 프로시저 실행 범용 API (`/callProc`, `/callProcTrans`)
- 다중 DB 커넥션 풀 관리 (`db.js`)
- Basedesign01.ejs 공통 템플릿 패턴
- 세션 기반 DB 전환 (로그인 시 DB 선택)

**개선해야 할 것 (VWeb에서 해결)**:
- ORM 없음 → **Prisma 도입**
- MSSQL만 지원 → **다중 DB provider 지원**
- 테스트 없음 → **Jest 기반 테스트**
- API 문서 없음 → **Swagger/OpenAPI**
- RBAC 미구현 → **역할 기반 접근 제어**
- CSRF 보호 없음 → **보안 미들웨어 추가**
- 에러 핸들링 미흡 → **통합 에러 핸들러**

---

## 5. 다중 DB 지원 전략

### 5.1 Prisma 기반 접근

Prisma는 `schema.prisma`의 `provider`를 통해 DB를 전환한다.

```prisma
// prisma/schema.prisma
datasource db {
  provider = env("DB_PROVIDER")  // "sqlserver", "postgresql", "oracle" 등
  url      = env("DATABASE_URL")
}
```

**지원 DB별 provider**:
| DB | Prisma Provider | 비고 |
|----|----------------|------|
| MSSQL | `sqlserver` | 기본 지원 |
| PostgreSQL | `postgresql` | 기본 지원 |
| MySQL | `mysql` | 기본 지원 |
| Oracle | 미지원 | 별도 드라이버 필요 |
| Tibero | 미지원 | ODBC 또는 별도 드라이버 |

### 5.2 하이브리드 접근 (권장)

Prisma가 지원하지 않는 DB(Oracle, Tibero)를 위해 이중 구조를 채택한다.

```
┌──────────────────────────────────────────┐
│              Service Layer               │
├──────────────────────────────────────────┤
│  Prisma Client    │  Raw DB Driver       │
│  (MSSQL, PG, MY)  │  (Oracle, Tibero)    │
├──────────────────────────────────────────┤
│              DB Pool Manager             │
│  (다중 DB 커넥션 풀 통합 관리)             │
└──────────────────────────────────────────┘
```

**DB Pool Manager 설계** (KASWIN의 `db.js` 확장):

```javascript
// src/utils/dbPool.js
const pools = {};

// Prisma 기반 DB
async function getPrismaClient(dbName) { /* ... */ }

// Native 드라이버 기반 DB (Oracle, Tibero)
async function getNativePool(dbName) { /* ... */ }

// 통합 인터페이스
async function getConnection(dbName) {
  const config = getDbConfig(dbName);
  if (['sqlserver', 'postgresql', 'mysql'].includes(config.provider)) {
    return getPrismaClient(dbName);
  }
  return getNativePool(dbName);
}
```

### 5.3 저장 프로시저 실행 (기존 호환)

기존 VMES의 SP를 그대로 활용하기 위한 범용 SP 실행기:

```javascript
// src/services/procedureService.js
async function callProcedure(procName, params, dbName) {
  const conn = await getConnection(dbName);
  
  if (conn.type === 'prisma') {
    // Prisma의 $queryRaw 또는 $executeRaw 사용
    return await conn.client.$queryRawUnsafe(
      `EXEC ${procName} ${buildParamList(params)}`
    );
  } else {
    // Native 드라이버 사용
    return await conn.pool.execute(procName, params);
  }
}
```

---

## 6. API 설계 규칙

### 6.1 기본 구조

```
[METHOD] /api/v1/[module]/[resource]
```

| 범주 | 경로 예시 | 설명 |
|------|-----------|------|
| 인증 | `POST /api/v1/auth/login` | 로그인 |
| 인증 | `POST /api/v1/auth/logout` | 로그아웃 |
| SP실행 | `POST /api/v1/procedure/execute` | 범용 SP 실행 |
| SP실행 | `POST /api/v1/procedure/transaction` | 트랜잭션 SP 실행 |
| 생산 | `GET /api/v1/manufacturing/work-orders` | 작업지시 조회 |
| 품질 | `GET /api/v1/quality/inspections` | 검사 현황 |
| 재고 | `GET /api/v1/inventory/stock` | 재고 현황 |
| 시스템 | `GET /api/v1/system/users` | 사용자 조회 |
| 시스템 | `GET /api/v1/system/codes` | 공통 코드 조회 |
| DB설정 | `GET /api/v1/dbconfig/connections` | DB 연결 목록 |

### 6.2 공통 응답 형식

```json
{
  "success": true,
  "data": { },
  "message": "조회 성공",
  "meta": {
    "total": 100,
    "page": 1,
    "pageSize": 20
  }
}
```

에러 응답:
```json
{
  "success": false,
  "error": {
    "code": "AUTH_INVALID_TOKEN",
    "message": "인증 토큰이 만료되었습니다."
  }
}
```

### 6.3 범용 SP 실행 API (KASWIN 방식 계승)

기존 VMES SP를 직접 호출해야 하는 경우:

```
POST /api/v1/procedure/execute
Content-Type: application/json

{
  "procName": "WIP_LOTSTS_SEL01",
  "params": {
    "BIZ_CD": "1000",
    "PLANT_CD": "A",
    "LOT_NO": "LOT20240101001"
  },
  "dbName": "KASWIN"
}
```

---

## 7. 모듈 매핑 (C# → Web)

기존 VMES 모듈을 VWeb 모듈로 매핑:

| 기존 C# 모듈 | VWeb 모듈 | 라우트 접두사 | 화면 ID 패턴 |
|--------------|-----------|-------------|-------------|
| PsSystem | system | `/api/v1/system` | BA_____ |
| PsManufacturing | manufacturing | `/api/v1/manufacturing` | MF_____ |
| PsQuality | quality | `/api/v1/quality` | QM/CT/QA_____ |
| PsInventory | inventory | `/api/v1/inventory` | WH/LT_____ |
| PsSales | sales | `/api/v1/sales` | SA_____ |
| PsPurchase | purchase | `/api/v1/purchase` | PU_____ |
| PsMaintenance | maintenance | `/api/v1/maintenance` | MT_____ |
| PsKPI | kpi | `/api/v1/kpi` | KPI_____ |
| PsFMB | monitoring | `/api/v1/monitoring` | FMB_____ |
| PsWMS | warehouse | `/api/v1/warehouse` | WMS_____ |

### 화면 ID 체계 유지

기존 VMES의 화면 ID 체계를 그대로 사용:
- `MF12020M` → 생산 > 작업지시 > 작업지시 진행현황 > 관리
- 패턴: `[모듈코드][대분류][중분류][일련번호][유형]`
- 유형: `M`(관리), `Q`(조회)

---

## 8. 인증 및 권한

### 8.1 인증 흐름

```
[브라우저] → POST /api/v1/auth/login
         → { userId, password, dbName }
         → 세션 생성 + JWT 토큰 발급
         → 세션에 selectedDB 저장
```

### 8.2 역할 기반 접근 제어 (RBAC)

기존 VMES의 역할 체계 활용:
- `ROLE_CD`: 역할 코드
- 화면별 권한: 조회(R), 등록(C), 수정(U), 삭제(D), 출력(P)
- 미들웨어에서 라우트 접근 전 권한 확인

```javascript
// src/middleware/auth.js
function authorize(screenId, permission) {
  return async (req, res, next) => {
    const userRole = req.session.loginData.ROLE_CD;
    const hasPermission = await checkPermission(userRole, screenId, permission);
    if (!hasPermission) return res.status(403).json({ success: false });
    next();
  };
}

// 사용 예
router.get('/work-orders', authorize('MF12020M', 'R'), getWorkOrders);
router.post('/work-orders', authorize('MF12020M', 'C'), createWorkOrder);
```

---

## 9. 개발 순서 (권장)

### Phase 1: 기반 구축
1. 프로젝트 초기화 (package.json, 폴더 구조)
2. Express 서버 + 기본 미들웨어 설정
3. 다중 DB 커넥션 풀 구현 (KASWIN db.js 참고)
4. Prisma 스키마 설정
5. 범용 SP 실행 API 구현 (`/callProc`, `/callProcTrans`)
6. 인증 시스템 (로그인/세션/DB선택)

### Phase 2: 시스템 관리 (BA)
1. 사용자 관리
2. 역할/권한 관리
3. 공통 코드 관리
4. 조직 구조 (사업장/공장/작업장/라인/부서)

### Phase 3: 생산 관리 (MF)
1. 생산계획 조회/확정
2. 작업지시 등록/조회/진행현황
3. 생산실적 등록/조회
4. 자재 소모 현황

### Phase 4: 품질 관리 (QM)
1. 검사 기준 등록
2. 검사 등록/조회
3. 부적합 관리

### Phase 5: 재고/창고 (WH)
1. 재고 현황
2. 입출고 관리
3. LOT 추적

### Phase 6: 영업/구매 (SA/PU)
1. 수주 관리
2. 구매 관리

### Phase 7: 설비/KPI/모니터링
1. 설비 보전
2. KPI 대시보드
3. 공장 모니터링 보드 (FMB)

---

## 10. 환경 설정

### .env 파일 구조

```env
# 서버
NODE_ENV=development
PORT=3000
HTTPS_PORT=3443

# 세션
SESSION_SECRET=your-session-secret-key

# 기본 DB (Prisma)
DB_PROVIDER=sqlserver
DATABASE_URL=sqlserver://localhost:1433;database=VMESDB;user=sa;password=xxx;encrypt=true;trustServerCertificate=true

# 추가 DB (Native 드라이버)
ORACLE_CONNECTION_STRING=...
TIBERO_CONNECTION_STRING=...
```

### package.json 핵심 의존성

```json
{
  "dependencies": {
    "express": "^4.21.x",
    "prisma": "^6.x",
    "@prisma/client": "^6.x",
    "mssql": "^11.x",
    "oracledb": "^6.x",
    "ejs": "^3.1.x",
    "express-session": "^1.18.x",
    "cors": "^2.8.x",
    "helmet": "^8.x",
    "jsonwebtoken": "^9.x",
    "dotenv": "^16.x",
    "winston": "^3.x",
    "swagger-jsdoc": "^6.x",
    "swagger-ui-express": "^5.x",
    "devextreme": "^25.x"
  },
  "devDependencies": {
    "nodemon": "^3.x",
    "jest": "^29.x"
  }
}
```

---

## 11. KASWIN에서 재사용 가능한 코드

VMES_KASWIN 프로젝트에서 직접 가져올 수 있는 코드:

| 파일 | 용도 | 위치 |
|------|------|------|
| `db.js` | 다중 DB 커넥션 풀 | `VMES_KASWIN/src/db.js` |
| `serverProcess.js` | SP 실행 로직 | `VMES_KASWIN/src/serverProcess.js` |
| `SerialCommunication.js` | 시리얼 통신 | `VMES_KASWIN/src/SerialCommunication.js` |
| `dbConfigManager.js` | DB 설정 관리 | `VMES_KASWIN/utils/dbConfigManager.js` |
| `Basedesign01.ejs` | 공통 화면 템플릿 | `VMES_KASWIN/views/PsCommon/` |
| `routes/index.js` | 라우트 패턴 참고 | `VMES_KASWIN/routes/index.js` |
| `public/js/` | 프론트엔드 유틸 | `VMES_KASWIN/public/js/` |

---

## 12. 참고 문서

| 문서 | 위치 | 내용 |
|------|------|------|
| 테이블명세서 | `VMES/Documents/테이블명세서.xlsx` | DB 테이블 스키마 정의 |
| 다중DB가이드 | `VMES_KASWIN/docs/multi-db-usage.md` | 다중 DB 사용법 |
| 템플릿가이드 | `VMES_KASWIN/docs/base-template-guide.md` | 공통 템플릿 사용법 |
| 시리얼통신가이드 | `VMES_KASWIN/docs/시리얼통신_바코드스캐너_구현가이드.md` | 바코드 스캐너 연동 |

---

## 13. 주요 설계 결정 사항

### 결정 1: SP 기반 vs ORM 기반
**결정**: 하이브리드 (SP + Prisma 병행)
- 기존 SP가 있는 기능 → SP 호출 (`/api/v1/procedure/execute`)
- 신규 기능 → Prisma 모델 사용
- 점진적으로 Prisma 전환 가능

### 결정 2: 프론트엔드 방식
**결정**: EJS + DevExtreme (SSR) → 추후 React/Vue 전환 가능
- 초기에는 KASWIN 방식 유지 (EJS 서버사이드 렌더링)
- DevExtreme 컴포넌트로 그리드/차트 처리
- API가 REST 기반이므로 추후 SPA 프론트엔드로 전환 용이

### 결정 3: 다중 DB 전환 방식
**결정**: 세션 기반 DB 선택 (KASWIN 방식 계승)
- 로그인 시 사용할 DB를 선택
- 세션에 `selectedDB` 저장
- 모든 API 호출 시 세션의 DB 사용

### 결정 4: 화면 ID 체계
**결정**: 기존 VMES 화면 ID 체계 그대로 유지
- 기존 사용자 교육 비용 최소화
- SP 이름과의 연관성 유지
