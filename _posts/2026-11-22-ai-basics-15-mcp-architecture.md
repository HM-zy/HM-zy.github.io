---
title: "[AI 기초 15] MCP(Model Context Protocol) 아키텍처: AI와 사내 DB·API를 잇는 USB-C 표준"
date: 2026-11-22 08:00:00 +0900
categories: [기초·개념]
tags: [MCP, ModelContextProtocol, JSONRPC, Host, Client, Server, ToolUse, Architecture, AI기초, AI]
---

*"사내 ERP에 AI를 붙이려면 연동 개발이 석 달 걸린다더라. 그런데 옆 팀은 이틀 만에 붙였대요. 뭐가 다른 거죠?"*

답은 케이블입니다. 노트북마다, 모니터마다 충전 단자가 달랐던 시절에는 기기마다 전용 케이블을 새로 사야 했습니다. USB-C 하나로 통일되자 케이블을 다시 설계할 이유가 사라졌죠. **MCP(Model Context Protocol)**는 AI와 기업 데이터 사이에서 똑같은 일을 합니다.

---

## 🔌 MCP를 한 문장으로: AI계의 USB-C

- **일상 비유**: 사내 DB, 슬랙, 깃허브, 파일 서버가 저마다 다른 단자를 쓰고 있었다면, MCP는 그 모든 곳에 꽂히는 **만능 C타입 단자**입니다.
- **핵심 원리**: MCP 서버는 "내가 제공하는 도구 목록과 사용법(스키마)"을 표준 형식(**JSON-RPC 2.0**)으로 알려주고, AI 앱(호스트)은 필요할 때 그 도구를 호출합니다. 즉 **AI가 DB에 직접 들어가는 게 아니라, 정해진 창구를 통해서만 요청**합니다.

> 연결 방식이 표준화되면, 도구를 가진 쪽은 **서버 하나만 띄워두면** 되고, AI를 쓰는 쪽은 **코드 수정 없이 플러그만 꽂으면** 됩니다.

---

## 🏗️ Host / Client / Server 3층 구조

MCP를 이해하는 가장 빠른 길은 '노트북에 주변기기를 꽂는 그림'을 그대로 옮겨 보는 것입니다.

| 계층 | 노트북 비유 | 실제 예시 | 하는 일 |
| --- | --- | --- | --- |
| **Host** | 노트북 본체 | Claude Desktop, Claude Code, 사내 챗봇 UI | 사용자와 대화하고, **권한·승인·대화 맥락**을 관리 |
| **Client** | USB-C 포트 컨트롤러 | Host 내부의 연결 객체 (서버당 1개) | 서버와 **1:1 연결**을 맺고 메시지를 중계 |
| **Server** | 모니터·외장하드 | 사내 ERP MCP 서버, Slack MCP 서버, GitHub MCP 서버 | **실제 실행** (SQL 실행, API 호출, 파일 읽기) |

핵심은 **책임이 섞이지 않는다**는 점입니다.

- 모델은 DB 비밀번호를 모릅니다. 테이블 구조도 모릅니다.
- 모델이 아는 것은 오직 **"이런 도구가 있고, 이런 인자를 받는다"**는 목록뿐입니다.
- 실제 실행 권한은 **서버 프로세스**가 가진 계정에만 있습니다.

이 분리가 바로 다음 시간(16일차 보안 하네스)에서 배우는 **보안 경계선**의 출발점입니다.

---

## 📨 실제로 오가는 메시지: 대화 상대는 세 가지

MCP 서버가 제공하는 것은 크게 세 갈래입니다.

1. **Tools (도구)** — 실행을 요청하는 것. `query_sales(period, region)` 처럼 인자를 받고 결과를 돌려줍니다. AI가 스스로 판단해 호출하는 쪽.
2. **Resources (자원)** — 읽기 전용 데이터. 파일 내용, 테이블 스키마처럼 "가져다 참고하는" 것.
3. **Prompts (프롬프트)** — 서버가 미리 만들어 둔 프롬프트 템플릿. 사용자가 `/` 명령처럼 골라 씁니다.

그리고 연결은 늘 이 순서로 시작합니다.

| 순서 | 메서드 | 무슨 일이 일어나나 |
| --- | --- | --- |
| ① | `initialize` | 악수. 서로의 프로토콜 버전과 지원 기능을 확인 |
| ② | `tools/list` | 서버가 도구 목록과 입력 스키마를 건네줌 |
| ③ | `tools/call` | 모델이 고른 도구를 인자와 함께 실행 요청 |

**전송 방식**은 두 가지입니다. 내 PC에서 도는 서버라면 **stdio**(표준 입출력으로 프로세스끼리 대화), 원격·사내 서버라면 **Streamable HTTP**(게이트웨이 뒤에 두고 인증을 붙임). 실무에서는 후자가 대부분입니다.

---

## 🧮 N×M 문제를 N+M으로 바꾼다

이게 MCP가 순식간에 표준이 된 진짜 이유입니다.

- **전통 방식**: AI 앱 5개 × 데이터 소스 5개 = **커넥터 25개**를 각각 개발·유지보수.
- **MCP 방식**: 서버 5개 + 클라이언트 구현 5개 = **연결점 10개**. 조합이 늘어도 총량이 곱셈이 아니라 덧셈으로 늘어납니다.

