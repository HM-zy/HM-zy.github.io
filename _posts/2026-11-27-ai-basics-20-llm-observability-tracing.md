---
title: "[AI 기초 20] LLM 관측성(Observability)과 트레이싱: 토큰 비용·응답 시간·실패 지점 추적"
date: 2026-11-27 08:00:00 +0900
categories: [기초·개념]
tags: [Observability, Tracing, OpenTelemetry, LLMOps, Metrics, Latency, Cost, Architecture, AI기초, AI]
---

*"사용자는 '어제는 되던 게 오늘은 답이 이상해요'라고 합니다. 그런데 우리는 그 요청이 어떤 문서를 검색했고, 어떤 모델로 갔고, 얼마를 썼는지 아무것도 모릅니다."*

AI 서비스를 운영하면서 가장 먼저 무너지는 것은 정확도가 아니라 **설명 가능성**입니다. 관측성(Observability)은 "지금 잘 되고 있나"에 답하는 계기판이고, 트레이싱(Tracing)은 "이 요청 하나가 어디서 시간을 썼나"에 답하는 블랙박스입니다.

---

## 🔭 세 가지 도구: 로그 / 메트릭 / 트레이스

| 도구 | 대답하는 질문 | AI 서비스에서의 예 |
| --- | --- | --- |
| **메트릭(Metrics)** | 얼마나 자주, 얼마나 느린가 | p95 지연 3.2초, 에러율 1.4%, 캐시 히트율 41% |
| **로그(Logs)** | 무슨 일이 있었나 | 요청 ID, 프롬프트 버전, 도구 인자, 에러 스택 |
| **트레이스(Traces)** | 어디서 시간·비용을 썼나 | 검색 120ms → 리랭킹 300ms → LLM 2.1초 → 가드레일 20ms |

- **일상 비유**: 메트릭은 자동차 **계기판**, 로그는 **정비 일지**, 트레이스는 **블랙박스 영상**. 셋 중 하나만 있으면 "엔진 경고등이 켜졌는데 왜 켜졌는지 모르는" 상태가 됩니다.
- **핵심 원리**: 하나의 요청을 **트레이스 ID 하나로 묶고**, 그 안의 단계들을 **Span(구간)** 으로 나눠 부모-자식으로 기록합니다. 각 Span에 토큰 수·비용·모델명을 속성으로 붙입니다.

---

## 🌳 트레이스는 이렇게 생겼다

```
● answer_request  (2,743ms, $0.014, tenant=인사팀)
  ├─ auth_check          12ms
  ├─ route              3ms   → tier=medium, reason=doc_question
  ├─ cache_lookup        4ms   → miss, best_sim=0.71
  ├─ retrieve           118ms  → docs=12, hybrid=bm25+vector
  │    ├─ bm25           42ms
  │    └─ vector         61ms
  ├─ rerank             296ms  → top_k=5
  ├─ llm_call         2,180ms  → model=middle, in=3,120 out=410, $0.0138
  │    └─ tool_call      84ms  → query_leave_balance → ok
  ├─ guardrail           26ms  → citations=4/4, pass
  └─ audit_write          8ms  → hash=9f3c...
```

이 그림 하나면 "느린 요청"이 **모델이 느린 것인지, 리랭커가 느린 것인지, 재시도가 붙은 것인지** 즉시 구분됩니다. 비용도 어느 단계에서 발생했는지가 그대로 보입니다.

---

## 📏 AI 서비스에서 반드시 보는 지표

| 지표 | 왜 중요한가 |
| --- | --- |
| **TTFT**(첫 토큰까지 시간) | 스트리밍 UX 체감 품질의 핵심 |
| **p50 / p95 지연** | 평균은 사용자 경험을 숨긴다. p95가 진짜 불만 |
| **요청당 토큰(in/out)** | 프롬프트 비대화·출력 폭주 조기 감지 |
| **요청당 비용 / 팀별 비용** | FinOps 예산 관리(18일차) |
| **도구 호출 실패율** | MCP 서버 장애·권한 오류(15·16일차) |
| **캐시 히트율** | 접두부가 흔들리는지, 임계값이 맞는지(19일차) |
| **검색 적중률·인용 커버리지** | RAG 품질의 실시간 대리 지표 |
| **가드레일 차단율 · HITL 승인율** | 안전장치가 실제로 작동하는지(17일차) |

표준화 팁: 속성 이름을 제각각 짓지 말고 **OpenTelemetry의 GenAI 시맨틱 컨벤션**(예: `gen_ai.request.model`, `gen_ai.usage.input_tokens`)을 따르면, 나중에 백엔드(Langfuse, LangSmith, Arize Phoenix, Datadog 등)를 바꿔도 계측 코드를 다시 쓰지 않습니다.

---

## 💻 5분 실습: 60줄 미니 트레이서 (중첩 Span + 비용 집계 + p95)

외부 라이브러리 없이, 위 그림과 같은 트레이스를 직접 만들어 출력합니다.

