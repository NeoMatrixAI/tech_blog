# CLAUDE.md - Tech Blog Project

## Project Overview

기술 블로그 작성 프로젝트 - AIFAP (AI Finance Agent Platform) 시스템 인프라 관련 기술 콘텐츠 제작

> 현재 Bitget 거래소 연동. Binance, Bybit 등 주요 거래소 지원 예정.

## Project Structure

```
tech_blog/
├── README.md          # 프로젝트 개요
├── CLAUDE.md          # 이 파일 (Claude 가이드)
├── drafts/            # 블로그 초안
├── published/         # 발행된 최종본
├── specs/             # API spec 문서 인덱스
└── assets/            # 다이어그램, 이미지
```

## Blog Categories

### 시스템 (System)
기술 블로그의 큰 틀:

- **Infra 및 업데이트**: Kubernetes, Docker, 인프라 변경사항
- **시스템 이슈**: 장애 대응, 버그 수정, 성능 문제
- **삽질 기록**: 트러블슈팅, 시행착오, 교훈
- **기타**: 도구, 팁, 일반 기술 노트

## Writing Guidelines

### Format Rules
- **Markdown 작성**: 모든 글은 .md 파일로 작성
- **발행 방식**: 미리보기에서 Ctrl+C → Medium/Substack에 Ctrl+V
- **Markdown 제한**: Medium/Substack은 복잡한 마크다운 미지원
  - 표(table), mermaid, ASCII 다이어그램 등 렌더링 안됨
  - 단순한 형식만: 헤더, 볼드, 리스트, 코드블록
- **들여쓰기 제한**: 1단계까지만 (Medium 제약)
  - OK: `- 항목` → `  - 하위항목`
  - NG: `    - 2단계 이상 들여쓰기`
- **이미지화 필수 항목**:
  - 표/테이블
  - ASCII 아트 다이어그램
  - 복잡한 구조도

### Image Handling

이미지가 필요한 부분은 Claude가 `[IMAGE NEEDED]` 블록으로 명시적으로 표시합니다.
사용자가 직접 이미지를 제작하므로, **매우 상세하게** 작성해야 합니다.

#### 필수 포함 항목

모든 `[IMAGE NEEDED]` 블록에는 다음 항목을 반드시 포함:

```
[IMAGE NEEDED]
- 파일명: (이미지 저장용 파일명, 예: 01_system_architecture)
- 타입: (다이어그램 종류)
- 목적: (이 이미지가 설명하려는 것)
- 내용: (구체적인 요소들)
- 레이아웃: (배치 방향 및 구조)
- 연결: (화살표, 선 등 요소 간 관계)
- 스타일: (색상, 테마, 폰트 등)
- 참고: (유사 이미지, 영감 소스 - 선택)
```

#### 이미지 타입별 작성 가이드

**1. 아키텍처 다이어그램 (Architecture Diagram)**
```
[IMAGE NEEDED]
- 타입: 시스템 아키텍처 다이어그램
- 목적: AIFAP 전체 시스템 구조와 데이터 흐름 설명
- 내용:
  - 레이어 1 (최상단): "External APIs" 영역
    - 박스 3개: "Bitget API" (주황색), "Auth API" (파란색), "Rquery API" (녹색)
    - 각 박스 크기: 동일, 수평 배치
  - 레이어 2 (중앙): "Kubernetes Cluster" 큰 경계 박스
    - 내부 왼쪽: "Istio Gateway" 박스 (보라색)
    - 내부 중앙: "AIFAP Pod" 박스 (주요 컴포넌트, 크게)
      - Pod 내부 표시: "FastAPI Server", "Trading Processes x3"
    - 내부 오른쪽: "Config" 작은 박스
  - 레이어 3 (하단): "Data Layer" 영역
    - "PostgreSQL (finplatform)" 원통형 DB 아이콘
    - "PostgreSQL (finwatcher)" 원통형 DB 아이콘
    - "Grafana" 대시보드 아이콘
- 레이아웃: 수직 3단 레이어, 좌우 대칭
- 연결:
  - External APIs → Istio Gateway: 단방향 화살표 (요청)
  - Istio Gateway → AIFAP Pod: 양방향 화살표
  - AIFAP Pod → PostgreSQL DBs: 양방향 화살표 (읽기/쓰기)
  - PostgreSQL (finwatcher) → Grafana: 단방향 화살표 (데이터 조회)
- 스타일:
  - 배경: 밝은 회색 또는 흰색
  - Kubernetes 영역: 점선 경계
  - 컴포넌트별 색상 구분 (위에 명시)
  - 화살표: 실선, 끝에 삼각형 머리
  - 폰트: Sans-serif, 읽기 쉬운 크기
- 참고: AWS/GCP 아키텍처 다이어그램 스타일
```