SA 입장에서 이 말은 곧 **"고객사 레거시 시스템을 건드리지 않고 AI를 붙일 수 있다"**는 영업 언어가 됩니다. 기존 ERP에 표준 MCP 서버를 하나 얹는 것만으로, 이후 어떤 AI 프런트엔드가 와도 그대로 재사용됩니다.

---

## 💻 5분 실습: 55줄로 MCP 서버·클라이언트 흉내내기

외부 라이브러리 없이, 실제 MCP 메시지 모양 그대로 최소 구현을 돌려봅니다.

```python
import json

# ── 1. MCP 서버: 실제 실행을 담당하는 쪽 (사내 DB 바로 앞에 서 있음)
TOOLS = {
    "query_sales": {
        "description": "기간(YYYY-MM)과 지역을 받아 매출 합계를 조회합니다.",
        "inputSchema": {"period": "string", "region": "string"},
    },
    "get_inventory": {
        "description": "제품 코드로 현재 재고 수량을 조회합니다.",
        "inputSchema": {"sku": "string"},
    },
}
FAKE_DB = {("2026-10", "서울"): 128_000_000, ("2026-10", "부산"): 41_500_000}
STOCK = {"SKU-1001": 32, "SKU-2002": 0}

def server_handle(method, params):
    """JSON-RPC 2.0 형태의 요청을 처리해 결과를 돌려준다."""
    if method == "initialize":
        return {"protocolVersion": "2025-06-18",
                "serverInfo": {"name": "erp-mcp", "version": "1.0"}}
    if method == "tools/list":
        return {"tools": [{"name": name, **spec} for name, spec in TOOLS.items()]}
    if method == "tools/call":
        tool, args = params["name"], params.get("arguments", {})
        if tool == "query_sales":
            value = FAKE_DB.get((args["period"], args["region"]), 0)
            return {"content": [{"type": "text", "text": f"{value:,}원"}]}
        if tool == "get_inventory":
            return {"content": [{"type": "text", "text": f"{STOCK.get(args['sku'], -1)}개"}]}
        return {"isError": True, "content": [{"type": "text", "text": f"unknown tool: {tool}"}]}
    return {"isError": True, "content": [{"type": "text", "text": f"unknown method: {method}"}]}

# ── 2. MCP 클라이언트: Host 안에서 서버 하나와 1:1로 연결되는 포트
class MCPClient:
    def __init__(self, name):
        self.name = name

    def request(self, method, params=None):
        msg = {"jsonrpc": "2.0", "id": 1, "method": method, "params": params or {}}
        print(f"[client→server] {json.dumps(msg, ensure_ascii=False)}")
        result = server_handle(msg["method"], msg["params"])
        print(f"[server→client] {json.dumps(result, ensure_ascii=False)}\n")
        return result

client = MCPClient("erp")
client.request("initialize")
tools = client.request("tools/list")["tools"]

print("🧰 모델이 처음 받는 것: 도구 이름·설명·인자뿐 (DB 주소도 비밀번호도 아님)")
for t in tools:
    print(f"  - {t['name']}: {t['description']} 인자={t['inputSchema']}")

# ── 3. 모델이 고른 도구 호출 (실제로는 LLM이 이 JSON을 만들어낸다)
call = {"name": "query_sales", "arguments": {"period": "2026-10", "region": "서울"}}
print(f"\n🤖 모델의 선택: {json.dumps(call, ensure_ascii=False)}")
answer = client.request("tools/call", call)["content"][0]["text"]
print(f"✅ 최종 답변 재료: {answer}")
```

**여기서 느껴야 하는 것**: 모델이 한 일은 *"어떤 도구를, 어떤 인자로 부를지 고르는 것"* 딱 하나입니다. SQL도, DB 커넥션도, 인증도 전부 **서버 코드 안에** 있습니다. 이 선이 있어야 "AI가 아무 테이블이나 긁어가지 않게" 막을 수 있고, AI 모델을 바꿔도 사내 연동 코드는 그대로 둘 수 있습니다.

---

## ⚠️ SA가 반드시 짚는 MCP의 함정 3가지

1. **서버 계정 권한이 곧 AI의 권한입니다.** MCP 서버를 DB 관리 자격증명으로 띄우면, AI가 실수할 때 피해도 관리자급이 됩니다.
2. **도구 설명은 공격 표면입니다.** 악성 문서에 심어진 문장이 모델을 속여 엉뚱한 도구를 부르게 할 수 있습니다(간접 프롬프트 인젝션). 도구 설명과 결과를 '데이터'로 취급해야 합니다.
3. **원격 서버는 인증이 생명입니다.** 사내 승인 목록(allowlist)에 등록된 서버만, OAuth 같은 표준 인증으로, 네트워크는 이그레스 제한과 함께.

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"MCP는 연동 개발을 줄이는 편의 기능이 아니라, 'AI가 무엇에 접근할 수 있는가'를 서버 단위로 통제할 수 있게 만드는 거버넌스 장치다. 그래서 SA는 MCP 서버 목록을 곧 권한 목록으로 관리한다."**
