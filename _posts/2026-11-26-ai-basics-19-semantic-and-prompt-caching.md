---
title: "[AI 기초 19] 시맨틱 캐싱 & 프롬프트 캐싱: API 비용과 지연시간을 동시에 줄이는 두 가지 전략"
date: 2026-11-26 08:00:00 +0900
categories: [기초·개념]
tags: [SemanticCache, PromptCaching, Caching, CostOptimization, Latency, Redis, FinOps, Architecture, AI기초, AI]
---

*"같은 질문이 하루에 400번 들어옵니다. '연차 며칠 남았어?'인데, 표현만 조금씩 다를 뿐입니다. 그때마다 모델을 호출하고 있으니 비용이 줄 리가 없죠."*

라우팅(18일차)이 **더 싼 모델을 고르는 일**이라면, 캐싱은 **아예 호출하지 않는 일**입니다. 비용 0원, 지연 0.1초. 다만 잘못 쓰면 **틀린 답을 아주 빠르게** 돌려주는 최악의 기능이 됩니다. 오늘은 두 종류의 캐시를 구분해 봅니다.

---

## 🍱 두 캐시는 지갑의 다른 주머니다

| 구분 | **시맨틱 캐시** | **프롬프트 캐시** |
| --- | --- | --- |
| 일상 비유 | 단골손님의 "늘 먹던 것" | 밀프렙(재료 미리 손질) |
| 재사용 대상 | **질문과 답변 한 쌍** | **입력의 앞부분(접두부) 계산 결과** |
| 절약 지점 | 모델 호출 자체를 생략 (비용 0) | 입력 토큰 일부만 재과금 (할인) |
| 위험 | 비슷한 다른 질문에 엉뚱한 답 | 거의 없음(단, 접두부가 어긋나면 효과 0) |
| 구현 위치 | 내 애플리케이션 / Redis | 모델 제공사의 API 옵션 |

둘은 대체재가 아니라 **함께 쓰는 것**입니다. 시맨틱 캐시에 miss가 나도, 프롬프트 캐시 덕분에 그 요청은 싸게 처리됩니다.

---

## ① 시맨틱 캐시: 임계값이 전부다

원리는 단순합니다. 질문을 임베딩해 저장해 두고, 새 질문과 **코사인 유사도가 임계값 이상이면 저장된 답을 재사용**합니다.

문제는 임계값입니다.

- **0.80으로 낮추면** — "연차 며칠 남았어?"와 "병가 며칠 남았어?"가 같은 답을 받습니다. 히트율은 오르고 정확도는 무너집니다.
- **0.98로 올리면** — 거의 같은 문장만 히트합니다. 안전하지만 절감 효과가 사라집니다.

실무 기본값은 **0.95 근처에서 시작해 로그를 보며 조정**입니다. 그리고 임계값만으로는 부족하니 아래 **4가지 금지 규칙**을 함께 겁니다.

1. **권한·테넌트를 캐시 키에 넣는다** — `role:dept:doc_version` 없이 캐시하면, 임원용 답변이 사원에게 나갑니다(13일차 RBAC 사고와 동일한 결말).
2. **숫자·날짜가 다르면 무조건 miss** — "3일 연차"와 "5일 연차"는 유사도가 높아도 답이 다릅니다.
3. **개인화 질문은 캐시하지 않는다** — "내 급여 명세", "내 프로젝트 일정"은 사용자별로 답이 다릅니다.
4. **문서가 갱신되면 무효화** — `doc_version`을 키에 넣거나 TTL을 짧게. 오래된 규정을 자신 있게 답하는 것이 가장 위험합니다.

---

## ② 프롬프트 캐시: 변하는 것은 반드시 뒤로

모델은 입력을 앞에서부터 처리하며 각 토큰의 내부 계산 결과(KV 캐시)를 만듭니다. **앞부분이 완전히 같으면** 그 계산을 다시 하지 않고 재사용합니다.

그래서 **순서 설계**가 전부입니다.

```
[고정] 시스템 프롬프트 (역할·말투·금지사항)      ← 캐시됨
[고정] 사내 규정 요약 / DB 스키마 설명            ← 캐시됨
[가변] 검색된 문서 조각 (요청마다 다름)           ← 여기서부터 새로 계산
[가변] 사용자 질문 + 날짜 + 이름
```

흔한 실수는 **날짜나 사용자 이름을 맨 앞에 넣는 것**입니다. 그러면 접두부가 매번 달라져 캐시가 한 번도 맞지 않습니다. "오늘은 {date}입니다"를 시스템 프롬프트 맨 앞에 두는 순간 캐시는 죽습니다.

- 제공사마다 정책이 다릅니다(자동 캐싱 vs `cache_control`로 구간 지정, TTL 5분/1시간, 쓰기 프리미엄과 읽기 할인 배율). **정확한 배율은 각 사 공식 문서의 최신 값을 확인**하고, 비용 계산에는 반드시 그 배율을 넣으세요.
- RAG에서는 **검색 결과가 매번 달라 캐시 구간이 짧아집니다.** 그래서 "시스템 프롬프트 + 고정 규정"까지만 캐시하고, 검색 문서는 뒤에 두는 구성이 현실적입니다.
- 캐시 히트율은 **관측 대상**입니다. 히트율이 낮으면 접두부가 흔들리고 있다는 신호입니다(20일차).

