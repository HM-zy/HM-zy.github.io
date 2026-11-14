---
title: "[AI 기초 21] 엔터프라이즈 AI 하네스 최종 통합: RAG + MCP + FinOps 라우터 + Evals 아키텍처 총정리"
date: 2026-11-28 08:00:00 +0900
categories: [기초·개념]
tags: [EnterpriseAI, Harness, RAG, MCP, FinOps, Evals, Architecture, Integration, AI기초, AI]
---

*"3주 동안 개념은 다 배웠습니다. 그런데 면접에서 '그래서 전체 아키텍처를 한 장으로 그려보세요'라고 하면 손이 멈춥니다."*

개별 부품은 이미 다 배웠습니다. 오늘은 그 부품들이 **한 요청 안에서 어떤 순서로 만나는지**를 한 장으로 묶습니다. 이 그림 하나를 설명할 수 있으면, 어떤 꼬리 질문이 와도 부품 단위로 되돌아가 답할 수 있습니다.

---

## 🏭 전체 아키텍처: 요청 한 건이 지나가는 12개 관문

```
사용자 질문 (사내 챗봇 / 업무 앱)
        │
[1]  인증·권한 확인 (SSO · RBAC · 부서/직급)
        │
[2]  스마트 라우터 (FinOps 게이트웨이)
        │        ├─ 단순 질문 → 경량 모델
        │        └─ 복잡한 추론 → 대형 모델
[3]  시맨틱 캐시 ──── HIT ───────────────────────┐
        │ MISS                                  │
[4]  하이브리드 검색 (BM25 + Vector, RRF)         │
        │  + 메타데이터 사전 필터(권한)          │
[5]  리랭킹 · 컨텍스트 압축 (top_k 선별)          │
        │                                       │
[6]  프롬프트 조립                                │
        │  고정 접두부 = 프롬프트 캐시 구간      │
[7]  모델 호출 (도구 필요 시 MCP tools/call)      │
        │  └ 화이트리스트 · read-only 계정 · PII 마스킹
[8]  고위험 도구? ── YES ─→ HITL 승인 (없으면 실행 금지)
        │ NO / 승인 완료                        │
[9]  가드레일 · 출처 검증 (인용 없는 문장 차단)   │
        │                                       │
[10] 응답 + 출처 표시 ◄──────────────────────────┘
        │
[11] 감사 로그(해시 체인) + 트레이스·비용 기록
        │
[12] 골든셋 Evals 회귀 · 품질/비용 대시보드 (일·주 단위)
```

핵심은 **왼쪽(요청 경로)과 오른쪽(통제 경로)이 분리되어 있다**는 점입니다. 1·8·9·11은 통제, 2·3·5는 비용·속도, 4·6·7이 품질입니다. SA가 고객에게 설명할 때도 이 세 축으로 나누면 이야기가 흔들리지 않습니다.

---

## 🗺️ 3주 커리큘럼이 이 그림 어디에 꽂히나

| 주차 | 배운 것 | 아키텍처에서의 자리 |
| --- | --- | --- |
| 1주차 (1~7일차) | 토큰화, 역할 분리, Temperature·Top-P, 임베딩·벡터 DB, 청킹·출처, Function Calling, 상태 머신 에이전트 | [6] 프롬프트 조립, [7] 도구 호출의 기초 |
| 2주차 (8~14일차) | 하이브리드 검색(RRF), 환각 가드레일·Fallback, Streamlit RAG 데모, Evals 골든셋, 리랭킹·지연, 멀티테넌시·RBAC, ADR·PoV 제안서 | [4][5][9][12]와 의사결정 문서 |
| 3주차 (15~21일차) | MCP 아키텍처, 보안 하네스, HITL·감사 로그, 스마트 라우팅, 캐싱, 관측성, **최종 통합** | [1][2][3][7][8][11]과 전체 접합 |

