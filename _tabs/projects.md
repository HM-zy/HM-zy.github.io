---
title: Projects
icon: fas fa-diagram-project
order: 2
---

> **AI Solution Architect & Enterprise PoV Portfolio**  
> 글로벌 빅테크(OpenAI, Anthropic, AWS, Google Cloud) Solutions Architect 면접 대비 **'프로덕션 요구사항을 반영한 Enterprise PoV 솔루션'**을 단계적으로 구축하고 기록하는 공간입니다.

---

## 🏗️ 대표 프로젝트: Enterprise RAG & Agent PoV 솔루션 (3단계 진화)

단순한 시연용 데모에 그치지 않고, 기업 고객이 요구하는 **정량적 평가(Evals), 읽기 전용 보안 거버넌스(MCP), 비용·품질 실측 데이터, 임원용 비즈니스 제안서**를 A to Z로 완결하는 단 하나의 대표 솔루션입니다.

```
[1단계: 11월] 사내 문서 하이브리드 RAG + 20개 기준 질문셋 Evals 베이스라인
      ⬇️
[2단계: 12월] 읽기 전용 MCP DB 도구 + PII 마스킹 + 감사 로그 + 승인 워크플로우 안전 설계
      ⬇️
[3단계: 01월] 시맨틱 캐싱 비용·지연 실측 비교 리포트 + C-Level 1페이지 비즈니스 제안서
```

---

### [1단계 / 11월] RAG 코어 구축 & 조기 평가셋(Evals) 구축
- **진행 기간**: 2026.11 (11.30 1단계 릴리즈)
- **타깃 스택**: Python, LangChain / LlamaIndex, Chroma / Pinecone, Streamlit, Anthropic API
- **핵심 아키텍처**:
  - **하이브리드 검색**: 도메인 전문 용어 오인식을 막는 키워드(BM25) + 고차원 벡터 임베딩 결합
  - **출처 각주(Citations) 자동화**: 답변 생성 시 참고한 원문 파일명, 페이지 번호, 문단 메타데이터를 본문에 각주로 보존
  - **환각 방지 가드레일**: 관련 문맥이 없을 시 거짓 답변을 차단하고 신뢰할 수 있는 Fallback 제공
- **핵심 산출물 (PoV)**:
  - 🔗 GitHub 저장소: [HM-zy/enterprise-rag-solution](https://github.com/HM-zy)
  - 📊 **초기 20개 기준 질문셋(Golden Dataset) & 채점표(Evals)**: 정답 적중률, 환각 발생 여부 최초 베이스라인 측정 및 실패 사례 분석 리포트

---

### [2단계 / 12월] 읽기 전용 MCP 도구 연동 & 보안 안전 거버넌스
- **진행 기간**: 2026.12 (12.31 2단계 릴리즈)
- **타깃 스택**: Anthropic SDK, Model Context Protocol(MCP), Function Calling, LangGraph, NeMo Guardrails
- **핵심 아키텍처**:
  - **읽기 전용 MCP 연동**: 오픈 표준 C타입 단자 규격의 MCP 서버를 통해 사내 데이터베이스를 직접 쿼리하되, **읽기 전용(Read-only) 계정 및 허용 쿼리 화이트리스트**로 통제
  - **보안 거버넌스**: 민감정보(주민번호, 이메일, 연락처 등 PII) 실시간 마스킹 및 전 쿼리 감사 로그(Audit Log) 기록
  - **Human-in-the-loop**: 데이터 수정이나 결재 액션 발생 시 사람의 명시적 승인 후 실행되는 안전 워크플로우
- **핵심 산출물 (PoV)**:
  - 📄 보안 안전 거버넌스 설계서: `docs/security-governance.md`
  - 📊 1단계의 동일 20개 질문셋 재평가를 통한 도구 연동 후 성능 변화 비교 (Evals v2)

---

### [3단계 / 1월] 품질·지연·비용 실측 비교 & 1페이지 제안서 완성
- **진행 기간**: 2027.01 (01.31 3단계 완결)
- **타깃 스택**: LiteLLM, GPTCache, Prometheus, Grafana
- **핵심 아키텍처**:
  - **시맨틱 캐싱(Semantic Cache)**: 유사 질문 유입 시 LLM 재호출 없이 캐시 응답
  - **복잡도 기반 동적 라우팅**: 단순 요약은 경량 모델(Haiku/mini), 고난도 추론은 고성능 모델(Sonnet/o1) 분기
  - **토큰 메트릭 모니터링**: 전사 부서별 실시간 비용 집계 및 토큰 예산 한도 제어
- **핵심 산출물 (PoV)**:
  - 📊 **품질·지연·비용 실측 비교 리포트**: 동일 질문셋 기준 캐싱 및 라우팅 적용 전후의 실측 데이터 보고서
  - 📄 **임원용 1페이지 비즈니스 제안서(Executive Proposal)**:
    1. 고객 문제 (사내 규정 탐색 비효율 및 데이터 격리 요구)
    2. 기대 효과와 측정 방법 (실측된 검색 시간 단축 및 API 비용 절감치)
    3. 선택한 아키텍처 근거 (ADR: 기술 대안 비교 및 선택 이유)
    4. 주요 보안 위험 및 방어책 (읽기 전용 MCP, PII 마스킹, 승인 분기)
    5. 1차 소규모 파일럿의 다음 단계 로드맵

---

## 🛠️ 일일 연계 실습 아카이브 (Daily Hands-on PoVs)

매일 아침 스터디에서 학습한 핵심 개념을 100~200줄의 마이크로 코드로 직접 검증하고 위 대표 솔루션의 부품으로 조립한 기술 노트 모음입니다.

- **[실습 01]** 파이썬 코사인 유사도(Cosine Similarity) 계산 및 벡터 공간 거리 측정 원리
- **[실습 02]** 사내 매뉴얼 PDF 청킹(Chunking) 및 출처(Citation) 메타데이터 인덱싱
- **[실습 03]** JSON Schema 기반 Function Calling 도구 정의 및 파라미터 파싱
- **[실습 04]** LangGraph 기반 다단계 상태 머신 에이전트 루프 제어
