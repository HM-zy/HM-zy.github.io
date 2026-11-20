---
title: "[AI 기초 23] 구조화 출력(Structured Outputs): JSON 스키마로 LLM 응답을 서류 양식에 가두는 법"
date: 2026-11-30 08:00:00 +0900
categories: [기초·개념]
tags: [StructuredOutput, JSONSchema, Validation, Pydantic, FunctionCalling, Reliability, Architecture, AI기초, AI]
image:
  path: /assets/img/posts/structured-output-schema.png
  alt: 자유 텍스트가 스키마 검증기를 통과해 정형 JSON 카드로 나오고 실패분은 복구 루프로 되돌아가는 구조도
---

*"프롬프트에 'JSON으로만 답해'라고 다섯 번 썼는데, 여섯 번째 요청에서 모델이 '네, 알겠습니다! 다음은 JSON입니다:'라고 말하고 시작했습니다. 그 한 줄에 파서가 죽었습니다."*

![자유 텍스트가 스키마 검증을 통과해 정형 JSON이 되고, 실패분은 복구 루프로 되돌아간다](/assets/img/posts/structured-output-schema.png)
_자유 텍스트가 스키마 검증을 통과해 정형 JSON이 되고, 실패분은 복구 루프로 되돌아간다_

에이전트가 다른 시스템과 대화할 때 **자유 텍스트는 쓸 수 없습니다.** 결제 금액이 문자열 `"3,000원"`으로 오면 정산 배치가 멈추고, `currency`에 `"￦"`가 오면 환율 테이블 조회가 실패합니다. 오늘은 LLM 응답을 **계약(스키마)** 으로 묶는 방법과, 어긋났을 때 **자동으로 고치는 루프**를 다룹니다.

---

## 📋 일상 비유: 자유기술 칸 vs 서식 칸

관공서 서류를 떠올려 보세요.

- **자유기술 칸**: "상황을 자세히 적어 주세요." → 사람마다 형식이 다르고, 담당자가 매번 읽어야 합니다.
- **서식 칸**: `성명 / 금액(숫자만) / 통화(KRW·USD) / 첨부 여부` → 칸이 정해져 있어 **기계가 바로 처리**합니다.

LLM에게 "JSON으로 답해"라고 부탁하는 것은 **자유기술 칸에 "혹시 서식처럼 써 주실 수 있나요?"라고 적어 두는 것**과 같습니다. 대부분 지켜 주지만, 지키지 않았을 때 막을 방법이 없습니다. 구조화 출력은 이 부탁을 **실제 서식 칸**으로 바꾸는 일입니다.

---

## 🧱 3단계 사다리: 부탁 → 강제 → 검증

| 단계 | 방법 | 보장 수준 |
|---|---|---|
| ① 프롬프트 부탁 | "JSON으로만 답해" | ❌ 보장 없음 |
| ② JSON 모드 | `response_format={"type": "json_object"}` | △ 문법만 JSON, 스키마는 모름 |
| ③ 스키마 강제 | `json_schema` / Structured Outputs | ⭕ 스키마 준수 (지원 모델 한정) |
| ④ 검증 + 복구 | 파싱 → 스키마 검증 → 재요청 | ⭕⭕ 우리 코드가 최종 책임 |

실무에서 중요한 사실: **③이 있어도 ④가 필요합니다.** 스키마 강제는 "이 모델, 이 API, 이 파라미터"에서만 유효하고, 폴백 모델로 넘어가는 순간(24일차) 보장이 사라집니다. 또 스키마를 지켜도 **값이 틀릴 수** 있습니다. 금액 자리에 `0`이 오면 스키마는 통과하지만 정산은 틀립니다.

> 구조화 출력은 "형식을 지켰는가"를 보장하고, "내용이 맞는가"는 검증 규칙과 사람이 봐야 합니다.

---

## 🔧 스키마를 좁게 쓰는 법 (실무 체크리스트)

1. **enum으로 선택지를 닫는다**: `currency`는 `str`이 아니라 `["KRW", "USD"]`. 자유 문자열은 반드시 오염됩니다.
2. **숫자는 숫자 타입으로**: `amount`를 `str`로 두면 `"3,000원"`, `"약 3천"`이 들어옵니다. 단위는 필드로 분리하세요.
3. **필수/선택을 구분한다**: `required`를 비워 두면 모델이 편한 대로 뺍니다. 반대로 전부 필수로 두면 "모름"을 지어냅니다 → **`null` 허용 필드**를 따로 두는 편이 안전합니다.
4. **`additionalProperties: false`**: 정의하지 않은 필드가 슬쩍 끼는 것을 막습니다. 이게 없으면 스키마 진화 과정에서 유령 필드가 쌓입니다.
5. **중첩은 2단까지**: 3단을 넘으면 모델이 중간 배열을 비워 보냅니다.

