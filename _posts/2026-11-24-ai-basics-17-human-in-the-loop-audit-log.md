---
title: "[AI 기초 17] Human-in-the-Loop와 감사 로그: 실수해도 사고가 나지 않는 결재선 설계"
date: 2026-11-24 08:00:00 +0900
categories: [기초·개념]
tags: [HITL, HumanInTheLoop, AuditLog, Approval, Governance, Risk, Agent, Architecture, AI기초, AI]
---

*"에이전트가 고객에게 환불을 처리했습니다. 금액이 0 하나 더 붙어 있었지만, 승인 버튼은 없었습니다."*

자동화의 가치는 **사람이 안 봐도 되는 일**을 늘리는 데 있고, 신뢰의 가치는 **사람이 꼭 봐야 할 일**을 놓치지 않는 데 있습니다. 이 둘을 동시에 얻는 장치가 **Human-in-the-Loop(HITL)**와 **감사 로그(Audit Log)**입니다.

---

## 🚦 모든 도구를 사람이 승인하면 아무도 안 쓴다

- **일상 비유**: 은행 창구. 100만 원 이하는 창구 직원이 바로 처리하고, 그 이상은 지점장 결재를 받습니다. **금액이 아니라 '되돌릴 수 있는가'로 선을 긋습니다.**
- **핵심 원리**: 도구를 **되돌릴 수 있는가(Reversible)**와 **밖으로 나가는가(External)** 두 축으로 나눠 위험 등급을 매기고, 등급별로 자동·사후확인·사전승인을 다르게 겁니다.

| 등급 | 성격 | 예시 도구 | 처리 |
| --- | --- | --- | --- |
| **LOW** | 읽기, 되돌릴 수 있음 | `query_sales`, `get_inventory`, 문서 검색 | 즉시 실행 |
| **MEDIUM** | 되돌릴 수 있지만 흔적이 남음 | `create_mail_draft`, `export_csv`, 대량 조회 | 실행 후 알림 / 1클릭 확인 |
| **HIGH** | **밖으로 나가거나 되돌릴 수 없음** | `issue_refund`, `send_mail`, `delete_record`, `change_permission` | **사전 승인 필수** |

---

## 🧩 승인 화면을 '예/아니오'로 만들면 아무도 읽지 않는다

승인 피로(Approval Fatigue)가 생기면 사람은 무의식적으로 예를 누릅니다. 그래서 승인 요청에는 **판단에 필요한 것만 5줄**로 담습니다.

1. **무엇을** 하려는가 — 도구 이름이 아니라 사람의 언어로 ("고객 A에게 12만 원 환불")
2. **왜** — 사용자의 원래 요청 문장
3. **어떤 인자로** — 실제 실행될 파라미터(금액·수신자·대상 레코드)
4. **영향 범위** — 몇 건, 어느 기간, 되돌릴 수 있는지
5. **누가** 승인하면 되는지 — 권한자만 버튼이 보이게

그리고 네 가지 안전장치를 반드시 붙입니다.

- **만료**: 30분 안에 승인 없으면 자동 취소(밤새 대기하다 다음 날 실행되는 사고 방지)
- **승인 대상 고정**: 승인 시점의 인자를 **해시로 묶어** 저장. 실행 직전 인자가 바뀌면 재승인.
- **1회 실행**: 멱등 키(Idempotency Key)로 같은 승인이 두 번 실행되지 않게.
- **거절 데이터**: 거절 사유를 쌓아 자동화 규칙을 조여 나감. 거절률이 높은 도구는 등급 재조정 신호입니다.

---

## 📜 감사 로그: 사고가 난 뒤에 답할 수 있어야 한다

AI 사고는 대부분 **"그때 무슨 일이 있었는지"를 아무도 설명하지 못할 때** 커집니다. 감사 로그는 다음 7가지에 답할 수 있어야 합니다.

| 항목 | 질문 |
| --- | --- |
| Who | 누가(사용자·에이전트·승인자) |
| When | 언제 (UTC + 서비스 타임존) |
| What | 어떤 도구를 어떤 인자로 |
| Why | 사용자의 원래 요청 문장 |
| Which | 어느 모델·프롬프트 버전 |
| Result | 성공/실패·에러 코드 |
| Approval | 누가 언제 승인했는지, 승인 해시 |

운영 원칙은 세 가지입니다.

- **Append-only**: 수정·삭제 API를 만들지 않는다.
- **해시 체인**: 각 레코드가 직전 레코드의 해시를 품는다 → 누가 중간을 고치면 사슬이 끊겨 즉시 드러남.
- **개인정보 분리**: 원문 PII는 마스킹해서 저장(16일차), 필요 시 별도 보관소 참조 ID만.

