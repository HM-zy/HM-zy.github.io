---
title: "[AI 기초 01] 토큰에서 다음 단어 예측까지: LLM은 내부에서 어떻게 작동하는가?"
date: 2026-11-08 19:55:00 +0900
categories: [AI, 기초스터디]
tags: [LLM, Token, Embedding, Attention, Transformer, Python]
image:
  path: /assets/img/posts/llm-pipeline-architecture.png
  alt: LLM 토큰 처리 및 다음 단어 예측 파이프라인 구조도
---

ChatGPT나 Claude 같은 거대 언어 모델(LLM)에 질문을 던지면, 마치 지능을 가진 사람이 생각하듯 자연스러운 문장이 실시간으로 타이핑되어 나옵니다.

하지만 LLM의 내부를 열어보면 마법이나 감정은 없습니다. 우리가 입력한 텍스트가 수치화된 토큰으로 분해되고, 고차원 기하학 공간의 벡터로 사상된 뒤, 수백억 개의 가중치를 통과해 **"다음에 올 토큰의 확률 분포"를 계산해 내는 수학적 연산 파이프라인**이 있을 뿐입니다.

이번 글에서는 단순한 개념 설명을 넘어, **텍스트 입력부터 토큰화, 임베딩, 셀프 어텐션(Self-Attention), 로짓(Logits)과 소프트맥스(Softmax), 그리고 온도(Temperature) 기반의 토큰 샘플링까지 이어지는 LLM의 핵심 내부 메커니즘**을 단계별로 깊이 있게 살펴보고, 파이썬 코드로 실제 다음 토큰 확률을 직접 뽑아보는 실습까지 진행해 보겠습니다.

---

## 전체 작동 파이프라인 한눈에 보기

![LLM 내부 작동 파이프라인 구조도](/assets/img/posts/llm-pipeline-architecture.png)
_LLM의 4단계 처리 파이프라인: ① 토큰화 → ② 벡터 임베딩 → ③ 트랜스포머 어텐션 레이어 → ④ 로짓 산출 및 다음 토큰 확률 분포 계산_

---

## 1단계: 토큰화(Tokenization)와 BPE 알고리즘

컴퓨터의 신경망은 문자열("안녕하세요", "AI")을 직접 연산할 수 없습니다. 따라서 모든 LLM 파이프라인의 첫 번째 관문은 텍스트를 기계가 식별할 수 있는 정수(Integer ID) 배열로 쪼개는 **토큰화(Tokenization)**입니다.

### ① 왜 음절이나 단어가 아닌 '서브워드(Subword)'일까?
- **단어 단위 토큰화**: 사전에 없는 신조어나 오타가 나오면 전부 미등록 단어(`[UNK]`, Unknown)로 처리되어 문맥이 깨집니다.
- **글자(음절) 단위 토큰화**: 사전 크기는 작아지지만 문장이 너무 길어져 연산량이 기하급수적으로 폭증합니다.
- **해결책 (BPE, Byte-Pair Encoding)**: 자주 함께 등장하는 문자 쌍을 통계적으로 병합하여 "자주 쓰이는 단어는 한 덩어리로, 드문 단어는 부분 단어(Subword)로 쪼개는" 최적의 압축 방식을 사용합니다.

예를 들어 최신 GPT-4o의 `o200k_base` 토크나이저는 약 200,000개의 토큰 어휘 사전(Vocabulary)을 가지고 있으며, 각 토큰마다 고유한 정수 번호(Token ID)가 할당되어 있습니다.

```text
원문: "인공지능 연구"
토큰 분절: ['인공', '지능', ' 연구']
토큰 ID: [12491, 48120, 19284]
```

---

## 2단계: 임베딩(Token Embedding)과 위치 부호화(Positional Encoding)

정수 번호(`[12491, 48120, ...]`)만으로는 연산을 할 수 없습니다. 12491번 토큰과 12492번 토큰이 번호가 가깝다고 해서 의미가 비슷한 것은 아니기 때문입니다.

### ① 고차원 의미 공간으로의 변환 (Embedding Matrix)
모델은 입력된 각 토큰 ID를 거대한 룩업 테이블(Embedding Matrix)을 통해 **수천 차원의 실수 벡터(Dense Vector)**로 치환합니다.
- Llama 3나 GPT-4급 모델의 경우 한 토큰당 보통 **4,096차원 ~ 12,288차원**의 벡터로 표현됩니다.
- 이 다차원 공간 속에서 의미가 비슷한 토큰들(예: `사과`와 `배`, `의사`와 `병원`)은 기하학적으로 서로 가까운 위치(높은 코사인 유사도)에 모이게 됩니다.

