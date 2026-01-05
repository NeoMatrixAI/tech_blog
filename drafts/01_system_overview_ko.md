# AIFAP: 쿠버네티스 네이티브 암호화폐 트레이딩 플랫폼 아키텍처

> 프로덕션급 자동매매 시스템을 처음부터 구축하기

**카테고리**: 업데이트
**읽는 시간**: 약 10분
**태그**: #kubernetes #python #trading #fintech #devops #cryptocurrency

---

## 서론

암호화폐 자동매매 플랫폼을 만들기 시작했을 때, 목표는 단순했습니다: 포트폴리오 리밸런싱 전략을 24시간 365일 수동 개입 없이 안정적으로 실행할 수 있는 시스템을 만드는 것. 하지만 프로젝트가 발전하면서 더 큰 비전이 생겼습니다—**멀티 매니저 시스템**을 구축하는 것. 여러 명의 매니저가 각각 여러 명의 트레이더를 관리하고, 이 모든 것이 하나의 슈퍼바이저 아래에서 조율됩니다.

이 글에서는 제가 개발하고 프로덕션에서 운영 중인 쿠버네티스 네이티브 트레이딩 시스템 **AIFAP**(AI Finance Agent Platform)의 아키텍처를 소개합니다. 현재 Bitget 거래소와 연동되어 있으며, Binance, Bybit 등 주요 거래소 지원 예정입니다. 첫 번째 글에서는 전체 아키텍처와 설계 결정을 다룹니다. 이후 글에서는 각 컴포넌트를 깊이 있게 살펴볼 예정입니다.

### 멀티 매니저 비전

AIFAP의 궁극적인 목표는 단일 트레이딩 봇을 넘어섭니다:

- **슈퍼바이저 (관리자)**: 전체 시스템을 관리하는 최상위 관리자
- **매니저**: 슈퍼바이저에 의해 고용되며, 각 매니저는 자신의 트레이더 팀을 감독
- **트레이더**: 매니저의 감독 하에 개별 전략을 실행

헤지펀드 구조와 비슷합니다: 하나의 펀드(슈퍼바이저)가 여러 포트폴리오 매니저를 고용하고, 각 매니저는 여러 트레이딩 전략을 운영합니다. AIFAP는 이 전체 계층 구조를 자동화하는 것을 목표로 합니다.

---

## AIFAP는 무엇을 하는가?

AIFAP의 핵심은 암호화폐 선물 및 현물 거래를 위한 **포트폴리오 리밸런싱 시스템**입니다. 현재 Bitget에서 운영 중이며, 추가 거래소 지원이 곧 예정되어 있습니다. 실제로 다음과 같이 동작합니다:

1. **사용자 정의 전략**: 사용자가 시장 데이터를 분석하고 목표 포트폴리오 비중을 출력하는 Python 함수 작성
2. **자동 리밸런싱**: 시스템이 주기적으로 현재 포지션과 목표 비중을 비교하여 맞추기 위한 거래 실행
3. **실시간 모니터링**: 매분마다 성과 지표를 계산하고 저장하여 시각화
4. **Discord 알림**: 에러 및 중요 이벤트 발생 시 Discord로 알림 전송

### 간단한 예시

사용자가 모멘텀 전략을 다음과 같이 정의할 수 있습니다:

```python
def strategy(context, config_dict):
    """간단한 모멘텀 전략"""
    assets = ['BTCUSDT', 'ETHUSDT', 'SOLUSDT']

    # 20봉 수익률 계산
    prices = context.get_history(assets, window=100, frequency='1m', fields='close')
    returns = prices.pct_change(20)

    # 양의 모멘텀 자산에 동일 비중 배분
    weights = {}
    positive = [a for a in assets if returns[a].iloc[-1] > 0]

    if positive:
        for asset in positive:
            weights[asset] = {
                'weight': 1.0 / len(positive),
                'presetStopSurplusPrice': prices[asset].iloc[-1] * 1.05,
                'presetStopLossPrice': prices[asset].iloc[-1] * 0.97
            }

    return weights
```

시스템이 나머지를 처리합니다: 시장 데이터 조회, 포지션 차이 계산, 주문 실행, 손절가 설정, 성과 추적.

---

## 아키텍처 개요

전체 구조를 보여드리겠습니다.