이렇게 매핑해 두면 면접에서 "MCP가 뭐냐"는 질문에 개념만 답하는 대신, **"제 아키텍처의 [7]번 자리에 있고, 8번 승인 게이트와 함께 설계했습니다"**라고 위치로 답할 수 있습니다.

---

## 💻 5분 실습: 65줄 파이프라인 스텁으로 12관문 통과시키기

같은 파이프라인에 성격이 다른 질문 4개를 넣고, 각각 어느 관문에서 갈라지는지 확인합니다.

```python
DOCS = [
    {"id": "HR-01", "text": "연차 휴가는 입사 1년 후 15일 부여됩니다.", "roles": {"staff", "intern", "exec"}},
    {"id": "FN-07", "text": "출장비 정산은 귀사 후 7일 이내, 증빙 필수입니다.", "roles": {"staff", "exec"}},
    {"id": "EX-03", "text": "임원 스톡옵션 배정 기준은 이사회 승인 사항입니다.", "roles": {"exec"}},
]
COST = {"small": 0.0004, "large": 0.021}          # 요청당 가정 단가
HIGH_RISK = {"issue_refund", "send_mail"}

class Gateway:
    def __init__(self):
        self.cache, self.audit, self.spent, self.hits = {}, [], 0.0, 0

    def gate(self, n, msg):
        print(f"   [{n:>2}] {msg}")

    def run(self, question, role="staff", action=None, approval=None):
        print(f"\n❓ {question}   (role={role}" + (f", action={action})" if action else ")"))
        self.gate(1, f"인증·권한 확인 → {role}")
        tier = "large" if any(k in question for k in ("근거", "설계", "왜", "정리")) else "small"
        self.gate(2, f"라우팅 → {tier}")

        key = f"{role}|{question.strip().lower()}"
        if key in self.cache:
            self.hits += 1
            self.gate(3, "시맨틱 캐시 HIT → 비용 0, 지연 ~0.1s")
            self.gate(11, "감사 로그·트레이스 기록")
            return self.cache[key]
        self.gate(3, "캐시 MISS")

        words = [w for w in question.split()[:2]]
        docs = [d for d in DOCS if role in d["roles"] and any(w in d["text"] for w in words)]
        self.gate(4, f"하이브리드 검색 + 권한 필터 → {[d['id'] for d in docs] or '없음'}")
        self.gate(5, "리랭킹 top_k=2 · 컨텍스트 압축")
        self.gate(6, "프롬프트 조립 (고정 접두부는 프롬프트 캐시 구간)")
        self.spent += COST[tier]
        self.gate(7, f"모델 호출 ({tier}, ${COST[tier]})")

        if action:
            level = "HIGH" if action in HIGH_RISK else "LOW"
            self.gate(8, f"도구 {action} 위험등급={level}")
            if level == "HIGH" and approval != action:
                self.audit.append({"q": question, "action": action, "result": "PENDING"})
                self.gate(11, "HITL 대기 상태로 감사 로그 기록")
                return "⏸️ 승인 대기 (30분 내 미승인 시 자동 취소)"
            self.gate(8, "승인 확인 → read-only 계정·화이트리스트로 실행")

        self.gate(9, "가드레일: 인용 없는 문장 제거")
        answer = f"{docs[0]['text']} [출처 {docs[0]['id']}]" if docs else "근거 문서를 찾지 못했습니다."
        self.gate(10, "응답 + 출처 표시")
        self.cache[key] = answer
        self.audit.append({"q": question, "role": role, "tier": tier, "result": "ok"})
        self.gate(11, f"감사 로그·트레이스·비용 누적(${self.spent:.4f})")
        return answer

gw = Gateway()
scenarios = [
    ("연차 규정 알려줘", "staff", None, None),          # 단순 + 캐시 대상
    ("연차 규정 알려줘", "staff", None, None),          # 같은 질문 → 캐시 HIT
    ("출장비 정산 근거 정리해줘", "intern", None, None),  # 권한 필터로 문서 0건
    ("환불 처리해줘", "staff", "issue_refund", None),    # 고위험 → 승인 대기
    ("환불 처리해줘", "staff", "issue_refund", "issue_refund"),  # 승인 후 실행
]
for q, role, action, approval in scenarios:
    print("→", gw.run(q, role, action, approval))

print(f"\n📊 요약: 요청 {len(scenarios)}건, 캐시 히트 {gw.hits}건, 누적 비용 ${gw.spent:.4f}, "
      f"감사 로그 {len(gw.audit)}건")
print("   → [12] 다음 단계: 이 로그를 골든셋 Evals와 대시보드로 되먹임")
```

