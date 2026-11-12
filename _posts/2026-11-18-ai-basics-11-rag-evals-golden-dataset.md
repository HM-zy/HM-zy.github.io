---
title: "[AI 기초 11] RAG 정량 평가(Evals)의 모든 것: 골든 데이터셋과 채점 기준표 만들기"
date: 2026-11-18 08:00:00 +0900
categories: [AI, 기초스터디]
tags: [Evals, Ragas, Observability, RAG, Benchmarking, Accuracy, Hallucination, Architecture, AI기초]
---

*"우리 RAG 시스템 성능이 아주 좋습니다!"*

빅테크 SA 면접이나 엔터프라이즈 기술 미팅에서 이렇게 말하면 반드시 다음 질문이 날아옵니다.

*"좋다는 걸 어떻게 측정하셨나요? 정확도가 몇 %이고, 어떤 질문셋으로 테스트했습니까?"*

AI 시스템은 감이나 눈대중으로 평가할 수 없습니다. 시스템을 수정할 때마다 성능이 실제로 올랐는지 객관적으로 검증하는 정량 평가 체계, 즉 **Evals(Evaluation)**가 필수적입니다.

---

## 🎯 Evals의 기본 뼈대: 20개 골든 데이터셋(Golden Dataset)

RAG를 제대로 평가하려면 사람이 직접 정답을 적어둔 **기준 질문셋(평가 데이터셋)**을 최소 20~50개 준비해야 합니다.

질문 유형은 균형 있게 섞어야 합니다:
1. **단답형 사실 확인 (Fact Retrieval, 40%)**: *"연차 발생 기준은?"* → 특정 조항을 정확히 짚어내는가?
2. **다중 문서 비교 및 요약 (Multi-hop, 30%)**: *"신입사원과 3년 차의 복지 혜택 차이는?"* → 2개 이상의 문서를 조합하는가?
3. **규정 외 질문 (Out of Domain, 30%)**: *"점심 식대 영수증 한도는?"* (사규에 없는 내용) → 당당하게 거절(Fallback)하는가?

---

## 📊 RAG의 3대 핵심 평가 지표

| 지표 | 측정 대상 | 계산 방법 |
| :--- | :--- | :--- |
| **검색 적중률 (Hit Rate)** | 검색 엔진이 정답이 담긴 문서를 상위 3위 안에 찾아왔는가? | 정답 문서 포함 횟수 / 전체 질문 수 |
| **답변 정확도 (Answer Accuracy)** | LLM이 찾은 문서를 바탕으로 틀린 말 없이 올바른 결론을 냈는가? | 1점(정답) / 0.5점(부분일치) / 0점(오답) |
| **환각률 (Hallucination Rate)** | 본문에 없는 내용을 마치 사실인 것처럼 지어낸 비율 | 환각 발생 건수 / 전체 질문 수 |

---

## 📐 프로덕션 필수 3대 지표: Ragas, LLM 관측, LLMOps

엔터프라이즈 실무에서는 단순히 눈대중으로 테스트하지 않고 정량 평가, 관측, 운영 파이프라인을 구축합니다.

### 1. Ragas (RAG Assessment)
- **일상 비유**: 대입 수능 시험의 공인 국영수 4대 표준 채점 기준표.
- **핵심 4대 지표**:
  1. **충실도 (Faithfulness)**: 문맥에 없는 거짓말을 지어내지 않았는가? (환각 측정)
  2. **답변 관련성 (Answer Relevance)**: 사용자의 질문 의도에 딱 맞는 답을 내놓았는가?
  3. **문맥 정밀도 (Context Precision)**: 검색된 청크 중 진짜 정답이 맨 위에 랭크되었는가?
  4. **문맥 재현율 (Context Recall)**: 정답을 도출하는 데 필요한 모든 배경 정보를 누락 없이 긁어왔는가?

### 2. LLM 관측 (LLM Observability / Tracing)
- **일상 비유**: 수술실 환자의 실시간 바이탈 모니터이자, 통증 원인을 찾아내는 정밀 혈관 MRI.
- **핵심 원리**: 사용자의 질문부터 최종 답변까지 백그라운드에서 실행된 모든 검색 쿼리, 도구 호출, 소모 토큰, 단계별 지연시간을 한 줄씩 시각화해 추적하는 분산 트레이싱 시스템 (LangSmith, Arize Phoenix 등).

### 3. LLMOps (LLM Operations)
- **일상 비유**: 요리사 혼자 일하는 식당이 아니라, 식자재 입고부터 조리, 품질 검사, 매장 서빙, 고객 만족도 조사까지 전자동으로 돌아가는 프랜차이즈 공장 시스템.
- **핵심 원리**: '만들기(프롬프팅/RAG) → 평가(Evals/Ragas) → 배포(Serving/Gateway) → 모니터링/관측(Observability)'의 전 주기를 자동화하고 비용과 품질을 지속 통제하는 엔터프라이즈 운영 라이프사이클.

---

## 💻 5분 실습: Evals 채점표와 기본 평가 루프

```python
# 기준 골든 데이터셋 (질문, 기대 정답, 필수 출처 조항)
golden_dataset = [
    {
        "id": "Q1",
        "question": "경조사 휴가는 며칠인가요?",
        "expected_answer": "본인 결혼 시 5일 부여",
        "expected_doc": "제14조",
        "type": "Fact"
    },
    {
        "id": "Q2",
        "question": "개인 차량 주차비 지원되나요?",
        "expected_answer": "사규에 명시되지 않음 (거절 응답 필요)",
        "expected_doc": "None",
        "type": "Out-of-Domain"
    }
]

# 시스템 평가 결과 기록
def evaluate_rag_system(results):
    total = len(results)
    correct_count = 0
    
    print("📋 Evals 결과 리포트:")
    for r in results:
        is_pass = r["actual_doc"] == r["expected_doc"]
        if is_pass:
            correct_count += 1
        status = "✅ PASS" if is_pass else "❌ FAIL"
        print(f"[{r['id']}] {status} - 질문: {r['question']}")
        
    accuracy = (correct_count / total) * 100
    print(f"\n🎯 최종 검색 적중률: {accuracy:.1f}% ({correct_count}/{total})")

# 가상 테스트 실행
test_results = [
    {"id": "Q1", "question": "경조사 휴가?", "expected_doc": "제14조", "actual_doc": "제14조"},
    {"id": "Q2", "question": "주차비 지원?", "expected_doc": "None", "actual_doc": "제2조"} # 실패 사례!
]

evaluate_rag_system(test_results)
```

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"엔터프라이즈 AI 프로젝트에서 '평가 체계가 없다'는 것은 '품질을 통제할 수 없다'는 뜻이다. 첫 프로토타입 단계부터 기준 질문셋을 만들고 점수를 기록해야 한다."**