---

## 💻 5분 실습: 6개 응답을 검증하고, 자동 복구해 보자

모델이 실제로 자주 내는 **6가지 어긋남**을 준비했습니다. 코드펜스, 트레일링 콤마, 숫자 대신 문자열, enum 오염, 필드 누락, JSON 아예 없음. 검증 → 복구 루프를 직접 돌려 **통과율이 어떻게 변하는지** 확인합니다.

```python
import json
import re

# 우리 시스템이 기대하는 계약(스키마)
SCHEMA = {
    "person": str,                    # 문자열
    "amount": int,                    # 정수
    "currency": ("KRW", "USD"),       # 닫힌 선택지
    "tags": list,                     # 배열
}

FENCE = "`" * 3   # 코드펜스 3개를 문자열로 만들 때는 이렇게

# 모델이 실제로 내놓는 6가지 응답 패턴
raw_outputs = [
    '{"person": "홍길동", "amount": 12000, "currency": "KRW", "tags": ["신규"]}',
    FENCE + 'json\n{"person": "김철수", "amount": 5000, "currency": "KRW", "tags": ["재구매"],}\n' + FENCE,
    '{"person": "이영희", "amount": "7,000", "currency": "KRW", "tags": []}',
    '{"person": "박민수", "amount": 3000, "currency": "￦", "tags": ["환불"]}',
    '{"person": "최지우", "amount": 9000, "currency": "USD"}',
    "죄송하지만 금액 정보를 찾지 못했습니다. 확인 후 다시 요청드릴게요.",
]


def extract_json(text):
    """코드펜스·설명 문장을 걷어내고 첫 JSON 블록만 꺼낸다."""
    body = re.sub(FENCE + r"(?:json)?", "", text).strip("` \n")
    match = re.search(r"\{.*\}", body, re.S)
    return match.group(0) if match else None


def parse(text):
    body = extract_json(text)
    if body is None:
        raise ValueError("JSON 블록 없음")
    body = re.sub(r",\s*([}\]])", r"\1", body)   # 트레일링 콤마 제거
    return json.loads(body)


def validate(obj):
    """스키마 위반 목록을 사람이 읽을 수 있는 문장으로 돌려준다."""
    errors = []
    for key, rule in SCHEMA.items():
        if key not in obj:
            errors.append(f"필수 필드 누락: {key}")
            continue
        val = obj[key]
        if isinstance(rule, tuple):
            if val not in rule:
                errors.append(f"{key} 허용값 아님: {val!r}")
        elif rule is int:
            if isinstance(val, bool) or not isinstance(val, int):
                errors.append(f"{key} 정수 아님: {type(val).__name__}")
        elif not isinstance(val, rule):
            errors.append(f"{key} 타입 불일치: {type(val).__name__}")
    return errors


def repair(text):
    """실제로는 '아래 오류를 고쳐 같은 스키마로 다시 내라'고 모델을 재호출하는 단계.
    여기서는 같은 일을 규칙으로 흉내 낸다."""
    body = extract_json(text)
    if body is None:
        return None, ["복구 불가: JSON 자체가 없음"]
    try:
        obj = json.loads(re.sub(r",\s*([}\]])", r"\1", body))
    except json.JSONDecodeError as exc:
        return None, [f"복구 불가: {exc.msg}"]

    fixed = []
    if FENCE in text or re.search(r",\s*[}\]]", text):
        fixed.append("코드펜스·콤마 정규화")
    if isinstance(obj.get("amount"), str):
        digits = re.sub(r"[^\d]", "", obj["amount"])
        if digits:
            obj["amount"] = int(digits)
            fixed.append("amount 문자열→정수")
    CURRENCY = {"￦": "KRW", "원": "KRW", "$": "USD", "달러": "USD"}
    if obj.get("currency") in CURRENCY:
        obj["currency"] = CURRENCY[obj["currency"]]
        fixed.append("currency 코드 정규화")
    if "tags" not in obj:
        obj["tags"] = []
        fixed.append("tags 기본값 []")
    return obj, fixed


first_pass = repaired = escalated = 0
for i, raw in enumerate(raw_outputs, 1):
    # 1차: 파싱 + 검증
    try:
        obj = parse(raw)
        errors = validate(obj)
    except (ValueError, json.JSONDecodeError) as exc:
        obj, errors = None, [type(exc).__name__]

    if not errors:
        first_pass += 1
        print(f"{i}. 1차 통과 ✅  {obj}")
        continue

    # 2차: 오류를 붙여 재요청(복구)
    obj2, notes = repair(raw)
    errors2 = validate(obj2) if obj2 else notes
    if not errors2:
        repaired += 1
        print(f"{i}. 복구 후 통과 🔧  {obj2}  (수정: {', '.join(notes)})")
    else:
        escalated += 1
        print(f"{i}. 사람 확인 필요 🚨  사유: {errors2}")