**여기서 느껴야 하는 것**:
- 같은 질문 두 번은 **두 번째에서 비용이 0**이 됩니다.
- `intern`은 같은 질문에도 **FN-07 문서를 볼 수 없습니다**(권한 필터가 검색 전에 작동).
- 환불 요청은 **승인 없이는 절대 실행되지 않습니다**.
- 그리고 마지막 줄 — 이 모든 기록이 **골든셋 평가와 대시보드로 돌아가는 순 환**을 만듭니다. 이 순환이 없으면 아키텍처는 데모에서 멈춥니다.

---

## 🎤 면접 3분 답변 템플릿 (그림을 그리며 말하기)

> "저는 사내 지식 Q&A와 업무 자동화를 하나의 하네스로 설계했습니다.
> **첫째, 연결**은 MCP로 표준화했습니다. 레거시 ERP에 서버 하나를 얹으면 어떤 AI 프런트엔드든 재사용할 수 있어, 연동 개발을 시스템 수만큼 늘리지 않습니다.
> **둘째, 안전**은 하네스로 강제했습니다. 조회는 read-only 계정과 화이트리스트 도구만, 고위험 실행은 사람 승인 없이는 나가지 않고, 모든 실행은 변조 감지되는 감사 로그로 남깁니다.
> **셋째, 비용**은 게이트웨이에서 라우팅과 캐싱으로 통제했습니다. 단순 문의는 경량 모델, 복잡한 추론만 대형 모델로 보내고, 반복 질문은 캐시로 호출 자체를 생략합니다.
> **넷째, 품질**은 골든셋 30문항으로 회귀 검증하고, 트레이스로 요청 단위 지연·비용·실패 지점을 추적합니다.
> 결과적으로 파일럿에서 프로덕션으로 넘길 때 필요한 **품질·비용·보안·설명가능성** 네 가지를 모두 수치로 제시할 수 있었습니다."

*(수치는 본인 프로젝트의 실제 측정값으로 바꿔 말하세요. 측정하지 않은 숫자는 말하지 않는 것이 원칙입니다.)*

---

## ✅ 고객사에 남기는 산출물 체크리스트

| 산출물 | 왜 필요한가 |
| --- | --- |
| **아키텍처 1장** (오늘 그림) | 회의 시작 3분을 아껴준다 |
| **ADR 3장** — 검색 방식, MCP 도입, 라우팅 정책 | "왜 이 선택인가"에 문서로 답한다 |
| **권한 매트릭스** — 역할 × 문서/도구 | 보안 심사에서 가장 먼저 요구된다 |
| **골든셋 20~30문항** + 측정 결과 | 데모와 프로덕션을 가르는 기준 |
| **비용 대시보드** — 요청당·팀별·캐시 히트율 | FinOps 승인의 근거 |
| **장애 런북** — 모델 장애·도구 실패·캐시 오염 시 조치 | 운영 인수인계의 필수품 |

---

## 🎯 Solution Architect 관점 마지막 한 줄
> **"엔터프라이즈 AI의 완성은 똑똑한 모델을 붙이는 것이 아니라, 요청 하나가 통과하는 관문을 설계하고 그 관문마다 근거를 남기는 것이다. 그 관문의 설계도가 곧 SA의 언어다."**

3주 동안 21편을 달려왔습니다. 이제 이 그림 한 장을 자기 말로 그릴 수 있으면, 어떤 아키텍처 질문에도 부품 단위로 답할 수 있습니다. 수고하셨습니다.
