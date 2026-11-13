---
title: "[AI 기초 03] Temperature와 Top-P: AI의 성향을 조절하는 모범생 vs 예술가 다이얼"
date: 2026-11-10 08:00:00 +0900
categories: [기초·개념]
tags: [Temperature, TopP, Hyperparameter, LLM, OpenAI, Claude, AI기초, AI]
image:
  path: /assets/img/posts/temperature-top-p-concept.png
  alt: AI 온도와 확률 조절 다이얼 개념도
---

똑같은 질문을 던져도 어떤 AI는 항상 토씨 하나 안 틀리고 정답만 말하고, 어떤 AI는 물어볼 때마다 기발하고 새로운 아이디어를 쏟아냅니다.

이 성향 차이를 만드는 핵심 컨트롤 다이얼이 바로 **Temperature(온도)**와 **Top-P(핵 샘플링)**입니다.

---

## 🎛️ Temperature: 모범생 다이얼 vs 예술가 다이얼

![AI 온도 조절 다이얼 개념도](/assets/img/posts/temperature-top-p-concept.png)
_냉철한 사실 전달(0.0)부터 자유로운 창작(1.0 이상)까지 조절하는 Temperature 메커니즘_

1일차 스터디에서 LLM이 다음 단어를 고를 때 **'단어별 확률 인기투표'**를 한다고 배웠습니다. Temperature는 이 투표 결과의 격차를 벌리거나 좁히는 다이얼입니다.

- **Temperature = 0.0 (얼음장처럼 차가운 모범생)**:
  - 1등 단어의 확률을 100%로 강제합니다.
  - 아무리 다시 실행해도 **항상 똑같은 단어**만 선택합니다.
  - **추천 영역**: 코딩, 수학 문제 풀이, 계약서 분석, RAG 검색 결과 요약 등 "정확한 사실"이 중요한 곳.
- **Temperature = 0.7 ~ 1.0 (따뜻하고 재치 있는 이야기꾼)**:
  - 1등과 2등, 3등 단어의 확률 차이를 부드럽게 깎아줍니다.
  - 가끔 2등이나 3등 단어도 골라지면서 문장이 훨씬 다채롭고 자연스러워집니다.
  - **추천 영역**: 일상 대화 챗봇, 마케팅 카피라이팅, 브레인스토밍.

---

## ✂️ Top-P: 엉뚱한 단어를 걸러내는 커트라인

Temperature와 함께 자주 쓰이는 **Top-P(누적 확률 컷오프)**는 후보군을 정리하는 가위입니다.

- **Top-P = 0.9**라면, 가장 확률이 높은 단어부터 차례로 더해 **누적 확률이 90%가 되는 상위 후보들만** 남기고, 나머지 10%의 엉뚱한 단어(확률 0.001%짜리 이상한 단어)는 후보에서 원천 배제합니다.
- 덕분에 모델이 너무 엉뚱한 환각(Hallucination)에 빠지는 것을 막아줍니다.

---

## 💻 5분 실습: Temperature 값에 따른 출력 비교

파이썬 환경에서 두 값을 조절하는 직관적인 코드 예시입니다.

```python
# 업무 성격에 따른 SA 권장 파라미터 세팅표
tasks = {
    "사내 규정 및 계약서 검토": {
        "temperature": 0.0,
        "top_p": 1.0,
        "이유": "일관되고 왜곡 없는 사실 전달 필수"
    },
    "고객 상담 챗봇": {
        "temperature": 0.5,
        "top_p": 0.85,
        "이유": "자연스러운 말투 유지 + 엉뚱한 답변 차단"
    },
    "신제품 아이디어 기획": {
        "temperature": 0.9,
        "top_p": 0.95,
        "이유": "기존 틀을 벗어난 창의적 발상 유도"
    }
}

for task_name, config in tasks.items():
    print(f"[{task_name}] Temp: {config['temperature']}, Top-P: {config['top_p']} -> {config['이유']}")
```

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"기업용 RAG 솔루션의 첫 번째 원칙은 Temperature를 0으로 고정하는 것에서 시작한다. 창의성은 RAG의 적이다."**