---

## 💻 5분 실습: 위험 등급 게이트 + 변조 감지 로그 60줄

승인 없는 고위험 실행이 막히는지, 승인 후 인자가 바뀌면 무효가 되는지, 로그를 몰래 고치면 들키는지까지 한 번에 확인합니다.

```python
import hashlib, json

HIGH_RISK = {"issue_refund", "send_mail", "delete_record", "change_permission"}
MEDIUM_RISK = {"create_mail_draft", "export_csv"}

def classify(tool):
    if tool in HIGH_RISK:
        return "HIGH"
    if tool in MEDIUM_RISK:
        return "MEDIUM"
    if tool.startswith(("query_", "get_", "search_")):
        return "LOW"
    return "HIGH"          # 모르는 도구는 보수적으로

def args_hash(args):
    return hashlib.sha256(json.dumps(args, sort_keys=True, ensure_ascii=False).encode()).hexdigest()[:8]

def execute(tool, args, approval=None):
    level = classify(tool)
    if level == "HIGH":
        if approval is None:
            return "⏸️  승인 대기 (approval_id 생성, 30분 만료)"
        if approval["tool"] != tool or approval["args_hash"] != args_hash(args):
            return "⛔ 승인된 내용과 인자가 다름 → 재승인 필요"
    return f"✅ 실행 완료 ({level})"

class AuditLog:
    def __init__(self):
        self.records = []

    def append(self, event):
        prev = self.records[-1]["hash"] if self.records else "GENESIS"
        body = {**event, "prev": prev}
        body["hash"] = hashlib.sha256(
            json.dumps(body, sort_keys=True, ensure_ascii=False).encode()
        ).hexdigest()[:12]
        self.records.append(body)
        return body["hash"]

    def verify(self):
        prev = "GENESIS"
        for i, r in enumerate(self.records):
            body = {k: v for k, v in r.items() if k != "hash"}
            expected = hashlib.sha256(
                json.dumps(body, sort_keys=True, ensure_ascii=False).encode()
            ).hexdigest()[:12]
            if r["hash"] != expected or body["prev"] != prev:
                return False, i
            prev = r["hash"]
        return True, None

log = AuditLog()
refund = {"customer": "A-1024", "amount": 120_000, "currency": "KRW"}

print(execute("query_sales", {"period": "2026-10"}))
print(execute("issue_refund", refund))
print(execute("issue_refund", refund, approval={"tool": "issue_refund", "args_hash": args_hash(refund)}))
print(execute("issue_refund", {**refund, "amount": 1_200_000},
              approval={"tool": "issue_refund", "args_hash": args_hash(refund)}))

log.append({"who": "agent-01", "tool": "query_sales", "args": {"period": "2026-10"}, "result": "ok"})
log.append({"who": "agent-01", "tool": "issue_refund", "args": refund,
            "approval": "manager@corp.com", "result": "ok"})
log.append({"who": "agent-01", "tool": "create_mail_draft", "args": {"to": "A-1024"}, "result": "ok"})

print("\n🔗 검증:", log.verify())
log.records[1]["args"]["amount"] = 12_000_000     # 누군가 로그를 조작
print("🔗 조작 후 검증:", log.verify(), "← 1번 레코드에서 사슬이 끊김")
```

**여기서 느껴야 하는 것**: 세 번째 실행과 네 번째 실행의 차이를 보세요. 승인은 *"환불해도 된다"*가 아니라 **"이 금액·이 고객에게 환불해도 된다"**에 대한 승인입니다. 그래서 승인은 인자 해시에 묶여야 합니다. 그리고 로그는 *"있으면 좋은 기록"*이 아니라 **조작을 스스로 증명하는 장치**여야 합니다.

---

## ⚠️ HITL이 형식만 남는 흔한 실패

- **모든 요청에 승인 버튼** → 3일 만에 습관적으로 클릭. (등급 설계 실패)
- **승인자가 자기 요청을 승인** → 직무 분리(Segregation of Duties) 위반.
- **승인 후 재시도**로 중복 실행 → 멱등 키 없음.
- **감사 로그를 운영 DB에 두고 앱 계정이 삭제 가능** → 별도 저장소·최소 권한.
- **프롬프트 원문 미보관** → 나중에 "왜 그랬나"를 재현할 수 없음(단, PII는 마스킹).

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"엔터프라이즈 AI 도입 심사에서 통과하는 문장은 '정확도 92%'가 아니라 '고위험 실행은 사람 승인 없이 절대 나가지 않고, 모든 실행은 변조 불가 로그로 남습니다'다. 자동화율은 그다음에 이야기한다."**
