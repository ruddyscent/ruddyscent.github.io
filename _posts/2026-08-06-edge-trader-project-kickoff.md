---
layout: post
title: "EdgeTrader - 강화학습 기반 암호화폐 자동매매 프로젝트를 시작하며"
subtitle: 모델보다 데이터와 비용 모델을 먼저 세우기로 한 이유
tags: [reinforcement-learning, trading, cryptocurrency, korbit, jetson, ppo, timescaledb, python]
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/edge-trader.webp
share-img: /assets/img/develop.jpeg
author: 전경원
description: 코빗 시장 데이터를 수집해 강화학습 정책을 훈련하고 NVIDIA Jetson에서 추론하는 암호화폐 자동매매 연구 플랫폼 EdgeTrader의 설계 기록.
---

강화학습으로 RC 비행기를 조종하는 [HoverPilot](/tags/#hoverpilot) 작업을 하면서, 시뮬레이터 대신 실제 시장 데이터를 상대로 같은 접근을 시도해보고 싶다는 생각이 들었습니다. 그 결과가 **EdgeTrader**입니다. 코빗의 암호화폐 시장 데이터를 실시간으로 모으고, 강화학습 정책을 훈련해 매매 결정을 내리고, 최종적으로 NVIDIA Jetson 위에서 추론과 주문 실행을 돌리는 시스템을 목표로 합니다.

## 왜 이 프로젝트를 시작하는가

자동매매에서 강화학습이라고 하면 보통 정책 네트워크 구조나 알고리즘 선택을 먼저 떠올립니다. 그런데 실제로 시스템을 굴려보면, 수익률을 갉아먹는 건 모델의 정교함이 아니라 그 아래에 깔린 것들입니다. 데이터가 중간에 빠지거나 중복되고, 거래소 시각과 로컬 시각이 어긋나고, 재연결 후 오더북 상태가 깨지고, 수수료와 스프레드와 슬리피지가 백테스트에는 없다가 실거래에서만 튀어나옵니다.

그래서 EdgeTrader의 개발 순서를 이렇게 잡았습니다.

```text
시장 데이터 수집
→ 데이터 검증
→ 백테스트 엔진
→ 기준 전략
→ 강화학습 환경
→ 모의 거래
→ 소액 실거래
```

강화학습 환경은 이 목록의 중간에 있습니다. 데이터 파이프라인과 비용 모델, 백테스트가 먼저 정확해진 다음에야 그 위에서 정책 하나를 실험합니다. 이 순서를 지키는 것이 이 프로젝트에서 내가 가장 신경 쓰는 부분입니다.

## 왜 코빗인가

거래소는 코빗 하나로 시작합니다. 원화 마켓을 지원하고, 공개 WebSocket으로 실시간 시세와 오더북을 받을 수 있어서 개인이 데이터 수집기를 붙이기에 부담이 적습니다. 여러 거래소를 동시에 지원하면 초기부터 각 거래소의 응답 형식과 인증 방식 차이에 발목이 잡히기 쉬운데, 지금은 그럴 단계가 아닙니다.

대신 나중에 다른 거래소로 넓힐 여지는 코드 구조로 남겨둡니다. 거래소 연동을 인터페이스로 분리해서, 코빗 구현을 나중에 교체하거나 추가할 수 있게 합니다.

```python
class ExchangeClient(Protocol):
    async def connect_market_stream(self) -> None: ...
    async def get_balances(self) -> list[Balance]: ...
    async def place_order(self, request: OrderRequest) -> Order: ...
    async def cancel_order(self, order_id: str) -> None: ...
    async def get_open_orders(self) -> list[Order]: ...
```

## 거래 대상: BTC부터, ETH와 SOL은 나중에

후보 종목은 BTC/KRW, ETH/KRW, SOL/KRW 세 가집니다. 하지만 처음부터 셋을 하나의 포트폴리오 정책으로 묶어 학습하지는 않습니다. 단계를 나눕니다.

Phase 1은 BTC/KRW 하나로 데이터 수집기와 백테스트 엔진을 검증하는 데 씁니다. Phase 2에서 ETH/KRW를 추가해 같은 정책이 다른 종목에서도 일반화되는지 확인하고, Phase 3에서 변동성이 더 큰 SOL/KRW로 정책을 시험합니다. 종목을 늘리는 것 자체가 검증 항목이지, 처음부터 다 넣고 시작할 문제가 아닙니다.

숏 포지션과 레버리지는 쓰지 않습니다. 현물 거래만 대상으로 합니다. 초기 단계에서 위험을 늘릴 이유가 없습니다.

## 왜 1분 데이터를 모으면서 5분마다 결정하는가

시장 데이터는 1분 단위로 수집하지만, 매분 거래하지는 않습니다. 관찰 주기와 거래 주기를 분리했습니다.

```yaml
market_data_interval: 1m
decision_interval: 5m
observation_window: 120m
minimum_position_duration: 5m
```

동작은 이렇습니다. 매분 시장 데이터를 모으고, 5분마다 정책이 목표 포지션을 정하고, 실제 주문 여부는 위험 관리 계층이 최종적으로 판단합니다.

매분 거래하게 두면 수수료와 스프레드, 슬리피지가 매 거래마다 붙어서 과매매로 흐르기 쉽습니다. 1분 데이터를 모으는 건 관찰의 해상도를 높이기 위해서지, 그 빈도로 매매하기 위해서가 아닙니다. 두 주기를 분리하면 "자주 보되 드물게 움직인다"는 설계가 자연스럽게 나옵니다.

정책도 주문 수량과 가격을 직접 정하지 않습니다. 에이전트는 목표 포지션 비율만 결정합니다.

```text
0: 현금 100%
1: 자산 50%
2: 자산 100%
```

현재 잔고와 미체결 주문을 확인해 목표에 맞는 주문을 계산하는 일은 주문 실행 엔진의 몫입니다. 정책과 실행을 떼어놓아야 나중에 실행 로직을 바꿔도 정책을 다시 학습하지 않아도 됩니다.

## PostgreSQL과 TimescaleDB

데이터베이스는 PostgreSQL에 TimescaleDB 확장을 얹기로 했습니다. 이 데이터는 결국 시계열이라, 특정 구간을 범위로 조회하고 집계하는 일이 잦습니다. TimescaleDB의 hypertable과 continuous aggregate가 여기에 잘 맞습니다. 복잡한 분석 SQL과 window function을 쓰기에도 편하고, 원시 메시지는 `JSONB`로 남겨둘 수 있으며, 주문·체결·잔고처럼 트랜잭션 무결성이 중요한 데이터도 한 엔진 안에서 다룰 수 있습니다. Python 쪽에서 SQLAlchemy, Pandas, Polars와 붙이기 좋고 나중에 Grafana를 연결하기도 쉽습니다.

MongoDB도 후보로 올려두고 고민했습니다. 오더북 스냅샷처럼 단계 수가 유동적인 문서를 그대로 넣거나 원시 WebSocket 메시지를 스키마 없이 쌓기에는 문서 지향 모델이 편하고, 초기 개발 속도도 빠릅니다. 스키마가 아직 자주 바뀌는 단계라면 매력적인 선택입니다. 다만 이 프로젝트에서 정작 자주 하게 될 일은 특정 구간을 시계열로 범위 조회하고 집계하는 쪽이고, 주문·체결·잔고는 트랜잭션 무결성이 필요합니다. 이 두 요구를 한 엔진에서 깔끔하게 처리하는 데는 PostgreSQL과 TimescaleDB가 더 맞았습니다. 유연한 문서 저장이 필요한 원시 메시지는 `JSONB`로 흡수할 수 있어서, MongoDB를 따로 두면서 얻는 이점이 크지 않다고 봤습니다. 어느 한쪽이 일방적으로 낫다는 판단은 아니고, 이 프로젝트의 조회 패턴에 맞춘 선택입니다.

1분 캔들 테이블은 대략 이런 모양으로 시작합니다.

```sql
CREATE TABLE market_candles (
    time        TIMESTAMPTZ NOT NULL,
    symbol      TEXT NOT NULL,
    interval    TEXT NOT NULL,
    open        NUMERIC NOT NULL,
    high        NUMERIC NOT NULL,
    low         NUMERIC NOT NULL,
    close       NUMERIC NOT NULL,
    volume      NUMERIC NOT NULL,
    trade_count INTEGER,
    PRIMARY KEY (symbol, interval, time)
);
```

## 코빗 오더북을 1분 특징으로 집계하기

오더북은 코빗 WebSocket으로 실시간 이벤트를 받습니다. 코빗이 완성된 "1분 오더북"을 그대로 내려준다고 가정하지 않습니다. 그런 데이터는 직접 만들어야 합니다.

수집 구조는 이렇습니다.

```text
Korbit WebSocket
    ↓
실시간 오더북 이벤트 수신
    ↓
메모리에서 현재 오더북 상태 유지
    ↓
1분 동안 특징 계산
    ↓
매분 마지막 상태와 집계 특징 저장
```

메모리에 상위 10단계 호가를 유지하면서 1분 동안 스프레드, 호가 변경 횟수, 잔량 같은 값을 누적하고, 매분 경계에서 마지막 스냅샷과 집계 특징을 함께 저장합니다. 저장하는 집계 특징에는 평균·최대·최소 스프레드, 최우선 호가 변경 횟수, 상위 5단계와 10단계의 잔량, 그리고 오더북 불균형(imbalance)이 들어갑니다. 불균형은 이렇게 정의합니다.

```python
imbalance = (bid_volume - ask_volume) / (bid_volume + ask_volume)
```

분모가 0이면 0이나 결측값으로 처리합니다.

원시 WebSocket 이벤트를 처음부터 전부 영구 저장하면 용량이 금방 감당하기 어려워집니다. 그래서 보존 정책을 계층별로 나눴습니다.

```yaml
retention:
  raw_websocket_events: 7d
  orderbook_snapshots_1m: 1y
  orderbook_features_1m: permanent
  candles_1m: permanent
  orders_and_executions: permanent
```

원시 이벤트는 디버깅용으로 짧게만 두고, 분석에 계속 쓰는 1분 특징과 캔들은 영구 보존합니다.

한 가지 더, 시각은 여러 개를 나눠서 기록합니다.

```text
exchange_timestamp
received_timestamp
processed_timestamp
bucket_timestamp
```

거래소 시각과 수신 시각을 따로 남겨야 네트워크 지연, 처리 지연, 데이터 누락, 그리고 "오래된 데이터로 실행된 추론"을 나중에 추적할 수 있습니다. 이 구분을 빼먹으면 문제가 생겼을 때 원인을 되짚을 방법이 없어집니다.

## 강화학습 환경: 상태, 행동, 보상

강화학습 알고리즘은 PPO로 시작합니다. Stable-Baselines3 구현이 안정적이고 이산 행동을 붙이기 쉬우며, 기준선을 세우고 디버깅하기에 무난합니다. Recurrent PPO나 SAC 같은 후보는 첫 버전이 돌아간 다음의 이야기입니다.

상태는 최근 120분의 1분 데이터를 관찰 창으로 씁니다. 수익률, 거래량 특징, 오더북 특징에 더해 현재 포지션, 현금 비율, 미실현 손익, 직전 행동, 최근 거래 후 경과 시간을 넣습니다. 절대 가격 대신 정규화된 값을 쓰는 게 핵심입니다. 예를 들어 가격은 로그 수익률로 바꿉니다.

```python
log_return = np.log(close_t / close_t_minus_1)
```

행동은 앞서 말한 세 가지 이산 선택(현금 100%, 자산 50%, 자산 100%)으로 시작합니다.

보상이 이 설계에서 제일 조심스러운 부분입니다. 단순 수익률만 보상으로 주면 에이전트는 비용을 무시한 채 자주 거래하는 쪽으로 학습하기 쉽습니다. 그래서 비용과 위험을 보상 안에 명시적으로 넣습니다.

```python
reward = (
    portfolio_log_return
    - transaction_cost
    - slippage_cost
    - turnover_penalty
    - drawdown_penalty
)
```

수수료, bid-ask 스프레드, 슬리피지, 포지션 변경 비용, 최대 낙폭, 잦은 거래 페널티를 모두 반영합니다. 보상 함수는 별도 모듈로 떼어내서 각 항의 비중을 실험으로 조정할 수 있게 둡니다. drawdown penalty를 얼마나 세게 줄지 같은 값은 지금 확정하지 않고 열어둔 상태입니다.

## Jetson의 역할

배포 대상은 NVIDIA Jetson Orin 계열입니다. 다만 훈련까지 Jetson에서 하지는 않습니다. 역할을 나눕니다.

데스크톱이나 클라우드는 무거운 일을 맡습니다. 데이터 전처리, 강화학습 훈련, 하이퍼파라미터 탐색, 대규모 백테스트, 모델 평가가 여기 속합니다. Jetson은 실시간에 붙는 일을 맡습니다. 데이터 수집, 특징 계산, 모델 추론, 주문 실행, 위험 관리, 모니터링입니다.

PPO의 MLP 정책은 추론 비용이 작을 가능성이 높아서, 초기에는 PyTorch 모델을 그대로 올려도 됩니다. TensorRT 변환은 필요해지면 그때 합니다. 저장장치는 데이터베이스 쓰기가 계속 발생하는 걸 감안해 microSD보다 NVMe SSD를 쓸 생각입니다. 프로젝트 이름의 "Edge"가 여기서 두 번째 의미를 갖습니다. 시장에서 확보하려는 통계적 우위(trading edge)이자, 엣지 디바이스에서 도는 시스템이라는 뜻입니다.

## 위험 관리가 모델보다 앞에 있는 이유

가장 강조하고 싶은 설계 결정입니다. 실거래 안전장치는 강화학습 바깥에 둡니다. 정책이 무엇을 출력하든, `RiskManager`가 그 위에서 최종 결정을 합니다.

```python
decision = agent.predict(observation)

validated_order = risk_manager.validate(
    decision=decision,
    portfolio=portfolio,
    market=market_state,
)

if validated_order is not None:
    execution_engine.submit(validated_order)
```

최대 주문 금액, 최대 자산 비중, 일일 손실 한도, 최대 낙폭 같은 규칙은 정책이 바꿀 수 없습니다. WebSocket 데이터가 오래됐거나, 스프레드가 기준을 넘거나, 잔고 조회에 실패하거나, 거래소 시각과 시스템 시각 차이가 크거나, 긴급 정지 플래그가 켜진 상황에서는 아예 주문을 내지 않습니다.

```yaml
risk:
  max_position_ratio: 0.25
  max_order_ratio: 0.10
  daily_loss_limit: 0.03
  max_drawdown: 0.10
  stale_market_data_seconds: 10
  order_cooldown_minutes: 5
  max_open_orders: 1
```

학습된 정책은 평가 데이터에서 좋아 보여도 실시장의 예외 상황을 다 겪어본 게 아닙니다. 안전장치를 모델 안에 넣으면 학습 과정에서 흐려질 수 있으니, 절대 협상 불가능한 규칙은 바깥에 고정해 둡니다.

## 단계별 개발 계획

전체를 한 번에 만들지 않습니다. Phase 0에서 저장소를 초기화하고 Docker Compose로 PostgreSQL과 TimescaleDB를 띄웁니다. Phase 1은 코빗 데이터 수집기입니다. WebSocket 자동 재연결과 exponential backoff, stale connection 감지를 갖추고 BTC/KRW의 ticker·trades·orderbook을 모아 1분 특징까지 저장합니다. 이 단계의 완료 조건은 명확합니다.

```text
24시간 이상 중단 없이 데이터를 수집한다.
재연결 후 오더북 상태가 정상적으로 복구된다.
1분 캔들과 오더북 특징이 중복 없이 저장된다.
```

그다음 데이터 검증(Phase 2), 백테스트 엔진(Phase 3), Gymnasium 환경(Phase 4), PPO 기준 모델(Phase 5), 모의 거래(Phase 6), Jetson 배포(Phase 7), 소액 실거래(Phase 8), 확장(Phase 9)으로 이어집니다. 실거래는 마지막이고, 그마저도 출금 권한 없는 API 키로 아주 작은 포지션부터 시작합니다.

저장소 초기화와 데이터 수집기 구현은 Codex에 맡길 생각입니다. 이 글은 그 작업에 앞서 설계 판단을 정리해 두는 출발점입니다.

## 이름과 라이선스

이름은 EdgeTrader, 저장소는 `edge-trader`, 패키지는 `edge_trader`입니다. 라이선스는 Apache License 2.0으로 정했습니다. 상업적 이용과 수정을 허용하면서 명시적인 특허 허여 조항이 있어, MIT보다 특허 관련 보호가 분명합니다. 자동매매 엔진과 AI 알고리즘을 다루는 프로젝트에는 이쪽이 더 맞는다고 봤습니다.

## 마무리

EdgeTrader의 첫 성공 기준은 높은 수익률이 아닙니다. 이렇게 잡았습니다.

> 시장 데이터를 안정적으로 수집하고, 동일한 데이터와 설정으로 재현 가능한 백테스트를 수행하며, 위험 관리 규칙 아래에서 모의 거래를 장기간 중단 없이 실행하는 것.

강화학습은 이 기반 위에서 실험하는 하나의 정책일 뿐입니다. 데이터 누락과 중복을 막고, 시각을 정확히 기록하고, 오더북 상태를 복구하고, 수수료와 스프레드와 슬리피지를 제대로 반영하는 일이 먼저입니다. 이 순서가 흔들리면 아무리 좋은 모델을 얹어도 결과를 믿을 수 없습니다.

다음 글에서는 코빗 WebSocket 수집기를 실제로 붙이면서 마주친 문제들을 기록할 생각입니다. 이 프로젝트가 어디까지 갈지는 아직 모르지만, 적어도 기반부터 쌓는다는 원칙만큼은 지켜보려 합니다.

> **면책**: 이 프로젝트는 교육과 연구 목적으로 공개합니다. 투자 조언이 아니며, 암호화폐 거래에는 상당한 손실 위험이 따릅니다. 실제 자금으로 사용하기 전에 그 위험을 충분히 이해하고 받아들여야 합니다.