# 검증 없이 json.loads만 썼다면 어떻게 됐을까?
naive_crash = naive_silent = 0
for raw in raw_outputs:
    try:
        obj_naive = json.loads(raw)
    except json.JSONDecodeError:
        naive_crash += 1
        continue
    if validate(obj_naive):
        naive_silent += 1

total = len(raw_outputs)
print()
print(f"1차 통과 {first_pass}/{total} → 복구 후 {first_pass + repaired}/{total} "
      f"(복구 성공 {repaired}, 에스컬레이션 {escalated})")
print(f"검증 없이 json.loads만 썼다면: 예외 {naive_crash}건, "
      f"잘못된 값이 조용히 통과 {naive_silent}건")
```

**실행 결과**

```text
1. 1차 통과 ✅  {'person': '홍길동', 'amount': 12000, 'currency': 'KRW', 'tags': ['신규']}
2. 1차 통과 ✅  {'person': '김철수', 'amount': 5000, 'currency': 'KRW', 'tags': ['재구매']}
3. 복구 후 통과 🔧  {'person': '이영희', 'amount': 7000, 'currency': 'KRW', 'tags': []}  (수정: amount 문자열→정수)
4. 복구 후 통과 🔧  {'person': '박민수', 'amount': 3000, 'currency': 'KRW', 'tags': ['환불']}  (수정: currency 코드 정규화)
5. 복구 후 통과 🔧  {'person': '최지우', 'amount': 9000, 'currency': 'USD', 'tags': []}  (수정: tags 기본값 [])
6. 사람 확인 필요 🚨  사유: ['복구 불가: JSON 자체가 없음']

1차 통과 2/6 → 복구 후 5/6 (복구 성공 3, 에스컬레이션 1)
검증 없이 json.loads만 썼다면: 예외 2건, 잘못된 값이 조용히 통과 3건
```

**체감 포인트**: 6건 중 2건만 그대로 통과합니다. 그중 2번은 원래 파서가 죽을 응답이었는데, **코드펜스·트레일링 콤마를 걷어내는 정규화** 덕분에 살았습니다. 모델을 비싼 것으로 바꿔도 이런 형식 오염은 사라지지 않습니다.

더 무서운 줄은 마지막입니다. 검증 없이 `json.loads`만 썼다면 **예외 2건**은 바로 보이지만, **3건은 "성공"으로 통과**합니다. 금액이 `"7,000"` 문자열이고, 통화가 `"￦"`이고, 태그가 없는데도요. 정산 배치는 그때부터 **조용히 틀린 값을 계산**합니다. 장애 알림도 없습니다.

또 하나, 5번(필드 누락)은 스키마 강제를 켜면 애초에 막힐 수 있습니다. 하지만 **6번처럼 JSON 자체가 안 나오는 경우**는 어떤 스키마 강제로도 못 막습니다. 그래서 마지막 방어선은 언제나 **우리 코드의 검증 + 사람 에스컬레이션**입니다. 5/6이 아니라 6/6을 목표로 설계하면 "없는 값을 지어내는" 더 나쁜 시스템이 됩니다.

---

## ⚠️ 자주 하는 실수 3가지

1. **파싱 성공을 검증 성공으로 착각**: `json.loads`가 통과해도 `currency`에 `"￦"`가 들어 있을 수 있습니다. **문법 검사(파싱)와 의미 검사(스키마·도메인 규칙)는 별개**입니다.
2. **복구 루프에 상한이 없음**: "고쳐 올 때까지 재시도"는 무한 재시도가 됩니다. **최대 1~2회, 그 다음은 에스컬레이션**이 정석입니다.
3. **스키마를 프롬프트에만 적음**: 프롬프트의 스키마와 코드의 검증 스키마가 따로 놀면, 프롬프트를 고친 날 검증이 조용히 헐거워집니다. **스키마는 코드 한 곳(Pydantic 모델 등)에서 정의하고 프롬프트는 그것을 불러 쓰세요.**

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"에이전트 시스템의 신뢰도는 모델 성능이 아니라 '계약을 어겼을 때 무슨 일이 일어나는가'로 측정된다."**
>
> 고객 PoV에서 구조화 출력은 늘 **연동 안정성** 질문으로 돌아옵니다. ① ERP로 넘길 때 필드가 어긋나면 어디서 멈추는가 ② 모델을 다른 벤더로 바꿔도 같은 스키마가 유지되는가 ③ 실패 건은 누가 어떻게 처리하는가. 답은 "스키마 강제를 씁니다"가 아니라 **"검증 계층과 에스컬레이션 큐가 있고, 벤더 교체 시에도 계약은 우리 코드에 있습니다"** 입니다. 이 한 문장이 아키텍처 성숙도를 보여줍니다.
