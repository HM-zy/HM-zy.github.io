---
title: "[AI 기초 08] 하이브리드 검색의 비밀: 키워드(BM25)와 벡터 임베딩을 결합하는 법"
date: 2026-11-15 08:00:00 +0900
categories: [AI, 기초스터디]
tags: [HybridSearch, BM25, VectorSearch, RAG, RRF, Architecture, AI기초]
---

현업에서 RAG 시스템을 만들고 가장 먼저 부딪히는 벽이 있습니다. 

*"벡터 검색이 의미를 잘 파악한다더니, 왜 우리 회사 제품 모델명이나 사번, 특허 번호는 하나도 못 찾지?"*

AI에게 *"퇴직금 언제 나와?"*라고 물어보면 귀신같이 취업규칙 문서를 찾아오지만, *"ERR-902 오류 코드 해결법 알려줘"*라고 물어보면 엉뚱한 에러 문서를 들고 옵니다. 

이 문제를 해결하기 위해 엔터프라이즈 RAG가 반드시 도입하는 표준 기술이 바로 **하이브리드 검색(Hybrid Search)**입니다.

---

## 🔍 두 명의 탐정: 키워드 탐정(BM25) vs 의미 탐정(Vector)

하이브리드 검색은 서로 다른 장점을 가진 두 가지 검색 방식을 동시에 돌려 상위 결과를 합치는 기법입니다.

1. **키워드 검색 (BM25 / Sparse)**
   - **특기**: 글자가 토씨 하나 안 틀리고 정확히 일치하는 단어 찾기.
   - **강점**: 사번(`E10492`), 오류 코드(`ERR-501`), 장비 품명(`ThinkPad-X1`), 특허 번호.
   - **약점**: *"월급"*과 *"급여"*가 같은 뜻인지는 전혀 모름.

2. **벡터 검색 (Dense Embedding)**
   - **특기**: 문맥과 뉘앙스를 파악해 '의미'가 비슷한 문장 찾기.
   - **강점**: *"휴가 규정 좀 알려줘"* ↔ *"연차 유급휴가 발생 기준"*.
   - **약점**: 복잡한 고유명사나 알파벳·숫자 조합은 벡터 공간에서 왜곡되어 엉뚱한 결과를 냄.

---

## ⚖️ RRF(Reciprocal Rank Fusion): 두 검색 결과를 공정하게 합치는 알고리즘

두 탐정이 각자 상위 10개의 문서를 가져왔을 때, 둘의 점수 체계가 달라 단순 덧셈을 할 수 없습니다. 이때 사용하는 표준 공식이 **RRF(순위 역수 융합)**입니다.

$$RRF\_Score = \sum \frac{1}{k + rank}$$

점수가 아니라 **'몇 위로 뽑혔는가(Rank)'**를 기준으로 순위를 매기기 때문에, 양쪽 검색 엔진 모두에서 상위권에 든 문서가 압도적인 1등으로 올라옵니다.

---

## 💻 5분 실습: 키워드 순위와 벡터 순위를 RRF로 합쳐보기

```python
def reciprocal_rank_fusion(bm25_ranks, vector_ranks, k=60):
    rrf_scores = {}
    
    # BM25 검색 순위 반영 (doc: rank)
    for rank, doc in enumerate(bm25_ranks, start=1):
        rrf_scores[doc] = rrf_scores.get(doc, 0.0) + (1.0 / (k + rank))
        
    # 벡터 검색 순위 반영
    for rank, doc in enumerate(vector_ranks, start=1):
        rrf_scores[doc] = rrf_scores.get(doc, 0.0) + (1.0 / (k + rank))
        
    # 최종 RRF 점수 기준 내림차순 정렬
    return sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)

# 검색 예시: "ERR-902 급여 정산 오류"
bm25_results = ["문서 C (오류코드 ERR-902)", "문서 A (급여 정산 지침)", "문서 B (연말정산)"]
vector_results = ["문서 A (급여 정산 지침)", "문서 D (시스템 장애 가이드)", "문서 C (오류코드 ERR-902)"]

final_results = reciprocal_rank_fusion(bm25_results, vector_results)

print("🏆 하이브리드 최종 랭킹:")
for rank, (doc, score) in enumerate(final_results, start=1):
    print(f"{rank}위: {doc} (점수: {score:.5f})")
```

키워드에서 1등을 한 `문서 C`와 벡터에서 1등을 한 `문서 A`가 서로의 단점을 보완하며 가장 높은 점수를 얻게 됩니다.

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"엔터프라이즈 사내 문서는 고유명사와 전문 용어가 넘쳐나므로, 순수 벡터 검색만 쓰는 것은 위험하며 BM25를 결합한 하이브리드 검색이 엔터프라이즈의 표준 아키텍처다."**