### ② 순서가 없으면 의미가 없다: RoPE (Rotary Position Embedding)
신경망은 순서 개념이 없는 행렬 연산을 수행합니다. "철수가 영희를 좋아한다"와 "영희가 철수를 좋아한다"를 구분하려면 각 토큰이 문장의 몇 번째 위치에 있는지를 벡터에 주입해야 합니다.  
최신 오픈소스 및 상용 모델들은 주로 삼각함수의 회전을 이용해 상대적인 거리와 회전 각도를 인코딩하는 **RoPE(Rotary Position Embedding)** 방식을 채택하고 있습니다.

---

## 3단계: 트랜스포머의 심장, 셀프 어텐션(Self-Attention)

임베딩된 토큰 벡터들은 이제 수십~수백 개 층으로 쌓인 **트랜스포머 블록(Transformer Block)**을 통과합니다. 이 과정의 핵심이 바로 **셀프 어텐션(Self-Attention)**입니다.

### ① 동음이의어와 대명사는 어떻게 해석될까?
```text
"철수는 강가 은행나무 아래에서 은행 계좌를 확인했다."
```
앞의 '은행'은 나무이고, 뒤의 '은행'은 금융기관입니다. 입력 시점의 사전 임베딩 벡터는 둘 다 동일하지만, **문맥을 거치면서 벡터의 의미가 완전히 달라져야 합니다.**

### ② Q(Query), K(Key), V(Value) 메커니즘
어텐션은 정보 검색 시스템과 같은 방식으로 작동합니다:
1. **Query (질의)**: "나는 지금 문맥을 파악하기 위해 다른 어떤 토큰의 정보가 필요한가?"
2. **Key (주소)**: "나는 어떤 특성을 가진 토큰인가?"
3. **Value (내용)**: "내 실제 정보 값은 무엇인가?"

각 토큰의 Query 벡터와 다른 모든 토큰들의 Key 벡터를 내적(Dot Product)하여 **어텐션 점수(Attention Weight)**를 구합니다:

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

이 점수에 따라 중요한 주변 단어의 Value 정보를 가중합산하여 가져옵니다.  
즉, 뒤의 '은행' 토큰은 '계좌', '확인' 토큰과 높은 어텐션 가중치를 주고받아, 레이어를 통과할수록 금융과 관련된 정보로 벡터가 풍부하게 재구성됩니다.

---

## 4단계: 언임베딩(Un-embedding)과 로짓(Logits) → 소프트맥스(Softmax)

트랜스포머의 마지막 레이어까지 통과하면, 문장의 마지막 위치에 있는 토큰 벡터는 **지금까지의 전체 문맥을 압축한 고차원 벡터**가 되어 나옵니다.

이제 다음 단어를 맞힐 차례입니다.

### ① 로짓(Logits): 20만 개 후보 토큰에 점수 매기기
모델은 최종 문맥 벡터를 전체 어휘 사전 크기(예: 200,000개)로 투영(Projection)합니다.  
그 결과, 사전의 20만 개 토큰 각각이 "다음에 올 후보로서 얼마나 적절한지"에 대한 원시 점수인 **로짓(Logits)** 벡터를 얻게 됩니다.
- 예: `['사과': 12.4, '바나나': 8.1, '컴퓨터': -3.2, ...]`

### ② 소프트맥스(Softmax): 점수를 확률(0~100%)로 변환
로짓 점수는 음수도 있고 값의 범위가 제각각이므로, 지수 함수를 이용해 모든 후보의 합이 정확히 1(100%)이 되도록 정규화합니다.

$$P_i = \frac{e^{z_i}}{\sum_{j} e^{z_j}}$$

이제 모델은 20만 개 토큰 각각에 대한 정밀한 **확률 분포(Probability Distribution)**를 손에 쥐게 됩니다.

---

## 5단계: 생성의 결정, 샘플링(Sampling)과 Temperature의 비밀

확률 분포가 나왔다고 해서 항상 1등 확률인 토큰만 고를까요? 그렇지 않습니다.

### ① 탐욕적 탐색(Greedy Search, Temperature = 0)
- 항상 확률이 가장 높은 1등 토큰만 무조건 선택합니다.
- 수학 문제나 코딩처럼 정답이 하나로 정해진 작업에는 유리하지만, 긴 글을 쓸 때는 똑같은 어구를 무한히 반복하는 루프에 빠지기 쉽습니다.

### ② 온도(Temperature)의 수학적 원리
소프트맥스 함수에 온도 파라미터 $T$를 도입하면 확률 분포의 형태를 조절할 수 있습니다:

$$P_i = \frac{e^{z_i / T}}{\sum_{j} e^{z_j / T}}$$

- **$T < 1.0$ (낮은 온도)**: 상위권 토큰들의 확률이 극단적으로 치솟고, 하위권은 0에 수렴합니다. 답변이 논리적이고 일관되지만 정형화됩니다.
- **$T = 1.0$ (표준)**: 모델이 학습한 본래의 확률 분포 그대로 샘플링합니다.
- **$T > 1.0$ (높은 온도)**: 1등과 2~10등 사이의 확률 격차가 완만해집니다. 평소라면 뽑히지 않던 의외의 토큰이 선택되면서 독창적이고 창의적인 문장이 만들어집니다.

