# Spec Sources Index

프로젝트 spec.md 파일 경로 인덱스

**Last Updated**: 2025-12-22

---

## Kubernetes Namespace 구조

| Namespace | Services | 설명 |
|-----------|----------|------|
| `finplatform` | Auth API, AIFAP-BT API | 메인 트레이딩 플랫폼 |
| `fin-crypto` | CoinAPI | 암호화폐 시세 데이터 |
| `fin-finwatcher` | RqueryAPI, PyfolioAPI | 모니터링 및 성과 분석 |
| `fin-finwrapper` | BitgetTrader API | 거래소 주문 실행 래퍼 |

---

## Project Paths (로컬 경로)

> **Note**: 실제 경로는 `.env` 파일에서 환경변수로 관리됩니다. `.env.example` 참고.

| Project | Namespace | Path (env var) | spec.md | Status |
|---------|-----------|----------------|---------|--------|
| finplatform | `finplatform` | `${FINPLATFORM_PATH}` | CLAUDE.md | OK |
| coinapi | `fin-crypto` | `${COINAPI_PATH}` | spec.md | OK |
| bitgettrader | `fin-finwrapper` | `${BITGETTRADER_PATH}` | spec.md | OK |
| rqueryapi | `fin-finwatcher` | `${RQUERYAPI_PATH}` | spec.md | OK |
| pyfolioapi | `fin-finwatcher` | `${PYFOLIOAPI_PATH}` | spec.md | OK |

---

## 1. finplatform (Main Trading Platform) - CORE

경로: `${FINPLATFORM_PATH}`
네임스페이스: `finplatform`

**필수 참조**: CLAUDE.md (전체 시스템 아키텍처 포함)

**포함 서비스**:
- Auth API (인증)
- AIFAP-BT API (자동매매)

### 핵심 구조
- **aifap_main/**: 자동매매 시스템 코어
  - `futures_rebalancing_main.py` - 선물 리밸런싱 메인
  - `futures_rebalancing_strategy.py` - 선물 전략 실행
  - `spot_rebalancing_main.py` - 현물 리밸런싱 메인
  - `module/` - 거래 모듈 (futures/, spot/, common/)
  - `monitoring/` - Finwatcher (DB 저장)

- **api/**: FastAPI 라우터
  - `routers/command.py` - 시스템 실행/종료 API

- **auth/**: 인증 API 서비스

### 실행 흐름
```
API 요청 → command.py → subprocess.Popen()
    → rebalancing_main.py (3개 프로세스: strategy, finwatcher, trading)
    → 전략 실행 → order_info.txt 생성 → auto_trade() 주문 실행
```

### 문서
- docs/architecture_v2_plan.md
- docs/api_reference.md
- docs/api_integration_flow.md

---

## 2. coinapi (Crypto Data API)

경로: `${COINAPI_PATH}`
네임스페이스: `fin-crypto`

- spec.md - API 스펙
- CLAUDE.md - 프로젝트 가이드

**역할**: OHLCV 시세 데이터 제공 (Go 기반)

---

## 3. bitgettrader (Trading Execution API)

경로: `${BITGETTRADER_PATH}`
네임스페이스: `fin-finwrapper`

- spec.md - API 스펙
- CLAUDE.md - 프로젝트 가이드

**역할**: Bitget 거래소 주문 실행 래퍼 API

---

## 4. rqueryapi (Result Query API)

경로: `${RQUERYAPI_PATH}`
네임스페이스: `fin-finwatcher`

- spec.md - API 스펙
- CLAUDE.md - 프로젝트 가이드

**역할**: Finwatcher DB 결과 조회 API

---

## 5. pyfolioapi (Performance Analysis API)

경로: `${PYFOLIOAPI_PATH}`
네임스페이스: `fin-finwatcher`

- spec.md - API 스펙
- CLAUDE.md - 프로젝트 가이드

**역할**: Pyfolio 기반 성과 분석 API

---

## Update Log

| Date | Change | By |
|------|--------|-----|
| 2025-12-22 | Kubernetes 네임스페이스 구조 추가 | Claude |
| 2025-12-19 | rqueryapi, pyfolioapi 경로 수정 (eng_finwatcher 하위) | Claude |
| 2025-12-19 | finplatform 핵심 구조 상세 추가 | Claude |
| 2025-12-19 | 경로 검증 및 실제 존재하는 파일로 업데이트 | Claude |
| 2025-12-19 | Initial index created | Claude |

---

## Notes

- finplatform이 메인 시스템 - CLAUDE.md에 전체 아키텍처 상세 설명
- rqueryapi, pyfolioapi는 eng_finwatcher 프로젝트 하위에 위치
- 모든 프로젝트 경로 검증 완료 (OK)
- **Namespace 분리**: 서비스별로 다른 네임스페이스에서 운영
  - `finplatform`: Auth API, AIFAP-BT API
  - `fin-crypto`: CoinAPI
  - `fin-finwatcher`: RqueryAPI, PyfolioAPI
  - `fin-finwrapper`: BitgetTrader API
