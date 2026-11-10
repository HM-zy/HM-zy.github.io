---
title: "[AI 기초 13] 엔터프라이즈 권한 격리와 보안: 멀티테넌시 RAG와 메타데이터 필터링"
date: 2026-11-20 08:00:00 +0900
categories: [AI, 기초스터디]
tags: [Security, RBAC, MultiTenancy, RAG, MetadataFiltering, Architecture, AI기초]
---

*"신입사원이 챗봇에게 '우리 회사 임원 연봉 얼마야?'라고 물었더니, 이사회 회의록 문서를 읽고 답변해 버렸습니다."*

사내 모든 문서를 하나의 벡터 DB에 털어 넣으면 발생하는 최악의 보안 사고입니다. 

실제 기업에서는 부서별, 직급별, 프로젝트별로 열람 권한이 엄격히 나뉘어 있습니다. AI 시스템에서도 이를 통제하는 **역할 기반 접근 제어(RBAC)**와 **메타데이터 사전 필터링(Pre-filtering)**을 알아보겠습니다.

---

## 🔒 검색 전 필터링(Pre-filtering) vs 검색 후 필터링(Post-filtering)

1. **사후 필터링 (Post-filtering) ❌ 위험**
   - 전체 문서에서 유사한 상위 5개를 먼저 뽑은 뒤, 사용자의 권한을 체크해 권한 없는 문서를 버리는 방식입니다.
   - 문제점: 상위 5개 중 4개가 임원용 문서라면, 일반 사용자는 볼 수 있는 문서가 1개밖에 안 남아 검색 품질이 망가집니다.
2. **사전 필터링 (Pre-filtering) ⭕ 표준**
   - 벡터 검색을 시작할 때 DB 쿼리 자체에 `WHERE department IN ('개발팀') AND security_level <= 2` 조건을 걸어, **애초에 사용자가 읽을 수 있는 문서 범위 안에서만 유사도를 계산**합니다.

---

## 💻 5분 실습: 메타데이터 필터링 쿼리 시뮬레이션

```python
# 사내 문서 벡터 DB 가상 데이터
document_store = [
    {"id": 1, "text": "취업규칙: 연차 휴가 규정", "allowed_roles": ["intern", "staff", "exec"]},
    {"id": 2, "text": "개발팀 가이드: AWS 접근 키 발급 절차", "allowed_roles": ["staff", "exec"]},
    {"id": 3, "text": "2026년 임원별 스톡옵션 배정표", "allowed_roles": ["exec"]}
]

def search_with_rbac(query, user_role):
    print(f"👤 사용자 권한: [{user_role}]의 질의: '{query}'")
    
    # 1. 사전 권한 필터링 (Pre-filtering)
    accessible_docs = [
        doc for doc in document_store 
        if user_role in doc["allowed_roles"]
    ]
    
    # 2. 권한 내 문서 중에서 검색
    print(f"🔍 열람 가능한 후보 문서: {len(accessible_docs)}건")
    for doc in accessible_docs:
        print(f"  - (열람 가능) {doc['text']}")

# 일반 사원과 임원의 검색 범위 비교
search_with_rbac("급여 및 보상 규정", user_role="staff")
print("-" * 40)
search_with_rbac("급여 및 보상 규정", user_role="exec")
```

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"엔터프라이즈 RAG 아키텍처에서 가장 먼저 묻는 질문은 '정확도가 몇 %인가'가 아니라 '사원과 임원의 데이터 격리가 어떻게 보장되는가'이다."**
