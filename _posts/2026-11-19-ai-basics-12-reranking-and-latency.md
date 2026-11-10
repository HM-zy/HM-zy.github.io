---
title: "[AI 기초 12] 대규모 RAG 속도 개선: 리랭킹(Reranking)과 지연시간(Latency) 트러블슈팅"
date: 2026-11-19 08:00:00 +0900
categories: [AI, 기초스터디]
tags: [Reranking, Latency, RAG, Optimization, Architecture, AI기초]
---

*"검색 정확도를 높이려고 문서를 20개씩 LLM에 넘겼더니, 답변이 나오는 데 8초가 넘게 걸립니다."*

엔터프라이즈 환경에서 지연시간(Latency)은 사용자 경험을 망치는 주범입니다. 정확도를 챙기자니 문서를 많이 읽혀야 하고, 속도를 챙기자니 문서를 적게 읽혀 검색 누락이 발생합니다.

이 트레이드오프를 해결하는 표준 2단계 파이프라인이 바로 **리랭킹(Reranking)**입니다.

---

## ⚡ 2단계 검색 파이프라인 (Two-Stage Retrieval)

1. **1단계: 고속 대량 추출 (Bi-Encoder / Dense Vector)**
   - 수십만 개 문서 중에서 질문과 관련 있어 보이는 후보군 **Top-20개**를 0.05초 만에 빠르게 건져 올립니다. (속도 중심)
2. **2단계: 정밀 재순위 산정 (Cross-Encoder / Reranker)**
   - Top-20개 후보군을 전문 리랭커 모델(Cohere Rerank, BGE-Reranker)에 넣어 질문과의 실제 밀접도를 심층 재채점하고, 가장 정확한 **Top-3개**만 최종 압축합니다. (정확도 중심)

이렇게 하면 LLM이 읽어야 하는 문서 분량이 80% 줄어들어 **답변 지연시간(Latency)을 2~3초대로 대폭 단축**할 수 있습니다.

---

## 💻 5분 실습: 리랭커의 동작 원리 시뮬레이션

```python
# 1단계 벡터 검색으로 뽑힌 Top-5 후보
candidate_chunks = [
    {"doc": "사내 복지 가이드: 복지포인트 연간 100만 원 지급", "initial_rank": 1},
    {"doc": "취업규칙 제14조: 경조휴가 및 경조금 지급 기준", "initial_rank": 2},
    {"doc": "인사고과 평가 규정: 분기별 평가 프로세스 안내", "initial_rank": 3},
    {"doc": "퇴직금 지급 지침: DC형 퇴직연금 운용 방법", "initial_rank": 4},
]

# 2단계 질문과 문맥을 동시에 교차 채점하는 Cross-Encoder 시뮬레이션
def simple_rerank(query, candidates):
    query_terms = set(query.split())
    for item in candidates:
        # 단어 정밀 매칭 및 문맥 가중치 부여 (가상 리랭킹 스코어)
        doc_terms = set(item["doc"].split())
        match_count = len(query_terms.intersection(doc_terms))
        item["rerank_score"] = match_count * 0.4 + (0.5 if "경조금" in item["doc"] else 0.1)
        
    return sorted(candidates, key=lambda x: x["rerank_score"], reverse=True)

query = "결혼할 때 경조금 얼마나 나와?"
reranked = simple_rerank(query, candidate_chunks)

print("🎯 리랭킹 후 최종 Top-2 선정:")
for rank, item in enumerate(reranked[:2], start=1):
    print(f"{rank}위: {item['doc']} (리랭크 점수: {item['rerank_score']:.2f})")
```

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"검색 후보를 무작정 늘리는 것은 토큰 비용과 지연시간을 폭발시킨다. Top-20 추출 후 Top-3로 압축하는 Two-Stage Reranking이 엔터프라이즈의 속도와 정확도 균형점이다."**