---

## 💻 5분 실습: 권한 스코프가 들어간 시맨틱 캐시 60줄

임계값을 바꿔 가며 히트율과 **오답 히트**가 어떻게 변하는지 직접 봅니다.

```python
import math, re
from collections import Counter

def tokens(text):
    # 형태소 분석기 없이 단어 단위로 근사 (실제로는 임베딩 벡터를 씁니다)
    return Counter(re.findall(r"[가-힣a-zA-Z0-9]+", text))

def cosine(a, b):
    common = set(a) & set(b)
    num = sum(a[t] * b[t] for t in common)
    den = math.sqrt(sum(v * v for v in a.values())) * math.sqrt(sum(v * v for v in b.values()))
    return num / den if den else 0.0

def digits(text):
    return set(re.findall(r"\d+", text))

class SemanticCache:
    def __init__(self, threshold):
        self.store, self.threshold, self.hits, self.misses = [], threshold, 0, 0

    def _scope(self, role, doc_version):
        return f"{role}|{doc_version}"          # ① 권한·문서버전을 키에 포함

    def lookup(self, query, role, doc_version):
        q, best, best_sim = tokens(query), None, 0.0
        for e in self.store:
            if e["scope"] != self._scope(role, doc_version):
                continue
            if digits(query) != e["digits"]:     # ② 숫자·날짜가 다르면 무조건 miss
                continue
            sim = cosine(q, e["vec"])
            if sim > best_sim:
                best, best_sim = e, sim
        if best and best_sim >= self.threshold:
            self.hits += 1
            return best["answer"], round(best_sim, 3)
        self.misses += 1
        return None, round(best_sim, 3)

    def save(self, query, answer, role, doc_version):
        self.store.append({"scope": self._scope(role, doc_version), "vec": tokens(query),
                           "digits": digits(query), "answer": answer})

queries = [
    ("연차 휴가 며칠 남았어?", "staff", "v3"),
    ("연차 휴가 며칠 남았는지 알려줘", "staff", "v3"),   # 거의 같은 질문 → 히트 기대
    ("병가 휴가 며칠 남았어?", "staff", "v3"),           # 다른 제도 → 미스여야 정상
    ("연차 휴가 며칠 남았어?", "intern", "v3"),          # 권한 다름 → 스코프 미스
    ("연차 휴가 5일 남았어?", "staff", "v3"),            # 숫자 다름 → 즉시 미스
]

for threshold in (0.6, 0.7, 0.8):
    cache = SemanticCache(threshold)
    cache.save(queries[0][0], "2026년 잔여 연차 7.5일입니다.", "staff", "v3")
    results = []
    for q, role, ver in queries[1:]:
        answer, sim = cache.lookup(q, role, ver)
        results.append((q, "HIT " if answer else "MISS", sim))
    print(f"\n🔧 임계값 {threshold} → 히트 {cache.hits} / 미스 {cache.misses}")
    for q, state, sim in results:
        flag = "⚠️ 오답 히트" if (state == "HIT " and "병가" in q) else ""
        print(f"  {state} 유사도={sim:<5} | {q} {flag}")
```

**여기서 느껴야 하는 것**: 임계값 0.6~0.7에서는 **"병가"** 질문이 "연차" 답변을 받습니다(⚠️ 표시). 더 고약한 것은 0.7 설정입니다 — 정상 질문("…알려줘")은 유사도 0.671로 미스되는데, **틀린 질문만 0.75로 히트**합니다. 히트율만 보면 좋아 보이는 설정이 실제로는 오답을 유통시키는 설정입니다. 그래서 캐시는 **히트율과 오답률을 함께** 봐야 하고, 숫자·권한 규칙을 코드로 강제해야 합니다.

---

## 📉 실제로 얼마나 줄어드나 (가정 계산)

하루 1,000건, 그중 35%가 반복·유사 질문이고, 요청당 평균 비용 $0.02라고 가정하면:

| 구성 | 월 비용(30일, 가정) | 비고 |
| --- | --- | --- |
| 캐시 없음 | $600 | 기준선 |
| + 시맨틱 캐시(히트 35%) | $390 | 히트는 비용 0, 지연도 0.1초 수준 |
| + 프롬프트 캐시(입력 60% 재사용) | $300 내외 | 배율은 제공사 정책에 따라 달라짐 |

캐시는 **비용 절감**보다 **지연 단축(체감 품질)** 효과가 더 클 때도 많습니다. 첫 토큰까지의 시간(TTFT)이 2초에서 0.1초로 줄면 사용자는 "AI가 빨라졌다"고 느낍니다.

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"캐시는 성능 기능이 아니라 정합성 기능이다. SA는 '히트율 몇 %'가 아니라 '어떤 질문에 캐시를 쓰지 않을 것인가(권한·숫자·개인화)'를 먼저 문서로 못 박고, 그다음에 임계값을 정한다."**