**2. 프로세스/흐름 다이어그램 (Process Flow Diagram)**
```
[IMAGE NEEDED]
- 타입: 프로세스 상호작용 다이어그램
- 목적: 3개 프로세스 간 통신 및 실행 순서 설명
- 내용:
  - 박스 1 (왼쪽): "Strategy Process"
    - 내부 단계 (위→아래):
      1. "Execute strategy.py"
      2. "Calculate target weights"
      3. "Write order_info.txt"
    - 아이콘: 뇌 또는 계산기 심볼
  - 박스 2 (오른쪽): "Main Process"
    - 내부 단계 (위→아래):
      1. "Wait for strategy signal"
      2. "Read order_info.txt"
      3. "Execute orders (batch of 5)"
      4. "Set SL/TP"
    - 아이콘: 실행/재생 심볼
  - 박스 3 (하단 중앙): "Finwatcher Process"
    - 내부: "Every 60s: Collect metrics → Save to DB"
    - 아이콘: 차트/모니터 심볼
- 레이아웃:
  - Strategy와 Main: 상단에 좌우 배치
  - Finwatcher: 하단 중앙에 독립 배치
- 연결:
  - Strategy ↔ Main: 양방향 점선 화살표, 라벨 "multiprocessing.Event"
  - Strategy → Main: "strategy_done_event" 라벨
  - Main → Strategy: "autotrade_done_event" 라벨
  - Finwatcher는 독립: 다른 박스와 연결 없음, "Independent" 라벨
- 스타일:
  - 박스: 둥근 모서리, 그림자 효과
  - 색상: Strategy(파란색), Main(녹색), Finwatcher(주황색)
  - 단계 번호: 원 안에 숫자
```

**3. 데이터 흐름 다이어그램 (Data Flow / Flowchart)**
```
[IMAGE NEEDED]
- 타입: 수직 데이터 흐름 다이어그램
- 목적: 리밸런싱 사이클의 5단계 과정 설명
- 내용:
  - Stage 1 "전략 실행" (최상단):
    - 시작점: 타원형 "Start"
    - 과정: "Load strategy.py" → "Fetch OHLCV data" → "Calculate weights"
    - 출력: 박스 "{BTC: 0.5, ETH: 0.5}"
  - Stage 2 "비중 비교":
    - 과정: "Get current positions" → "Calculate current weights" → "Compute diff"
    - 출력: 박스 "{BTC: +0.2, ETH: +0.3}"
  - Stage 3 "주문 생성":
    - 과정: "Convert to sizes" → "Add SL/TP prices" → "Write order_info.txt"
  - Stage 4 "주문 실행":
    - 과정: "Sort orders (close first)" → "Batch execute (5 at a time)" → "Set SL/TP"
  - Stage 5 "모니터링" (최하단):
    - 과정: "Query balance" → "Calculate PV, ROE" → "Save to PostgreSQL"
    - 종료점: 타원형 "End / Wait for next cycle"
- 레이아웃:
  - 수직 방향 (위→아래)
  - 각 Stage는 수평으로 구분된 영역
  - Stage 번호: 왼쪽에 큰 원형 배지 (1, 2, 3, 4, 5)
- 연결:
  - 각 Stage 사이: 굵은 아래 방향 화살표
  - Stage 내부 과정: 가는 화살표
- 스타일:
  - 과정 박스: 직사각형
  - 데이터 박스: 평행사변형 또는 둥근 직사각형
  - Stage별 배경색 그라데이션 (밝은 색 → 진한 색)
  - 전체 배경: 흰색 또는 연한 회색
```

**4. 테이블/비교표 (Table as Image)**
```
[IMAGE NEEDED]
- 타입: 기술 스택 테이블
- 목적: AIFAP에서 사용하는 기술과 선택 이유 설명
- 내용 (컬럼: Category | Technology | Purpose):
  | Category      | Technology          | Purpose                              |
  |---------------|---------------------|--------------------------------------|
  | Language      | Python 3.8          | 금융 분석 라이브러리 풍부             |
  | API Framework | FastAPI + Uvicorn   | 비동기, 자동 문서화, 고성능          |
  | Container     | Docker + Kubernetes | 클라우드 네이티브 배포, 오케스트레이션 |
  | Service Mesh  | Istio               | TLS 종료, 트래픽 관리, 관측성        |
  | Database      | PostgreSQL          | 안정성, 시계열 쿼리 최적화           |
  | Monitoring    | Grafana             | 대시보드, 알림 설정                  |
  | Analysis      | Zipline (modified)  | 백테스팅 호환성                      |
- 레이아웃:
  - 헤더 행: 진한 배경색 (네이비 또는 다크 그레이)
  - 데이터 행: 교차 색상 (흰색/연한 회색)
  - 첫 번째 열: 약간 진한 배경
- 스타일:
  - 테두리: 얇은 선, 회색
  - 폰트: 헤더는 볼드, 데이터는 레귤러
  - 아이콘: 각 Technology 옆에 작은 로고 아이콘 (선택)
  - 전체 모서리: 둥글게 처리
```

