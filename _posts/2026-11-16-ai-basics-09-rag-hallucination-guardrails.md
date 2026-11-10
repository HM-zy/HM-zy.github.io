---
title: "[AI 기초 09] RAG 환각(Hallucination) 방어: 가드레일과 Fallback 아키텍처"
date: 2026-11-16 08:00:00 +0900
categories: [AI, 기초스터디]
tags: [RAG, Hallucination, Guardrails, Fallback, PromptEngineering, Architecture, AI기초]
---

*"모르면 모른다고 솔직하게 말해줬으면 좋겠는데, 왜 자꾸 그럴듯하게 거짓말을 지어낼까요?"*

기업 고객이 RAG 도입을 가장 망설이는 단 하나의 이유는 바로 **환각(Hallucination)**입니다. 취업규칙에 육아휴직 급여 규정이 없는데도, 일반 인터넷 상식을 버무려 *"최대 200만 원까지 지급됩니다"*라고 지어내면 노사 분쟁이나 법적 책임으로 이어질 수 있습니다.

환각을 시스템적으로 차단하는 **가드레일(Guardrails)**과 **우아한 거절(Graceful Fallback)** 아키텍처를 살펴보겠습니다.

---

## 🛡️ 환각 방어의 3단계 안전벨트

1. **1단계: 엄격한 프롬프트 가드레일 (Context-Only Rule)**
   - 시스템 프롬프트에 *"제공된 문맥(Context)에 근거가 없다면 절대로 지어내지 말고 '사내 규정에 명시되어 있지 않습니다'라고 답하라"*는 명시적 제약 조건을 겁니다.
2. **2단계: 유사도 임계값 필터 (Score Threshold)**
   - 검색 엔진이 문서를 찾아왔더라도 유사도 점수가 기준치(예: 코사인 유사도 0.75) 미만이라면, 그 문서를 LLM에게 전달조차 하지 않고 즉시 사전 거절(Early Fallback) 처리합니다.
3. **3단계: 출처 각주 매핑 검증 (Citation Check)**
   - LLM이 답변을 작성할 때 *"어느 문단에서 가져왔는지 각주를 표기하라"*고 강제하면, 근거 없는 상상을 지어내는 확률이 80% 이상 급감합니다.

---

## 💻 5분 실습: 유사도 기반 조기 차단과 Fallback 프롬프트 템플릿

```python
def generate_rag_answer(query, search_results, threshold=0.75):
    # 상위 문서의 유사도 점수가 낮으면 LLM 호출 없이 즉시 Fallback
    top_doc, top_score = search_results[0]
    if top_score < threshold:
        return "❌ 죄송합니다. 질문과 직접 관련된 사내 공식 문서를 찾을 수 없습니다. (인사팀 문의 필요)"

    # 엄격한 제약을 건 시스템 프롬프트
    system_prompt = """당신은 사내 문서 전문 AI 비서입니다.
반드시 아래 [제공된 문서] 내용에만 기반하여 답변하세요.
문서에 명시되지 않은 내용은 추측하거나 지어내지 마세요.
답변 마지막에 참고한 문서의 제목을 [출처]로 명시하세요."""

    user_prompt = f"""[제공된 문서]:
{top_doc}

질문: {query}"""

    # 가상 LLM 응답 시뮬레이션
    return f"답변: {top_doc}에 따라 처리됩니다.\n[출처: 취업규칙 제14조]"

# 테스트
good_search = [("제14조(경조휴가): 본인 결혼 시 유급휴가 5일을 부여한다.", 0.88)]
bad_search = [("제2조(용어정의): 본 사규의 효력은...", 0.42)]

print("1. 관련 문서가 있을 때:\n", generate_rag_answer("결혼 휴가 며칠?", good_search))
print("\n2. 관련 문서가 없을 때:\n", generate_rag_answer("주차비 지원돼?", bad_search))
```

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"엔터프라이즈 AI에서는 '오답을 당당하게 말하는 것'보다 '모르는 것을 솔직하게 모른다고 거절하는 것'이 시스템의 신뢰도를 결정짓는 핵심이다."**
