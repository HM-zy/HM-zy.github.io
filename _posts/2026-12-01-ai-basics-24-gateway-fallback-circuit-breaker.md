---
title: "[AI 기초 24] LLM 게이트웨이와 장애 대응: 폴백(Fallback) 체인과 서킷 브레이커 설계"
date: 2026-12-01 08:00:00 +0900
categories: [기초·개념]
tags: [Gateway, Fallback, CircuitBreaker, Resilience, Timeout, SRE, Architecture, AI기초, AI]
image:
  path: /assets/img/posts/llm-gateway-circuit-breaker.png
  alt: 게이트웨이가 장애 난 모델 제공사를 피해 폴백 경로로 트래픽을 돌리고 서킷 브레이커 상태를 관리하는 구조도
---

*"새벽 3시에 모델 제공사 API가 5xx를 뱉기 시작했습니다. 우리 서비스는 아침 6시까지 '잠시 후 다시 시도해 주세요'만 돌려줬고, 고객사 슬랙에는 200줄의 장애 로그가 쌓였습니다."*

![게이트웨이가 장애 난 제공사를 피해 폴백 경로로 트래픽을 돌리고, 차단기 상태를 관리한다](/assets/img/posts/llm-gateway-circuit-breaker.png)
_게이트웨이가 장애 난 제공사를 피해 폴백 경로로 트래픽을 돌리고, 차단기 상태를 관리한다_

LLM은 **우리가 통제할 수 없는 외부 의존성**입니다. 같은 회선, 같은 리전, 같은 모델이라도 어제는 0.9초에 답하던 것이 오늘은 30초 동안 응답이 없습니다. 오늘은 그 상황에서 **서비스가 같이 죽지 않게 만드는 구조** — 폴백 체인과 서킷 브레이커를 다룹니다.

> 18일차의 **스마트 라우팅**은 "싼 길로 보내기(비용)"였고, 오늘은 **"막힌 길을 피해가기(가용성)"** 입니다. 같은 게이트웨이 위에 얹히는 서로 다른 두 정책입니다.

---

## 🔌 일상 비유: 두꺼비집과 예비 창구

집에 합선이 나면 **두꺼비집(차단기)이 먼저 내려갑니다.** 그 순간 그 방의 불은 꺼지지만, 집 전체가 불타는 것은 막습니다. 그리고 사람이 원인을 고친 뒤 **레버를 한 번 올려** 확인합니다. 잘 되면 다시 켜 두고, 또 내려가면 다시 내려 둡니다.

LLM 게이트웨이도 똑같습니다.

- **두꺼비집 없음** = 실패하는 API에 계속 요청을 보내며, 매 요청이 타임아웃(2초)을 다 기다립니다. 사용자는 3초 넘게 멈춘 화면을 보고, 서버는 죽은 상대에게 계속 전화를 겁니다.
- **두꺼비집 있음** = 5번 연속 실패하면 **그 제공사를 잠시 차단(OPEN)** 하고, 20초 뒤에 **딱 1건만 찔러 봅니다(HALF-OPEN)**. 성공하면 복구, 실패하면 다시 차단.

그리고 **예비 창구(폴백 체인)** 가 옆에 있습니다. 1번 창구가 닫혀 있으면 2번 창구로, 2번도 안 되면 3번으로. 단, 창구마다 "기다리는 시간"이 정해져 있어야 합니다 — 안 그러면 예비 창구를 찾다가 사용자가 먼저 떠납니다.

---

## 🏛️ 게이트웨이가 하는 일 (LLM 앞단의 교통정리)

모든 LLM 호출을 **하나의 엔드포인트**로 모으면, 애플리케이션 코드를 고치지 않고 정책을 바꿀 수 있습니다.

