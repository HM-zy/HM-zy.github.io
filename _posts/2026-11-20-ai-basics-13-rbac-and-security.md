---
title: "[AI 기초 13] 엔터프라이즈 권한 격리와 보안: 멀티테넌시 RAG와 메타데이터 필터링"
date: 2026-11-20 08:00:00 +0900
categories: [기초·개념]
tags: [Security, RBAC, MultiTenancy, RAG, MetadataFiltering, Architecture, AI기초, AI]
---

*"신입사원이 챗봇에게 '우리 회사 임원 연봉 얼마야?'라고 물었더니, 이사회 회의록 문서를 읽고 답변해 버렸습니다."*

사내 모든 문서를 하나의 벡터 DB에 털어 넣으면 발생하는 최악의 보안 사고입니다. 검색 품질이 조금 떨어지는 건 다시 물어보면 되지만, 한 번 새어 나간 문서는 되돌릴 수 없습니다.

실제 기업에서는 부서별, 직급별, 프로젝트별로 열람 권한이 엄격히 나뉘어 있습니다. AI 시스템도 똑같이 통제해야 하고, 그 중심에 **역할 기반 접근 제어(RBAC)**와 **메타데이터 사전 필터링(Pre-filtering)**이 있습니다.

---

## 🎫 일상 비유: 사원증 태그가 찍힌 엘리베이터

회사 사옥 엘리베이터를 떠올려 보세요. 아무 층이나 버튼이 눌리지 않습니다. 사원증을 태그하면 **그 사람이 출입 권한을 가진 층의 버튼만 불이 들어옵니다.** 신입 사원은 3층(자기 부서)과 1층 로비만, 임원은 9층 임원실까지.

RAG 검색도 정확히 같아야 합니다. "질문이 들어옴 → 벡터 DB에서 비슷한 문서 Top-5 검색" 사이에 **사원증 태그를 찍는 단계**가 빠지면, 모델은 9층 문서를 읽고 그 내용을 답변에 실어 보냅니다.

---

## 🔑 RBAC: 권한을 '사람'이 아니라 '역할'에 붙인다

RBAC(Role-Based Access Control)는 개인별 권한표를 관리하지 않고, **역할(role)**에 권한을 묶어 두는 방식입니다.

| 역할 | 볼 수 있는 문서 |
|---|---|
| `intern` | 공통 규정·취업규칙 (`security_level` 1) |
| `staff` | 자기 부서 문서 + 공통 규정 (`security_level` 2 이하) |
| `exec` | 전사 문서 (`security_level` 3 포함) |
| `finance` | 재무·급여 문서 (부서 필터) |

문서(청크)마다 이 정보를 **메타데이터**로 붙여 둡니다. 벡터와 함께 저장되는 작은 라벨입니다.

```json
{
  "chunk_id": "hr-0031",
  "text": "2026년 임원별 스톡옵션 배정표...",
  "tenant_id": "acme",
  "department": "hr",
  "security_level": 3,
  "allowed_roles": ["exec"]
}
```

핵심은 **임베딩(벡터)에는 권한 정보가 없다**는 점입니다. 벡터는 "의미가 비슷한가"만 알 뿐, "이 사람이 읽어도 되는가"는 모릅니다. 그래서 권한은 반드시 벡터 밖의 메타데이터로 들고 다녀야 합니다.

---

## 🔒 검색 전 필터링(Pre-filtering) vs 검색 후 필터링(Post-filtering)

1. **사후 필터링 (Post-filtering) ❌ 위험**
   - 전체 문서에서 유사한 상위 5개를 먼저 뽑은 뒤, 사용자의 권한을 체크해 권한 없는 문서를 버리는 방식입니다.
   - 문제점: 상위 5개 중 4개가 임원용 문서라면 일반 사원에게는 **쓸 수 있는 문서가 1개**만 남습니다. 답변 품질이 무너지고, "이 문서는 못 봅니다"라는 식으로 존재 자체가 새는 경우도 생깁니다.
   - 더 나쁜 건, 필터를 코드에서 한 번만 빠뜨려도(예: 다른 API 경로) 그대로 유출된다는 점입니다. 안전장치가 "검색 뒤"에 있으면 우회로가 생깁니다.