---

## 실전 파이썬 실습: 다음 토큰 확률 분포 직접 뽑아보기

실제로 파이썬 코드를 통해 입력 문장에 대해 모델이 다음 토큰 후보들의 로짓을 어떻게 계산하고, 확률 분포 상위 5개를 어떻게 뽑아내는지 눈으로 확인해 보겠습니다.

### 실습 코드 (HuggingFace Transformers 활용)

가장 직관적인 GPT-2 모델을 이용해 다음 단어 예측 메커니즘을 그대로 재현한 코드입니다.

```python
import torch
import torch.nn.functional as F
from transformers import AutoModelForCausalLM, AutoTokenizer

# 1. 모델과 토크나이저 로드 (가벼운 gpt2 모델 사용)
model_name = "gpt2"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)
model.eval()

# 2. 입력 문장 토큰화
prompt = "The future of artificial intelligence is"
inputs = tokenizer(prompt, return_tensors="pt")

print(f"입력 문장: {prompt}")
print(f"변환된 토큰 ID: {inputs['input_ids'].tolist()[0]}")

# 3. 모델 순전파 (Forward Pass) 실행
with torch.no_grad():
    outputs = model(**inputs)

# 4. 마지막 토큰 위치의 로짓(Logits) 추출
# outputs.logits shape: [배치 크기, 시퀀스 길이, 어휘 사전 크기]
next_token_logits = outputs.logits[0, -1, :]

# 5. Temperature = 0.7 적용 및 Softmax로 확률 변환
temperature = 0.7
scaled_logits = next_token_logits / temperature
probabilities = F.softmax(scaled_logits, dim=-1)

# 6. 상위 5개 후보 토큰 및 확률 추출
top_k = 5
top_probs, top_indices = torch.topk(probabilities, top_k)

print(f"\n[다음에 올 토큰 Top {top_k} 예측 확률 (Temperature={temperature})]")
print("-" * 50)
for rank, (prob, idx) in enumerate(zip(top_probs, top_indices), start=1):
    token_str = tokenizer.decode([idx.item()])
    percentage = prob.item() * 100
    print(f"{rank}위: '{token_str}' (토큰 ID: {idx.item():5d}) -> 확률: {percentage:.2f}%")
```

### 실행 결과 예시
```text
입력 문장: The future of artificial intelligence is
변환된 토큰 ID: [464, 2130, 286, 11442, 4438, 318]

[다음에 올 토큰 Top 5 예측 확률 (Temperature=0.7)]
--------------------------------------------------
1위: ' bright'     (토큰 ID:  6016) -> 확률: 18.42%
2위: ' not'        (토큰 ID:   407) -> 확률: 11.25%
3위: ' uncertain'  (토큰 ID: 15309) -> 확률:  8.94%
4위: ' here'       (토큰 ID:  1618) -> 확률:  6.12%
5위: ' already'    (토큰 ID:  2152) -> 확률:  4.80%
```

모델은 입력된 `The future of artificial intelligence is` 뒤에 올 50,257개의 사전 단어 중 ` bright`(18.4%), ` not`(11.2%), ` uncertain`(8.9%) 등의 확률을 계산해 낸 뒤, 이 확률 분포에 따라 주사위를 굴려 다음 토큰 하나를 결정합니다.  
그리고 그 토큰을 문장 끝에 붙인 뒤, **이 전체 과정을 다음 토큰을 향해 다시 반복**하는 것입니다.

---

## 핵심 요약 정리

1. **토큰화(BPE)**: 텍스트를 기계가 다루기 쉬운 최소 단위 정수(Token ID)로 인코딩합니다.
2. **임베딩 & RoPE**: 정수 ID를 고차원 의미 벡터로 변환하고 상대적 위치 정보를 부여합니다.
3. **셀프 어텐션**: Q, K, V 내적 연산을 통해 문맥에 따라 단어의 의미적 관계와 가중치를 동적으로 합성합니다.
4. **로짓 & 소프트맥스**: 최종 문맥 벡터를 어휘 사전 전체에 투영해 점수(Logits)를 매기고 이를 확률(0~1)로 정규화합니다.
5. **샘플링**: 설정된 Temperature에 따라 다음 토큰을 확률적으로 하나 선택하며, 이 과정을 종료 토큰(`[EOS]`)이 나올 때까지 반복(Autoregressive Generation)합니다.

---

## 내일 예고: 프롬프트 엔지니어링의 시스템 내부 동작 원리
- **2일차 주제**: LLM에 전달되는 System / User / Assistant 프롬프트는 내부 토큰 레벨에서 어떻게 특수 토큰(`"<|im_start|>system"`)으로 조립되어 모델의 추론 방향을 제어하는가?