| 기능 | 없을 때 생기는 일 |
|---|---|
| 단일 엔드포인트 | 서비스마다 모델 SDK·키·재시도 로직이 흩어져 장애 대응이 불가능 |
| 키 관리·쿼터 | 키가 클라이언트에 노출되고, 한 팀이 전체 한도를 소진 |
| 타임아웃 예산 | 한 요청이 30초씩 붙잡혀 스레드 풀이 고갈 |
| 폴백 체인 | 특정 제공사 장애 = 서비스 전면 장애 |
| 서킷 브레이커 | 죽은 제공사에 계속 과금·지연 |
| 요청 로그·트레이스 | "어느 모델이 왜 실패했나"를 사후에 알 수 없음 (20일차 관측성) |

**한 줄 원칙**: 애플리케이션은 "LLM을 부른다"만 알고, **"어느 제공사로 어떻게 갈지"는 게이트웨이가 안다.**

---

## 🧯 오류를 구분해야 폴백이 산다

모든 실패에 재시도하면 **비용만 두 배**가 됩니다.

| 오류 | 의미 | 올바른 대응 |
|---|---|---|
| **429** Too Many Requests | 한도 초과 | 짧은 백오프 후 재시도 / 다른 제공사로 폴백 |
| **5xx** 서버 오류 | 제공사 장애 | 폴백 + 서킷 브레이커 |
| **타임아웃** | 응답 없음 | 폴백 (같은 제공사 재시도는 대개 무의미) |
| **400** 잘못된 요청 | 우리 프롬프트/스키마 문제 | 재시도 금지 — 즉시 실패·알림 |
| **401/403** 인증 | 키·권한 문제 | 재시도 금지 — 즉시 알림 |

여기서 SA가 자주 놓치는 것: **400을 재시도하면 400이 세 번 더 쌓이고, 그 사이 사용자는 6초를 기다립니다.** 실패에도 "빨리 실패하기(fail fast)"가 필요합니다.

**타임아웃 예산**은 전체 응답 목표에서 거꾸로 계산합니다. 예: 사용자 체감 목표 8초 → 1차 모델 3초, 2차 폴백 3초, 저장·렌더 2초. 각 구간의 합이 목표를 넘으면, 폴백을 아무리 잘 짜도 SLA는 못 지킵니다.

---

## ⚡ 서킷 브레이커의 3가지 상태

```text
        연속 실패 N회
CLOSED ───────────────→ OPEN
  ↑                       │  대기(cooldown) 경과
  │   성공                 ↓
  └────────────────── HALF-OPEN
        ↑  실패 → 즉시 OPEN
```

- **CLOSED**: 정상. 모든 요청을 통과시키고 실패 횟수를 셉니다.
- **OPEN**: 통과 금지. 요청은 **즉시** 다음 후보로 넘어갑니다(기다리지 않음). 이 "즉시"가 핵심입니다 — 차단기의 가치는 지연을 **0으로 만드는 것**입니다.
- **HALF-OPEN**: 쿨다운이 지나면 **소량(보통 1건)** 만 통과시켜 상태를 확인합니다. 성공하면 CLOSED, 실패하면 다시 OPEN.

임계값은 감이 아니라 데이터로 정합니다. **연속 5회 / 20초**는 시작값이고, 트래픽이 많으면 비율 기준(1분간 실패율 50%)이 더 안정적입니다.

---

## 🪂 실패했을 때 무엇을 보여줄 것인가 (Degraded Mode)

차단기가 열렸을 때 사용자에게 **빈 화면**을 보여주면, 아키텍처가 아무리 좋아도 실패입니다. 폴백 사다리를 미리 정해 두세요.

1. **다른 모델로 폴백** (같은 벤더의 경량 모델 → 다른 벤더)
2. **캐시된 유사 답변** (19일차 시맨틱 캐시, 단 "실시간 정보 아님" 표시)
3. **검색 결과만 제시** (요약 없이 근거 문서 목록)
4. **사람 연결** (상담원·티켓 생성)
5. **정직한 안내** ("지금 답변 품질이 낮습니다" — 환각으로 채우는 것보다 낫습니다)