2. **사전 필터링 (Pre-filtering) ⭕ 표준**
   - 벡터 검색을 시작할 때 DB 쿼리 자체에 `WHERE department IN ('dev') AND security_level <= 2` 조건을 걸어, **애초에 사용자가 읽을 수 있는 문서 범위 안에서만 유사도를 계산**합니다.
   - 남은 후보가 모두 볼 수 있는 문서이므로 Top-5가 그대로 쓸 만하고, 권한 없는 문서는 검색 단계에서부터 존재하지 않습니다.

---

## 🧱 실무 3중 방어선

한 겹으로는 부족합니다. 실제 엔터프라이즈 RAG는 세 겹으로 막습니다.

1. **검색 단계 — 사전 필터**: `tenant_id`·`department`·`security_level`을 쿼리 조건에 강제. 멀티테넌시는 컬렉션/네임스페이스를 물리적으로 분리하면 사고 반경이 더 줄어듭니다.
2. **생성 단계 — 시스템 프롬프트 가드**: "제공된 문맥에 없는 내용은 답하지 말 것"을 명시. 다만 프롬프트는 모델이 무시할 수 있으므로 **보조 수단**일 뿐, 유일한 방어선으로 삼지 않습니다.
3. **응답 단계 — 인용 검증 + 감사 로그**: 답변에 붙은 출처 ID가 실제로 그 사용자의 접근 가능 목록에 있는지 한 번 더 확인하고, "누가·언제·어떤 문서를 근거로 답했는지"를 남깁니다. 사고 조사와 컴플라이언스(ISO 27001, 금융 규제) 대응의 근거가 됩니다.

---

## 💻 5분 실습: 사전 필터 vs 사후 필터, 숫자로 비교하기

같은 질문을 두 방식으로 던져서 **일반 사원이 실제로 쓸 수 있는 근거가 몇 개 남는지** 직접 확인해 봅시다.

```python
# 사내 문서 벡터 DB 가상 데이터 (메타데이터 포함)
document_store = [
    {"id": 1, "text": "취업규칙: 연차 휴가 규정", "department": "hr", "level": 1, "allowed_roles": ["intern", "staff", "exec"]},
    {"id": 2, "text": "개발팀 가이드: AWS 접근 키 발급 절차", "department": "dev", "level": 2, "allowed_roles": ["staff", "exec"]},
    {"id": 3, "text": "2026년 임원별 스톡옵션 배정표", "department": "hr", "level": 3, "allowed_roles": ["exec"]},
    {"id": 4, "text": "임원 보상 규정: 성과급 지급 기준", "department": "hr", "level": 3, "allowed_roles": ["exec"]},
    {"id": 5, "text": "이사회 의사록: 2026년 보상위원회 결정 사항", "department": "hr", "level": 3, "allowed_roles": ["exec"]},
    {"id": 6, "text": "급여 지급일 및 명세서 확인 방법", "department": "hr", "level": 1, "allowed_roles": ["intern", "staff", "exec"]},
]

# 임베딩 유사도 대신 질의 키워드 매칭으로 점수를 흉내 냅니다 (실습용 단순화)
def similarity(query, text):
    return sum(1 for term in query.split() if term in text)

def post_filtering(query, user_role, k=3):
    """❌ 전체에서 k개 뽑은 뒤에 권한을 확인"""
    ranked = sorted(document_store, key=lambda d: similarity(query, d["text"]), reverse=True)
    top_k = ranked[:k]
    allowed = [d for d in top_k if user_role in d["allowed_roles"]]
    return top_k, allowed

def pre_filtering(query, user_role, k=3):
    """⭕ 권한 있는 문서만 후보에 올린 뒤 k개 뽑기"""
    candidates = [d for d in document_store if user_role in d["allowed_roles"]]
    ranked = sorted(candidates, key=lambda d: similarity(query, d["text"]), reverse=True)
    return candidates, ranked[:k]

query = "임원 보상 급여 규정"
role = "staff"

top_k, allowed = post_filtering(query, role)
print(f"👤 권한: {role} / 질의: '{query}'")
print("\n[사후 필터링 ❌]")
print(f"  검색된 Top-3: {len(top_k)}건 → 권한 통과: {len(allowed)}건")
for d in top_k:
    mark = "✅" if d in allowed else "🚫 차단"
    print(f"    {mark} | {d['text']}")
print(f"  → 남은 근거 {len(allowed)}건으로는 답변 품질이 무너집니다.")

candidates, chosen = pre_filtering(query, role)
print("\n[사전 필터링 ⭕]")
print(f"  검색 후보군: {len(candidates)}건 (권한 있는 문서만)")
print(f"  최종 Top-{len(chosen)}:")
for d in chosen:
    print(f"    ✅ {d['text']}")

# 같은 질의를 임원 권한으로 실행하면 후보군이 넓어집니다
_, exec_chosen = pre_filtering(query, "exec")
print(f"\n👤 권한: exec → 최종 Top-{len(exec_chosen)}")
for d in exec_chosen:
    print(f"    ✅ {d['text']}")
```

