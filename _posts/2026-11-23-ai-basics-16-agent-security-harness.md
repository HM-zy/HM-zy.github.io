---
title: "[AI 기초 16] 에이전트 보안 하네스: Read-only DB 격리, 도구 화이트리스트, PII 마스킹"
date: 2026-11-23 08:00:00 +0900
categories: [기초·개념]
tags: [Security, Harness, Whitelist, ReadOnly, PII, SQLGuardrail, Agent, Architecture, AI기초, AI]
---

*"데모 시연 중이었습니다. 에이전트가 '테스트 삼아 정리해 보겠습니다'라며 DELETE를 날렸고, 시연용 DB라서 망정이었죠."*

프롬프트에 *"절대 삭제하지 마"*라고 적어두는 것은 안전장치가 아닙니다. 그건 **부탁**입니다. AI가 실수해도 사고가 나지 않게 만드는 것은 모델 바깥의 코드와 인프라, 즉 **보안 하네스(Security Harness)**입니다.

---

## 🛡️ 하네스는 안전벨트가 아니라 '울타리 + 작업대'

- **일상 비유**: 아이에게 칼을 쥐여줄 때 "조심해"라고 말하는 대신 **둥근 칼만 주고, 손잡이를 잡게 하고, 앞에 손가락을 두지 않게** 만드는 것. 도구를 제한하고, 자세를 제한하고, 결과를 확인하는 3중 장치입니다.
- **핵심 원리**: 모델이 **무엇을 할 수 있는지(도구)**, **어디까지 읽을 수 있는지(자격증명)**, **무엇이 나가는지(출력)**를 코드 레벨에서 강제합니다. 프롬프트는 이 울타리 **안에서만** 작동합니다.

하네스는 네 겹으로 쌓습니다.

| 겹 | 막는 사고 | 구현 수단 |
| --- | --- | --- |
| ① 자격증명 격리 | 대량 삭제·스키마 변경 | **Read-only 계정 / 조회 전용 뷰** |
| ② 도구 화이트리스트 | 결제·발송 같은 위험 행동 | 노출 도구 목록을 코드로 고정 |
| ③ 입력 검증 | SQL 인젝션·다중 문장·대량 조회 | 쿼리 파서, SELECT-only, LIMIT 강제 |
| ④ 출력 마스킹 | 개인정보 유출·로그 2차 유출 | 정규식 기반 PII 토큰 치환 |

---

## ① 자격증명 격리: 권한은 프롬프트가 아니라 DB가 정한다

가장 강력한 방어는 **"그럴 권한 자체를 주지 않는 것"**입니다.

```sql
-- AI 전용 읽기 계정 (예시)
CREATE ROLE ai_readonly LOGIN PASSWORD '<vault에서 주입>';
GRANT CONNECT ON DATABASE erp TO ai_readonly;
GRANT USAGE ON SCHEMA public TO ai_readonly;
GRANT SELECT ON sales_v, inventory_v TO ai_readonly;   -- 뷰만, 원본 테이블 아님
-- INSERT/UPDATE/DELETE/DDL은 아예 부여하지 않는다
```

실무 포인트 세 가지:

- **뷰(View)를 노출**하면 원본 테이블의 민감 컬럼(주민번호, 계좌번호)을 애초에 뺄 수 있습니다.
- 가능하면 **읽기 전용 리플리카**에 붙여, 잘못된 쿼리가 운영 DB 성능을 흔들지 않게 합니다.
- 스테이트먼트 타임아웃(예: 10초)과 로우 리밋을 계정/세션 레벨에 함께 겁니다.

---

## ② 도구 화이트리스트: 목록에 없는 도구는 '존재하지 않는' 도구

MCP 서버(15일차)를 띄울 때 **등록할 도구를 최소로** 잡습니다.

| 등록 O | 등록 X |
| --- | --- |
| `query_sales` (조회) | `delete_order` |
| `get_inventory` (조회) | `issue_refund` |
| `create_mail_draft` (초안만) | `send_mail` (즉시 발송) |

"초안 생성"과 "발송"을 분리하는 것이 핵심입니다. 초안은 사람이 보고 보내면 되고(17일차 HITL), 발송 도구 자체가 없으면 AI가 실수로 보낼 방법도 없습니다.

---

## 💻 5분 실습: SQL 가드레일 + PII 마스킹 60줄