> "장애 시 무엇을 보여줄 것인가"는 기술이 아니라 **제품 결정**입니다. 그래서 SA가 고객과 먼저 합의해야 하는 항목입니다.

---

## 💻 5분 실습: 200건 요청에 60초 장애를 넣어보자

기본 제공사(primary)가 **100~159번째 요청 구간에서 죽는** 상황을 만들고, 브레이커 유무에 따라 **총 대기시간·p95·SLA 초과 건수**를 비교합니다. 브레이커는 CLOSED/OPEN/HALF-OPEN 상태를 실제로 전이합니다.

```python
TIMEOUT = 2.0                       # 응답 없으면 이만큼 기다린 뒤 실패(초)
LATENCY = {"primary": 0.9, "secondary": 1.4, "cheap": 2.2}
CHAIN = ["primary", "secondary", "cheap"]   # 폴백 순서
OUTAGE = range(100, 160)            # 100~159번 요청 구간에 primary 장애
SLA = 3.0                           # 사용자 체감 목표(초)


class Breaker:
    """제공사 1곳당 하나씩 두는 상태 기계."""

    def __init__(self, threshold=5, cooldown=20):
        self.threshold = threshold      # 연속 실패 몇 번이면 열까
        self.cooldown = cooldown        # 몇 요청 뒤에 찔러볼까
        self.state = "CLOSED"
        self.fails = 0
        self.opened_at = None
        self.events = []

    def allow(self, now):
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            if now - self.opened_at >= self.cooldown:
                self._set("HALF-OPEN", now)
                return True             # 탐침 1건 허용
            return False                # 그 외에는 즉시 다음 후보로
        return True                     # HALF-OPEN

    def record(self, ok, now):
        if ok:
            self.fails = 0
            self._set("CLOSED", now)
            return
        self.fails += 1
        if self.state == "HALF-OPEN" or self.fails >= self.threshold:
            self._set("OPEN", now)

    def _set(self, state, now):
        if state != self.state:
            self.events.append((now, state))
            self.state = state
            if state == "OPEN":
                self.opened_at = now


def route(i, breaker=None):
    """한 요청을 폴백 체인에 태운다. 반환값: (총 대기 초, 실제 응답한 제공사)"""
    waited = 0.0
    for provider in CHAIN:
        if provider == "primary" and breaker is not None and not breaker.allow(i):
            continue                       # 열린 차단기 = 기다리지 않고 건너뜀
        healthy = (i not in OUTAGE) if provider == "primary" else True
        waited += LATENCY[provider] if healthy else TIMEOUT
        if provider == "primary" and breaker is not None:
            breaker.record(healthy, i)
        if healthy:
            return waited, provider
    return waited, None                    # 전부 실패 → 저하 모드


def run(use_breaker, n=200):
    breaker = Breaker() if use_breaker else None
    waits, used = [], {}
    for i in range(n):
        waited, provider = route(i, breaker)
        waits.append(waited)
        used[provider] = used.get(provider, 0) + 1
    ordered = sorted(waits)
    return {
        "total": sum(waits),
        "avg": sum(waits) / len(waits),
        "p95": ordered[int(len(waits) * 0.95) - 1],
        "over_sla": sum(1 for w in waits if w > SLA),
        "failed": used.get(None, 0),
        "used": used,
        "events": breaker.events if breaker else [],
    }


for name, use in [("브레이커 없음", False), ("브레이커 있음", True)]:
    r = run(use)
    print(f"[{name}]")
    print(f"  총 대기 {r['total']:.0f}초 / 평균 {r['avg']:.2f}초 / p95 {r['p95']:.2f}초")
    print(f"  SLA({SLA:.0f}초) 초과 {r['over_sla']}건 / 최종 실패 {r['failed']}건")
    print(f"  응답 경로: {r['used']}")
    if r["events"]:
        print(f"  상태 전이: {r['events']}")

r0, r1 = run(False), run(True)
print()
print(f"→ 절약된 대기시간 {r0['total'] - r1['total']:.0f}초, "
      f"SLA 초과 {r0['over_sla']}건 → {r1['over_sla']}건")
```