![System_Architecture](../images/01_system_overview/01_system_architecture.png)
<!-- [IMAGE NEEDED]
- 파일명: 01_system_architecture
- 타입: 시스템 아키텍처 다이어그램
- 목적: AIFAP 전체 시스템 구조와 Kubernetes 네임스페이스, 크로스 네임스페이스 통신 및 데이터 흐름 표시
- 내용:
  - 레이어 1 (상단): "External APIs" 영역
    - "Bitget Exchange API" (주황색, #FF6B35) - 외부 거래소, 상단 중앙 배치
  - 레이어 2 (메인): "Kubernetes Cluster" 큰 경계 박스 (점선 테두리)
    - **Namespace 1: finplatform** (파란색 경계, #E3F2FD) - 왼쪽
      - "Istio Gateway" 박스 (보라색, #9B59B6)
      - "Auth API Pod" 박스 (파란색, #4A90D9)
      - "AIFAP Pod" 메인 박스 (가장 큰 컴포넌트)
        - Pod 내부: 상단에 "FastAPI Server" 라벨
        - Pod 내부: 하단에 "Trading Processes x3" 라벨
      - "ConfigMap" 작은 박스 (회색)
    - **Namespace 2: fin-crypto** (주황색 경계, #FFF3E0) - 중앙
      - "CronJob: OHLCV Collector" 박스 (노란색, #FFC107) - 작게, 스케줄 작업 표시
        - 라벨: "Bitget에서 실시간 데이터 수집"
      - "CoinAPI Pod" 박스 (주황색, #FF6B35)
        - 라벨: "OHLCV 데이터 서비스 (DB 데이터 서빙)"
    - **Namespace 3: fin-finwatcher** (녹색 경계, #E8F5E9)
      - "RqueryAPI Pod" 박스 (녹색, #2ECC71)
      - "PyfolioAPI Pod" 박스 (청록색, #26A69A)
      - 라벨: "Monitoring & Analysis Services"
    - **Namespace 4: fin-finwrapper** (빨간색 경계, #FFEBEE) - 오른쪽, 강조된 위치
      - "BitgetTrader Pod" 박스 (빨간색, #E53935) - 더 크게, 핵심 컴포넌트로 강조
      - 라벨: "Exchange Gateway"
      - 서브 라벨: "주문, 포지션, 계정 데이터"
  - 레이어 3 (하단): "Data Layer" 영역
    - "PostgreSQL (finplatform)" 원통형 DB 아이콘 (파란색)
    - "PostgreSQL (fin-crypto)" 원통형 DB 아이콘 (주황색) - OHLCV 데이터 저장
    - "PostgreSQL (finwatcher)" 원통형 DB 아이콘 (녹색)
    - "Grafana" 대시보드 아이콘 (주황색)
- 레이아웃: 수직 3단 아키텍처, 네임스페이스가 그룹 영역으로 표시
- 연결:
  **데이터 수집 흐름 (OHLCV) - 번호 순서:**
  - Bitget Exchange API → CronJob: 단방향 아래 화살표 (라벨 "1. Fetch OHLCV")
  - CronJob → PostgreSQL (fin-crypto): 단방향 아래 화살표 (라벨 "2. Store")
  - PostgreSQL (fin-crypto) → CoinAPI Pod: 단방향 위 화살표 (라벨 "3. Read") - DB가 CoinAPI에 데이터 제공
  - AIFAP Pod → CoinAPI Pod: 크로스 네임스페이스 화살표 왼쪽→중앙 (라벨 "4. Query OHLCV")
  **거래소 통신 흐름 - 핵심 경로 (더 굵고 강조된 화살표 사용):**
  - AIFAP Pod ↔ BitgetTrader Pod: 크로스 네임스페이스 양방향 화살표, 굵게
    - BitgetTrader로: "주문 전송"
    - BitgetTrader에서: "포지션, 잔고, 거래 결과"
  - BitgetTrader Pod ↔ Bitget Exchange API: 양방향 화살표, 굵게
    - 위로: "주문 실행"
    - 아래로: "포지션, 계정, 거래 데이터"
  - 참고: BitgetTrader는 거래소 유일한 게이트웨이 - 모든 거래소 통신이 이를 통해 수행 (주문, 포지션, 잔고, 거래 내역)
  **내부 통신:**
  - Auth API Pod ↔ AIFAP Pod: 양방향 화살표 (인증)
  - Istio Gateway → Auth API, AIFAP: 라우팅 화살표
  - AIFAP Pod ↔ PostgreSQL (finplatform): 읽기/쓰기 (설정, 사용자 데이터)
  - AIFAP Pod → RqueryAPI Pod: 크로스 네임스페이스 화살표 (결과 조회)
  - AIFAP Pod → PyfolioAPI Pod: 크로스 네임스페이스 화살표 (성과 분석)
  - RqueryAPI/PyfolioAPI ↔ PostgreSQL (finwatcher): 읽기/쓰기
  - PostgreSQL (finwatcher) → Grafana: 데이터 조회
- 스타일:
  - 배경: 밝은 회색 (#F5F5F5) 또는 흰색
  - 네임스페이스 경계: 네임스페이스 테마에 맞는 색상의 점선
  - Kubernetes 경계: 어두운 점선, 은은한 그림자
  - 크로스 네임스페이스 화살표: 더 두껍게, "cross-ns" 라벨 표시
  - 주문 실행 화살표: 특별히 굵게 (빨간색 또는 볼드)로 핵심 경로 강조
  - 데이터 흐름 화살표: 순서 표시 (1, 2, 3, 4)
  - CronJob: 점선 테두리로 스케줄 작업 표시
  - BitgetTrader: 약간 더 큰 박스, 빨간색 글로우 또는 강조로 중요성 표시
  - 화살표: 실선, 삼각형 머리
  - 폰트: Sans-serif (Inter, Roboto 또는 유사)
  - 컴포넌트 그림자: 깊이감을 위한 은은한 드롭 섀도우
- 참고: AWS/GCP 아키텍처 다이어그램 스타일, Kubernetes 네임스페이스 시각화 -->

### 핵심 컴포넌트

**Namespace: finplatform** (핵심 트레이딩 플랫폼)
1. **Auth API**: 사용자 인증 및 권한 관리
2. **AIFAP API**: FastAPI 서버가 포함된 메인 트레이딩 서비스
3. **트레이딩 프로세스**: 각 사용자 세션마다 3개의 협력 프로세스 생성

**Namespace: fin-crypto** (시장 데이터)
4. **CronJob (OHLCV Collector)**: Bitget에서 실시간 OHLCV 데이터를 수집하여 PostgreSQL에 저장하는 스케줄 작업
5. **CoinAPI**: 데이터베이스에서 읽어 AIFAP에 데이터를 제공하는 OHLCV 데이터 서비스 (Go 기반)

**Namespace: fin-finwatcher** (모니터링 및 분석)
6. **RqueryAPI**: Finwatcher 데이터베이스에서 트레이딩 결과 조회
7. **PyfolioAPI**: Pyfolio 라이브러리 기반 성과 분석

**Namespace: fin-finwrapper** (거래소 게이트웨이)
8. **BitgetTrader API**: Bitget 거래소 게이트웨이 - 주문 실행, 실시간 포지션 조회, 계정 잔고, 거래 내역 처리

**인프라**
9. **PostgreSQL 데이터베이스**: 인증/설정, OHLCV 시장 데이터, 트레이딩 결과용 별도 DB
10. **Grafana**: 실시간 성과 지표 시각화
11. **Istio**: TLS 처리, 라우팅, 크로스 네임스페이스 트래픽 관리

---

## 3개 프로세스 아키텍처

사용자가 트레이딩 세션을 시작하면 시스템이 함께 동작하는 3개의 프로세스를 생성합니다:

![Three_Process_Architecture](../images/01_system_overview/02_three_process_architecture.png)
<!-- [IMAGE NEEDED]
- 파일명: 02_three_process_architecture
- 타입: 프로세스 상호작용 다이어그램
- 목적: 3개 프로세스 간 통신 방식과 실행 순서 설명
- 내용:
  - 박스 1 (상단 왼쪽): "Strategy Process"
    - 배경색: 파란색 (#4A90D9)
    - 아이콘: 뇌 또는 계산기 심볼 (우측 상단)
    - 내부 단계 (수직 리스트, 원 안에 숫자):
      1. "strategy.py 실행"
      2. "목표 비중 계산"
      3. "order_info.txt 작성"
    - 하단 라벨: "스케줄에 따라 실행 (예: 5분마다)"
  - 박스 2 (상단 오른쪽): "Main Process"
    - 배경색: 녹색 (#2ECC71)
    - 아이콘: 재생/실행 심볼 (우측 상단)
    - 내부 단계 (수직 리스트, 원 안에 숫자):
      1. "strategy_done_event 대기"
      2. "order_info.txt 읽기"
      3. "주문 실행 (5개씩 배치)"
      4. "SL/TP 가격 설정"
    - 하단 라벨: "신호 대기"
  - 박스 3 (하단 중앙): "Finwatcher Process"
    - 배경색: 주황색 (#FF6B35)
    - 아이콘: 차트/모니터 심볼 (우측 상단)
    - 내부 내용: "60초마다:"
      - "• 계좌 잔고 조회"
      - "• PV, ROE, 지표 계산"
      - "• PostgreSQL에 저장"
    - 하단 라벨: "독립 스케줄"
- 레이아웃:
  - Strategy와 Main: 상단 행, 나란히 간격 두고 배치
  - Finwatcher: 하단 행, 중앙 정렬, 더 작은 너비
  - 전체: 역삼각형 배치
- 연결:
  - Strategy → Main: 오른쪽 방향 점선 화살표
    - 상단 라벨: "strategy_done_event"
  - Main → Strategy: 왼쪽 방향 점선 화살표
    - 하단 라벨: "autotrade_done_event"
  - 화살표 사이: "multiprocessing.Event" 라벨
  - Finwatcher: 연결선 없음
    - 근처 라벨: "Independent (동기화 불필요)"
- 스타일:
  - 박스: 둥근 모서리 (8px 반경), 은은한 그림자
  - 단계 번호: 숫자가 들어간 흰색 원
  - 화살표: 점선, 중간 두께
  - 폰트: Sans-serif, 단계 텍스트 12-14px
  - 전체 배경: 흰색 또는 매우 밝은 회색 -->

### 왜 3개 프로세스인가?

1. **관심사 분리**: 전략 로직, 주문 실행, 모니터링을 격리
2. **장애 허용**: 한 프로세스의 크래시가 다른 프로세스에 즉시 영향 안 줌
3. **타이밍 독립성**: Finwatcher는 자체 스케줄(매분)로 실행

### 이벤트 기반 동기화

Strategy 프로세스와 Main 프로세스는 `multiprocessing.Event` 객체로 통신합니다:

```python
# 프로세스 간 공유 이벤트
rebalance_start_event = multiprocessing.Event()
strategy_done_event = multiprocessing.Event()
autotrade_done_event = multiprocessing.Event()
```

이 패턴이 엄격한 순서를 보장합니다: 전략 계산 → 메인 실행 → 전략이 다음 주기 대기.

---

## 기술 스택

AIFAP를 구동하는 기술들:

![Tech_Stack_Table](../images/01_system_overview/03_tech_stack_table.png)
<!-- [IMAGE NEEDED]
- 파일명: 03_tech_stack_table
- 타입: 기술 스택 테이블
- 목적: AIFAP에서 사용하는 모든 기술과 선택 이유 표시
- 내용 (3열: 카테고리 | 기술 | 사용 이유):
  | 카테고리       | 기술                  | 사용 이유                              |
  |----------------|----------------------|----------------------------------------|
  | Language       | Python 3.8           | 풍부한 금융 라이브러리 (pandas, numpy)   |
  | API Framework  | FastAPI + Uvicorn    | 비동기 지원, 자동 OpenAPI 문서, 빠름    |
  | Container      | Docker + Kubernetes  | 클라우드 네이티브, 오케스트레이션, 스케일링 |
  | Service Mesh   | Istio                | TLS 종료, 트래픽 라우팅                 |
  | Database       | PostgreSQL           | 안정적, 시계열 쿼리에 탁월              |
  | Monitoring     | Grafana              | 아름다운 대시보드, 알림 규칙            |
  | Analysis       | Zipline (수정 버전)   | 백테스팅 호환성, context API            |
- 레이아웃:
  - 헤더 행: 어두운 배경 (네이비 #1A237E 또는 다크 그레이 #333)
  - 데이터 행: 흰색 (#FFFFFF)과 밝은 회색 (#F5F5F5) 교차
  - 첫 번째 열: 강조를 위해 약간 더 어두운 배경
  - 전체 너비: 컨테이너 채움, 비례 열 (20% / 25% / 55%)
- 스타일:
  - 테두리: 얇은 선 (#E0E0E0), 외곽에 둥근 모서리
  - 헤더 폰트: 볼드, 흰색 텍스트
  - 데이터 폰트: 레귤러 웨이트, 다크 그레이 (#333) 텍스트
  - 행 높이: 편안한 패딩 (수직 12-16px)
  - 선택사항: 각 기술명 옆에 작은 로고 아이콘 (Python 뱀, Docker 고래 등)
  - 그림자: 테이블 전체 아래 은은한 드롭 섀도우 -->

### 왜 쿠버네티스인가?

트레이딩 시스템에는 특별한 요구사항이 있습니다:

- **고가용성**: 시장 시간 중 다운타임 불가
- **리소스 격리**: 각 사용자 세션에 보장된 리소스 필요
- **쉬운 업데이트**: 활성 세션 중단 없이 롤링 배포
- **관측성**: 내장된 로깅 및 메트릭 수집

### 네임스페이스 분리

서비스들은 격리와 조직화를 위해 4개의 Kubernetes 네임스페이스에 배포됩니다:

- **finplatform**: 핵심 트레이딩 서비스 (Auth API, AIFAP API)
- **fin-crypto**: 시장 데이터 서비스 (OHLCV 데이터용 CoinAPI)
- **fin-finwatcher**: 모니터링 및 분석 (RqueryAPI, PyfolioAPI)
- **fin-finwrapper**: 주문 실행 래퍼 (BitgetTrader API)

이러한 분리는 서비스 간 보안 경계를 제공하고 각 서비스 그룹의 독립적인 스케일링과 배포를 가능하게 합니다.

---

## 데이터 흐름: 전략에서 실행까지

단일 리밸런싱 사이클의 경로를 추적해보겠습니다:

![Rebalancing_Data_Flow](../images/01_system_overview/04_rebalancing_data_flow.png)
<!-- [IMAGE NEEDED]
- 파일명: 04_rebalancing_data_flow
- 타입: 수직 데이터 흐름 다이어그램
- 목적: 단일 리밸런싱 사이클의 전체 경로를 전략부터 모니터링까지 추적
- 내용:
  - 시작점: 타원형 "리밸런싱 사이클 시작" (상단)
  - 1단계 "전략 실행" (파란색 영역 #E3F2FD):
    - 배지: 왼쪽에 "1" 원
    - 흐름 박스 (수평):
      - "strategy.py 로드" → "OHLCV 데이터 조회" → "비중 계산"
    - 출력 박스 (평행사변형): "{BTC: 0.5, ETH: 0.5}"
  - 2단계 "비중 비교" (녹색 영역 #E8F5E9):
    - 배지: 왼쪽에 "2" 원
    - 흐름 박스:
      - "현재 포지션 조회" → "현재 비중 계산" → "차이 계산"
    - 출력 박스: "{BTC: +0.2, ETH: +0.3}"
  - 3단계 "주문 생성" (노란색 영역 #FFF8E1):
    - 배지: 왼쪽에 "3" 원
    - 흐름 박스:
      - "비중을 수량으로 변환" → "SL/TP 가격 추가" → "order_info.txt 작성"
  - 4단계 "주문 실행" (주황색 영역 #FFF3E0):
    - 배지: 왼쪽에 "4" 원
    - 흐름 박스:
      - "order_info.txt 읽기" → "주문 정렬 (청산 먼저)" → "배치 실행 (5개)" → "SL/TP 설정"
  - 5단계 "모니터링" (보라색 영역 #F3E5F5):
    - 배지: 왼쪽에 "5" 원
    - 흐름 박스:
      - "잔고 조회" → "PV, ROE 계산" → "PostgreSQL에 저장"
  - 종료점: 타원형 "다음 사이클 대기" (하단)
- 레이아웃:
  - 방향: 위에서 아래로 (수직)
  - 각 단계: 밝은 배경색의 수평 밴드
  - 단계 배지: 왼쪽 가장자리, 단계 경계에 걸쳐 배치
  - 흐름 박스: 각 단계 내에서 수평 배열
  - 단계 사이: 굵은 아래 방향 화살표
- 연결:
  - 단계 내부: 박스 간 왼쪽에서 오른쪽으로 가는 얇은 화살표
  - 단계 사이: 아래 방향 굵은 화살표
  - 출력 박스: 다음 단계 입력으로 점선 화살표
- 스타일:
  - 단계 배경: 위에 명시된 파스텔 색상
  - 흐름 박스: 흰색 배경, 둥근 모서리, 얇은 테두리
  - 출력 박스: 평행사변형 형태, 약간 더 어두운 배경
  - 배지 원: 단계에 맞는 단색, 흰색 숫자
  - 화살표: 다크 그레이 (#666), 삼각형 머리
  - 전체: 깔끔하고 전문적인 플로우차트 스타일 -->

### 비중 계산

핵심 로직은 놀라울 정도로 단순합니다:

```python
# 현재 상태
current_weight = position_margin / total_allocation

# 목표 상태
target_weight = strategy_results[symbol]['weight']

# 필요한 조치
diff_weight = target_weight - current_weight

if diff_weight > threshold:
    # 더 매수
elif diff_weight < -threshold:
    # 일부 매도
```

임계값(threshold)이 작은 차이에 대한 과도한 거래를 방지합니다.

---

## 모니터링: 매분이 중요하다

Finwatcher 프로세스는 독립적으로 실행되며 매분 지표를 수집합니다:

![Metrics_Table](../images/01_system_overview/05_metrics_table.png)
<!-- [IMAGE NEEDED]
- 파일명: 05_metrics_table
- 타입: 지표 정의 테이블
- 목적: Finwatcher가 추적하는 각 성과 지표 설명
- 내용 (3열: 지표 | 공식 | 설명):
  | 지표            | 공식                              | 설명                                   |
  |-----------------|-----------------------------------|----------------------------------------|
  | PV              | 잔고 + 미실현 손익                 | 현재 시점의 총 포트폴리오 가치          |
  | ROE             | (PV - 초기값) / 초기값 × 100%     | 세션 시작 이후 자기자본 수익률          |
  | Unrealized PnL  | Σ (현재가 - 진입가) × 수량        | 오픈 포지션의 평가 손익                 |
  | Sharpe Ratio    | (수익률 - Rf) / Std(수익률)       | 위험 조정 수익률 (연간화)               |
  | Max Drawdown    | (고점 - 저점) / 고점 × 100%       | 고점에서 저점까지 최악의 하락률         |
- 레이아웃:
  - 헤더 행: 다크 블루 (#1A237E) 배경, 흰색 볼드 텍스트
  - 데이터 행: 흰색과 밝은 파란색 (#E3F2FD) 교차
  - 열 너비: 20% / 35% / 45%
  - 지표 열: 강조를 위해 볼드 텍스트
- 스타일:
  - 테두리: 얇은 선 (#BDBDBD), 둥근 모서리 (8px)
  - 폰트: 지표 이름은 볼드, 공식은 모노스페이스
  - 패딩: 셀당 10-12px
  - 그림자: 은은한 드롭 섀도우 -->

이 지표들은 PostgreSQL에 저장되고 Grafana에서 시각화됩니다:

![Grafana_Dashboard](../images/01_system_overview/06_grafana_dashboard.png)
<!-- [IMAGE NEEDED]
- 파일명: 06_grafana_dashboard
- 타입: Grafana 대시보드 목업
- 목적: 실제 모니터링 인터페이스가 어떻게 보이는지 시각화
- 내용:
  - 행 1 (상단): 3개 스탯 패널 (동일 너비, 수평)
    - 패널 1: "ROE"
      - 값: "+12.5%"
      - 색상: 녹색 (#00FF88)
      - 작은 위 화살표 아이콘
    - 패널 2: "Sharpe Ratio"
      - 값: "1.8"
      - 색상: 파란색 (#4A90D9)
      - 트렌드 아이콘 없음
    - 패널 3: "Max Drawdown"
      - 값: "-5.2%"
      - 색상: 빨간색 (#FF6B6B)
      - 작은 아래 화살표 아이콘
  - 행 2 (중간, 가장 큼): 시계열 라인 차트
    - 제목: "Portfolio Value Over Time"
    - X축: 날짜/시간 라벨 (12월 15일 - 12월 22일)
    - Y축: 달러 값 ($10,000 - $12,000)
    - 라인: 작은 변동과 함께 상승 추세를 보이는 부드러운 곡선
    - 라인 색상: 네온 녹색 (#00FF88)
    - 채우기: 라인 색상에서 투명으로 그라데이션
    - 그리드: 은은한 회색 선
  - 행 3 (하단): 2개 차트 나란히
    - 왼쪽: 파이 차트 "Position Allocation"
      - BTC: 50% (주황색 #FF6B35)
      - ETH: 30% (파란색 #4A90D9)
      - SOL: 20% (보라색 #9B59B6)
      - 범례: 아래 또는 오른쪽
    - 오른쪽: 수평 바 차트 "Position Sizes (USDT)"
      - BTC: 5,000 USDT 바
      - ETH: 3,000 USDT 바
      - SOL: 2,000 USDT 바
      - 파이 차트와 일치하는 색상의 바
- 레이아웃:
  - 그리드: 3행
  - 행 높이: 1/6 (스탯), 1/2 (메인 차트), 1/3 (하단 차트)
  - 간격: 패널 사이 8-12px
  - 카드 스타일: 각 패널이 둥근 컨테이너 안에
- 스타일:
  - 테마: Grafana 다크 모드
  - 배경: #1A1A1A (전체), #2A2A2A (패널 카드)
  - 텍스트: 흰색 (#FFFFFF)
  - 패널 테두리: 은은한 #3A3A3A 또는 없음
  - 차트 라인: 메인 라인에 글로우 효과
  - 시간 범위 선택기: 우측 상단에 표시
- 참고: Grafana 공식 데모 대시보드, 암호화폐 트레이딩 대시보드 -->

---

## 알림 시스템: 문제를 놓치지 않는다

문제가 발생하면 즉시 알아야 합니다. AIFAP는 Discord를 통해 실시간 알림을 보내며, 크리티컬 에러와 트레이딩 활동 모두를 알려줍니다.

![Discord_Alert_System](../images/01_system_overview/07_discord_alert_system.png)
<!-- [IMAGE NEEDED]
- 파일명: 07_discord_alert_system
- 타입: Discord 메시지 목업 / 알림 흐름 다이어그램
- 목적: Discord 알림의 종류와 시각적 형태 보여주기
- 내용:
  - 왼쪽: "알림 유형" 수직 목록 (아이콘 포함)
    - 🚀 "시스템 시작" (파란색 뱃지)
    - 🟢 "포지션 오픈/추가" (녹색 뱃지)
    - 📝 "포지션 업데이트" (노란색 뱃지)
    - 🔴 "포지션 청산" (빨간색 뱃지)
    - 🚨 "크리티컬 에러" (진한 빨간색 뱃지)
  - 오른쪽: Discord 메시지 목업 (5개 예시 수직 배치)
    - 예시 1: 시스템 시작 (파란색 왼쪽 테두리)
      ```
      🚀 [Futures] System Started
      Time: 2025-12-30 09:07 +0900
      User: trader_01
      Strategy: momentum_v2
      Environment: live
      Trading Duration: 720h 0m
      ```
    - 예시 2: 포지션 오픈 (녹색 왼쪽 테두리)
      ```
      🟢 [Trader A] Position Added
           Sym  Side   Size       Entry      SL      TP FundingFee
      DOGEUSDT  long    133     0.12278 0.09824 0.12894
       LTCUSDT short    0.4    78.17000   93.91   74.35
       BTCUSDT  long 0.0004 87200.00000 69781.2 91587.8
      ```
    - 예시 3: 포지션 업데이트 (노란색 왼쪽 테두리)
      ```
      📝 [Trader A] Position Updated
           Sym  Side   Size   Entry      SL      TP              FundingFee
       XRPUSDT  long      3 1.85000  1.4801  1.9426  0.0001 -> 0.0001394625
      AVAXUSDT short    3.1 12.3670  14.836  11.745  0.0019 -> 0.0026564241
      ```
      참고: "->"는 같은 컬럼 내에서 값 변경 표시 (이전 값 -> 새 값)
    - 예시 4: 포지션 청산 (빨간색 왼쪽 테두리)
      ```
      🔴 [Trader A] Position Closed: LTCUSDT
      🔴 [Trader A] Position Closed: DOGEUSDT, BNBUSDT
      ```
    - 예시 5: 에러 알림 (진한 빨간색 왼쪽 테두리)
      ```
      🚨 [Futures] StackFuturesIndicatorResult Partial Failure (1)
      Time: 2026-01-05 03:33 +0900
      Module: StackFuturesIndicatorResult
      An issue occurred. Admin has been notified.
      ```
- 레이아웃:
  - 2열 레이아웃 (30% / 70%)
  - 왼쪽에 색상 뱃지가 있는 알림 유형 목록
  - 오른쪽에 유형과 매칭되는 색상 왼쪽 테두리가 있는 Discord 목업
  - 알림 유형에서 해당 예시로 연결되는 화살표
  - 포지션 테이블 데이터는 모노스페이스 폰트 (정렬된 컬럼)
- 스타일:
  - Discord 다크 테마 배경 (#36393F)
  - 메시지 배경: 약간 밝은 색상 (#40444B)
  - 왼쪽 테두리 색상: 파랑 (#5865F2) 시작, 초록 (#57F287) 오픈, 노랑 (#FEE75C) 업데이트, 빨강 (#ED4245) 청산, 진한 빨강 (#A12D2F) 에러
  - 텍스트: 제목은 흰색 (#FFFFFF), 본문은 회색 (#B9BBBE)
  - 폰트: Discord 스타일 산세리프, 테이블 데이터는 모노스페이스
  - 타임스탬프: 작은 회색 텍스트
  - 포지션 테이블: 고정 너비 컬럼, 숫자는 오른쪽 정렬
- 참고: Discord 임베드 메시지 스타일, 트레이딩 터미널 테이블 포맷 -->

### Discord (메인 채널)

Discord가 AIFAP의 주요 알림 허브입니다. 여러 종류의 실시간 알림을 처리합니다:

**시스템 시작** - 트레이딩 세션 시작 알림:
```
🚀 [Futures] System Started
Time: 2025-12-30 09:07 +0900
User: trader_01
Strategy: momentum_v2
Environment: live
Trading Duration: 720h 0m
```

**포지션 오픈** - 진입가, SL/TP 정보와 함께 새로운 포지션 알림:
```
🟢 [Trader A] New Positions Opened
    Sym  Side Size  Entry SL    TP FundingFee
DOTUSDT short   43  1.823    1.733
```

**포지션 추가** - 기존 포트폴리오에 추가된 포지션:
```
🟢 [Trader A] Position Added
     Sym  Side   Size       Entry      SL      TP FundingFee
DOGEUSDT  long    133     0.12278 0.09824 0.12894
 LTCUSDT short    0.4    78.17000   93.91   74.35
 BTCUSDT  long 0.0004 87200.00000 69781.2 91587.8
```

**포지션 업데이트** - 기존 포지션 변경 (SL/TP 조정, 펀딩비):
```
📝 [Trader A] Position Updated
     Sym  Side   Size   Entry      SL      TP              FundingFee
 XRPUSDT  long      3 1.85000  1.4801  1.9426  0.0001 -> 0.0001394625
AVAXUSDT short    3.1 12.3670  14.836  11.745  0.0019 -> 0.0026564241
 DOTUSDT short     43  1.8230   2.189   1.733  0.0052 -> 0.00779289
```
참고: "->"는 같은 컬럼 내에서 값 변경을 표시 (이전 값 -> 새 값)

**포지션 청산** - 청산된 포지션:
```
🔴 [Trader A] Position Closed: LTCUSDT
🔴 [Trader A] Position Closed: DOGEUSDT, BNBUSDT
```

**에러 알림** - 조치가 필요한 시스템 이슈:
```
🚨 [Futures] StackFuturesIndicatorResult Partial Failure (1)
Time: 2026-01-05 03:33 +0900
Module: StackFuturesIndicatorResult
An issue occurred. Admin has been notified.
```

### 에러 처리 철학

API 레벨에서 재시도 로직을 구현한 후, 버블업되는 모든 에러는 진정한 크리티컬입니다:

```python
try:
    # 내장 재시도가 있는 API 호출
    result = retry_request(place_order, max_retries=3)
except Exception as e:
    # 여기까지 오면 정말 심각한 문제
    send_discord_alert(user_id, "Critical", str(e), "auto_trade")
    sys.exit(1)  # 시스템 중지
```

---

## 관련 주제

이번 글에서는 전체 그림을 다뤘습니다. 향후 글에서 각 컴포넌트를 깊이 있게 다룰 예정입니다:

- **리밸런싱 엔진**: 비중 계산, 프로세스 동기화
- **데이터 파이프라인**: 시장 데이터 조회 및 변환
- **Finwatcher**: 실시간 모니터링 시스템 구축
- **쿠버네티스 배포**: 단일 Pod에서 CronJob으로
- **알림 시스템**: 멀티채널 알림
- **API 설계**: FastAPI 백엔드 아키텍처

---

## 핵심 요약

1. **관심사 분리**: 3개 프로세스가 전략, 실행, 모니터링을 독립적으로 처리
2. **이벤트 기반 동기화**: `multiprocessing.Event`가 프로세스 간 깔끔한 조정 제공
3. **클라우드 네이티브 설계**: Kubernetes + Istio가 인프라 복잡성 처리
4. **모든 것을 모니터링**: 분 단위 지표로 실시간 가시성 확보
5. **시끄럽게 실패**: 재시도 후 모든 에러는 크리티컬 - 알림 보내고 중지

트레이딩 시스템 구축은 트레이딩 로직만큼이나 인프라에 관한 것입니다. 초기에 내리는 아키텍처 결정이 시스템의 안정성, 유지보수성, 확장성을 결정합니다.

---

## 리소스

- **Kubernetes**: https://kubernetes.io/docs/
- **Istio**: https://istio.io/latest/docs/
- **FastAPI**: https://fastapi.tiangolo.com/
- **Bitget API**: https://www.bitget.com/api-doc/

---

*질문이나 피드백이 있으시면 연락 주세요.*

---

**저자 소개**
자동매매 시스템과 클라우드 인프라를 구축합니다. 복잡한 시스템을 안정적이고 유지보수 가능하게 만드는 것에 열정을 가지고 있습니다.

---

*새 글이 발행되면 알림을 받으려면 구독하세요.*