**5. 대시보드 목업 (Dashboard Mockup)**
```
[IMAGE NEEDED]
- 타입: Grafana 스타일 대시보드 목업
- 목적: 실제 모니터링 화면이 어떻게 보이는지 시각화
- 내용:
  - 상단 행 (Stat Panels): 3개 수치 패널
    - Panel 1: "ROE" → "+12.5%" (녹색 텍스트, 위 화살표)
    - Panel 2: "Sharpe Ratio" → "1.8" (파란색 텍스트)
    - Panel 3: "Max Drawdown" → "-5.2%" (빨간색 텍스트, 아래 화살표)
  - 중간 행 (Time Series Chart):
    - 제목: "Portfolio Value Over Time"
    - X축: 날짜/시간 (최근 7일)
    - Y축: 달러 가치 ($)
    - 라인: 상승 추세의 곡선, 녹색
    - 배경 그리드: 연한 선
  - 하단 행 (Allocation Chart):
    - 왼쪽: 파이 차트 "Position Allocation"
      - BTC: 50% (주황색)
      - ETH: 30% (파란색)
      - SOL: 20% (보라색)
    - 오른쪽: 바 차트 "Position Sizes"
      - 각 심볼별 수평 바
- 레이아웃:
  - 3행 그리드 (상단: 1/4, 중간: 1/2, 하단: 1/4 높이 비율)
  - 반응형 느낌의 카드 레이아웃
- 스타일:
  - 테마: Grafana 다크 모드 (배경 #1a1a1a, 카드 #2a2a2a)
  - 텍스트: 흰색
  - 차트 라인: 네온 녹색 (#00ff88)
  - 패널 테두리: 미세한 경계선 또는 그림자
- 참고: Grafana 공식 데모 대시보드 스타일
```

#### 작성 시 주의사항

1. **구체적 수치/예시 포함**: "데이터 표시" 대신 "BTC: 0.5, ETH: 0.3" 처럼 실제 예시
2. **위치 명확히**: "왼쪽", "상단", "중앙" 등 정확한 위치 지정
3. **색상 명시**: "파란색" 보다는 "네이비 블루 #1a237e" 또는 "밝은 파란색"
4. **크기 비율**: "큰 박스", "작은 아이콘" 등 상대적 크기 표현
5. **화살표 방향**: 단방향/양방향, 실선/점선, 라벨 유무 명시
6. **참고 자료**: 유사한 스타일의 이미지나 출처 언급 (선택)

### Target Platforms
- **Medium**: 기술 심층 분석, 코드 스니펫
- **Substack**: 뉴스레터 형식, 업데이트 포스트

### Target Audience
- Developers/Engineers: 시스템 아키텍처, 코드 구현, DevOps
- Investors/Non-developers: 비즈니스 가치, 시스템 신뢰성

### Language
- 블로그 발행: English (원문)
- 한글 번역본: 사용자 검토용 (_ko.md)
- 내부 문서: Korean 허용

### File Naming Convention
```
drafts/
├── 01_system_overview.md      # 영문 (발행용)
├── 01_system_overview_ko.md   # 한글 (검토용)
└── ...

published/
├── 01_system_overview.md      # 영문 원본만 보관
└── ...
```

## Workflow

1. `drafts/00_master_draft.md` 참조하여 시스템 컨텍스트 파악
2. `specs/spec_sources.md`에서 API spec 위치 확인
3. 카테고리에 맞는 초안 작성 (단순 마크다운)
4. 이미지 필요 부분 표시 → 사용자가 이미지 추가
5. 미리보기 → 복사 → 플랫폼에 붙여넣기
6. 최종본 `published/`로 이동

## Source Reference

- **Main Project**: AIFAP (AI Finance Agent Platform)
- **Location**: `${FINPLATFORM_PATH}` (see `.env.example`)
- **Current Version**: `${AIFAP_VERSION}` (default: v01.00.06)
- **Current Exchange**: Bitget (Binance, Bybit 등 지원 예정)

> **Note**: 실제 경로는 `.env` 파일에서 관리됩니다. `.env.example`을 참고하여 설정하세요.

## Kubernetes Namespace Structure

AIFAP 시스템은 4개의 Kubernetes 네임스페이스에 분산 배포됩니다:

| Namespace | Services | 역할 |
|-----------|----------|------|
| `finplatform` | Auth API, AIFAP API | 핵심 트레이딩 플랫폼 |
| `fin-crypto` | CoinAPI | 암호화폐 OHLCV 시세 데이터 |
| `fin-finwatcher` | RqueryAPI, PyfolioAPI | 모니터링 및 성과 분석 |
| `fin-finwrapper` | BitgetTrader API | 거래소 주문 실행 래퍼 |

### 네임스페이스별 상세

**finplatform** (핵심)
- Auth API: 사용자 인증/권한
- AIFAP API: 자동매매 시스템 메인 서비스

**fin-crypto** (시장 데이터)
- CoinAPI: Go 기반 OHLCV 데이터 서비스

**fin-finwatcher** (분석)
- RqueryAPI: 트레이딩 결과 조회
- PyfolioAPI: Pyfolio 기반 성과 분석

**fin-finwrapper** (주문 실행)
- BitgetTrader API: Bitget 거래소 주문 실행 래퍼

### 크로스 네임스페이스 통신

AIFAP Pod는 다른 네임스페이스의 서비스와 통신:
- `fin-crypto` → OHLCV 데이터 조회
- `fin-finwatcher` → 결과 조회, 성과 분석
- `fin-finwrapper` → 주문 실행