정상 쿼리 1개와 공격 시도 3개를 넣고, 하네스가 각각을 어떻게 처리하는지 봅니다.

```python
import re

ALLOWED_TABLES = {"sales_v", "inventory_v"}
BLOCKED_KEYWORDS = {"insert", "update", "delete", "drop", "alter", "truncate",
                    "grant", "create", "copy", "merge", "vacuum"}

def guard_sql(sql, max_rows=100):
    """SELECT 단일 문장 + 허용 테이블 + LIMIT 강제. 통과 시 (True, 보정된 SQL)."""
    stripped = re.sub(r"--[^\n]*|/\*.*?\*/", " ", sql.lower(), flags=re.S)  # 주석 제거
    if ";" in stripped:
        return False, "다중 문장(;) 차단"
    if not stripped.strip().startswith("select"):
        return False, "SELECT로 시작해야 함"
    for word in BLOCKED_KEYWORDS:
        if re.search(rf"\b{word}\b", stripped):
            return False, f"금지 키워드 감지: {word}"
    tables = set(re.findall(r"\b(?:from|join)\s+([a-z_][a-z0-9_]*)", stripped))
    if illegal := tables - ALLOWED_TABLES:
        return False, f"허용되지 않은 테이블: {sorted(illegal)}"
    if " limit " not in stripped:
        sql = sql.rstrip().rstrip(";") + f" LIMIT {max_rows}"
    return True, sql

PII_RULES = [
    (re.compile(r"[\w.+-]+@[\w-]+\.[\w.]+"), "[EMAIL]"),
    (re.compile(r"01[016789]-?\d{3,4}-?\d{4}"), "[PHONE]"),
    (re.compile(r"\b\d{6}-\d{7}\b"), "[RRN]"),
    (re.compile(r"\b\d{4}-\d{4}-\d{4}-\d{4}\b"), "[CARD]"),
]

def mask_pii(text):
    for pattern, token in PII_RULES:
        text = pattern.sub(token, text)
    return text

tests = [
    "SELECT region, SUM(amount) FROM sales_v WHERE period = '2026-10' GROUP BY region",
    "DELETE FROM sales_v",
    "SELECT * FROM sales_v; DROP TABLE users",
    "SELECT * FROM employees WHERE name = '김*' -- 필요한 것만",
]

for sql in tests:
    ok, result = guard_sql(sql)
    print(("✅ 허용" if ok else "⛔ 차단"), "|", sql[:52])
    print("   →", result if not ok else f"실행 쿼리: {result}\n")

# ⑤ 출력 마스킹: 조회 결과가 그대로 챗봇에 나가면 안 되는 경우
row = "김하늘 / haneul.kim@corp.com / 010-1234-5678 / 900101-1234567"
print("원본:", row)
print("마스킹:", mask_pii(row))
```

**여기서 느껴야 하는 것**: 세 번째 테스트처럼 공격자는 **정상 SELECT 뒤에 세미콜론을 붙여** 두 번째 문장을 끼워 넣습니다. 첫 줄만 보고 판단하는 필터는 여기서 뚫립니다. 그래서 "말로 부탁"이 아니라 **문장 단위 검사**가 필요합니다.

---

## ⚠️ 자주 뚫리는 구멍 체크리스트

1. **주석·개행 우회** — `DR/**/OP`, 줄바꿈으로 키워드 쪼개기 → 정규화 후 검사.
2. **UNION으로 다른 테이블 읽기** — 허용 테이블만 검사하면 `UNION SELECT ... FROM users`가 통과할 수 있음.
3. **느린 쿼리로 자원 고갈** — 카티션 곱·무한 재귀 → 타임아웃과 로우 리밋.
4. **도구 설명 속 지시(간접 프롬프트 인젝션)** — 웹 페이지·사내 문서에 심긴 문장이 모델을 조종. 외부 텍스트는 항상 '데이터'로만 취급.
5. **에이전트가 만든 파일을 다시 읽기** — 자기 출력을 신뢰하는 순환. 신뢰 경계 밖으로 나갔다 들어온 것은 재검증.

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"고객사에 'AI가 실수하지 않게 프롬프트를 잘 썼습니다'라고 말하면 신뢰를 잃는다. '삭제 권한이 없는 계정으로 붙였고, 도구 3개만 열었고, 결과는 마스킹해서 로그에 남깁니다'라고 말할 때 계약이 된다."**