```python
import random, statistics, time
from contextlib import contextmanager
from types import SimpleNamespace

class Tracer:
    def __init__(self):
        self.stack, self.roots, self.records = [], [], []

    @contextmanager
    def span(self, name, **attrs):
        node = {"name": name, "attrs": attrs, "children": [],
                "start": time.perf_counter(), "ms": 0.0}
        (self.stack[-1]["children"] if self.stack else self.roots).append(node)
        self.stack.append(node)
        try:
            yield SimpleNamespace(attrs=node["attrs"])
        finally:
            self.stack.pop()
            node["ms"] = (time.perf_counter() - node["start"]) * 1000
            self.records.append(node)

    def render(self, nodes=None, depth=0):
        nodes = self.roots if nodes is None else nodes
        for n in nodes:
            mark = "●" if depth == 0 else "└─"
            attrs = " ".join(f"{k}={v}" for k, v in n["attrs"].items())
            print("  " * depth + f"{mark} {n['name']} {n['ms']:.0f}ms {attrs}")
            self.render(n["children"], depth + 1)

PRICE = {"middle": (2.50, 10.0)}   # $/1M tokens, 입력·출력 (가정)

def answer_request(t: Tracer, question, tenant):
    with t.span("answer_request", tenant=tenant) as root:
        with t.span("auth_check"): time.sleep(0.01)
        with t.span("route") as s: s.attrs["tier"] = "medium"
        with t.span("cache_lookup") as s: s.attrs["result"] = "miss"; s.attrs["sim"] = 0.71
        with t.span("retrieve") as s:
            s.attrs["docs"] = 12
            with t.span("bm25"): time.sleep(0.02)
            with t.span("vector"): time.sleep(0.03)
        with t.span("rerank") as s: time.sleep(0.05); s.attrs["top_k"] = 5
        with t.span("llm_call") as s:
            time.sleep(0.08)
            tin, tout = 3120, 410
            p_in, p_out = PRICE["middle"]
            s.attrs.update(model="middle", tokens_in=tin, tokens_out=tout,
                           cost=round(tin / 1_000_000 * p_in + tout / 1_000_000 * p_out, 4))
            with t.span("tool_call") as tc: tc.attrs["tool"] = "query_leave"; tc.attrs["ok"] = True
        with t.span("guardrail") as s: s.attrs["citations"] = "4/4"
        root.attrs["cost"] = next(r["attrs"]["cost"] for r in t.records if "cost" in r["attrs"])

t = Tracer()
for q, tenant in [("연차 규정 알려줘", "인사팀"), ("출장비 정산 절차", "재무팀"), ("휴가 신청 방법", "인사팀")]:
    answer_request(t, q, tenant)         # 요청마다 트레이스 1개 = 루트 Span 1개

print("🌳 트레이스 트리:"); t.render()

# 지연 분포와 비용 집계 (실제로는 수백 건이 쌓입니다)
latencies = [random.gauss(2700, 600) for _ in range(200)]
costs = [c["attrs"]["cost"] for c in t.records if "cost" in c["attrs"]]
costs = costs * 400

print(f"\n📊 p50 지연 {statistics.median(latencies):,.0f}ms")
print(f"📊 p95 지연 {sorted(latencies)[int(len(latencies) * 0.95)]:,.0f}ms")
print(f"💰 요청당 평균 비용 ${statistics.mean(costs):.4f} → 일 1,000건 ${statistics.mean(costs)*1000:,.2f}")
```

**여기서 느껴야 하는 것**: 트레이스가 없으면 "느리다"는 사실만 알고, 있으면 **`llm_call`이 80ms인데 `rerank`가 50ms, 전체가 170ms**처럼 구조가 보입니다. 관측성의 가치는 대시보드 예쁨이 아니라 **다음에 무엇을 고칠지 결정하게 해주는 것**입니다.

---

## ⚠️ 로그에 남기면 안 되는 것 / 반드시 남겨야 하는 것

| 남기지 말 것 | 남길 것 |
| --- | --- |
| 원문 개인정보(주민번호·연락처·계좌) | 요청 ID, 트레이스 ID |
| API 키·DB 접속 정보 | 모델명, 프롬프트 **버전** |
| 사용자가 입력한 전체 대화 원문(무기한) | 마스킹된 도구 인자·결과 요약 |
| | 승인자, 라우팅 근거, 캐시 히트 여부 |

프롬프트 원문을 저장해야 디버깅이 되지만, 그대로 저장하면 개인정보 저장소가 하나 더 생기는 셈입니다. **보존 기간 + 마스킹 + 접근 권한**을 함께 설계하세요(16일차).

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"관측성 없는 AI는 데모는 되지만 운영은 안 된다. SA가 제안서에 넣어야 할 문장은 '품질·비용·지연을 요청 단위로 추적할 수 있습니다'이며, 이것이 있어야 파일럿을 프로덕션으로 넘길 수 있다."**