**실행 결과**

```text
[브레이커 없음]
  총 대기 330초 / 평균 1.65초 / p95 3.40초
  SLA(3초) 초과 60건 / 최종 실패 0건
  응답 경로: {'primary': 140, 'secondary': 60}
[브레이커 있음]
  총 대기 226초 / 평균 1.13초 / p95 1.40초
  SLA(3초) 초과 7건 / 최종 실패 0건
  응답 경로: {'primary': 136, 'secondary': 64}
  상태 전이: [(104, 'OPEN'), (124, 'HALF-OPEN'), (124, 'OPEN'), (144, 'HALF-OPEN'), (144, 'OPEN'), (164, 'HALF-OPEN'), (164, 'CLOSED')]

→ 절약된 대기시간 104초, SLA 초과 60건 → 7건
```

**체감 포인트**: 두 시나리오 모두 **최종 실패는 0건**입니다. 폴백이 있으니 사용자는 답을 받습니다. 차이는 **어떻게 받았는가**입니다.

- 브레이커가 없으면 장애 60건 동안 매번 2초를 버리고 3.4초 만에 답합니다 → **SLA 초과 60건**.
- 브레이커가 있으면 104번째 요청에서 문을 닫고, 그 뒤로는 **0초 만에 secondary로** 넘어갑니다 → p95가 3.40초에서 **1.40초**로 내려갑니다.

상태 전이 로그를 보세요. `(124, 'HALF-OPEN')` 직후 `(124, 'OPEN')` — 20초 뒤 찔러봤더니 아직 죽어 있어서 바로 다시 닫힌 것입니다. 164번째에서야 `CLOSED`로 복구됩니다. **이 로그가 곧 운영 대시보드**입니다. "지금 어느 제공사가 열려 있나"를 보여주지 않으면, 장애 때 아무도 원인을 모릅니다.

---

## ⚠️ 자주 하는 실수 3가지

1. **폴백 체인에 같은 리전을 넣음**: primary(버지니아) → secondary(버지니아) → cheap(버지니아)는 폴백이 아니라 **같은 재해를 세 번 맞는 것**입니다. 최소 1개는 **다른 제공사·다른 리전**이어야 합니다.
2. **타임아웃 없이 재시도**: 타임아웃이 없으면 실패를 알 수 없고, 재시도는 스레드를 더 붙잡습니다. **재시도 횟수 × 타임아웃 ≤ 전체 예산**을 코드가 아니라 설정으로 강제하세요.
3. **차단기를 수동 해제만 두기**: 장애가 풀렸는데 사람이 레버를 안 올려서 **몇 시간을 secondary로 버티는** 사고가 납니다. HALF-OPEN 자동 탐침이 반드시 있어야 합니다.

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"LLM 아키텍처의 가용성은 '몇 %인가'가 아니라 '제공사 하나가 죽었을 때 몇 초 만에 무엇으로 우회하는가'로 증명된다."**
>
> 고객 보안·인프라팀과의 PoV에서 이 주제는 반드시 나옵니다. 후속 질문 셋: ① 특정 벤더 장애 시 우리 서비스는 어떤 화면을 보여주는가 ② 폴백 모델의 품질 저하는 어떻게 공지하는가 ③ 차단기 상태와 폴백 발생률은 어디서 모니터링하는가. 이 셋에 답하려면 **폴백 체인·타임아웃 예산·저하 모드 안내 문구**가 설계 단계에 있어야 합니다. 이건 데모에서 절대 드러나지 않고, 계약 후 첫 장애에서 드러납니다.