**체감 포인트**: 사후 필터링은 "차단은 했지만 쓸 게 남지 않은" 상태를 만들고, 사전 필터링은 검색 자체를 안전한 범위로 좁혀 후보를 온전히 남깁니다. 다만 staff에게는 임원용 문서가 애초에 존재하지 않으므로 결과가 다소 엉뚱하게 나오는 것이 정상입니다 — 사전 필터는 **'권한 안에서 최선의 근거'**를 줄 뿐, 없는 정보를 만들어 내지 않습니다. 이때 "확인할 권한이 없습니다"라고 정직하게 답하는 것이 환각으로 채우는 것보다 안전합니다. 권한이 곧 **검색 공간의 크기**를 결정한다는 감각이 SA 면접에서 바로 쓰입니다.

---

## ⚠️ 자주 하는 실수 3가지

1. **청크에 메타데이터를 안 붙임**: 문서 단위 권한만 정하고 청킹할 때 라벨을 잃으면, 필터 조건이 빈 값이 되어 전체 검색으로 되돌아갑니다. 청킹 파이프라인에서 권한 메타데이터를 **상속**시키세요.
2. **캐시 키에 권한을 빼먹음**: "같은 질문이면 캐시 재사용"에서 권한이 다른 사용자가 같은 답을 받는 사고가 납니다. 캐시 키에 `tenant_id`·역할·권한 버전을 반드시 포함합니다.
3. **권한 변경이 인덱스에 반영되지 않음**: 퇴사자·부서 이동 시 벡터 DB의 메타데이터가 낡으면 유령 권한이 남습니다. 권한 변경 이벤트를 받아 메타데이터를 갱신하는 경로를 설계 단계에서 넣어야 합니다.

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"엔터프라이즈 RAG 아키텍처에서 가장 먼저 묻는 질문은 '정확도가 몇 %인가'가 아니라 '사원과 임원의 데이터 격리가 어떻게 보장되는가'이다."**
>
> PoV(개념검증) 단계에서 고객 보안팀이 던지는 후속 질문은 대개 세 가지입니다. ① 권한이 바뀌면 인덱스는 어떻게 갱신되는가 ② 답변의 근거 문서를 감사 로그로 남길 수 있는가 ③ 테넌트 간 벡터가 물리적으로 분리되는가. 이 세 질문에 아키텍처 그림으로 답할 수 있으면 보안 검토는 통과입니다.
