---
title: Projects
icon: fas fa-diagram-project
order: 2
---

> **AI Solution Architect & Enterprise PoV Portfolio**  
> 글로벌 빅테크(OpenAI, Anthropic, AWS, Google Cloud) Solutions Architect 면접 대비 실전 엔터프라이즈 AI 시스템 아키텍처 및 PoV 프로젝트를 기록하는 공간입니다.

---

## 🏗️ 3대 핵심 엔터프라이즈 토이프로젝트

기업 고객이 생성형 AI 도입 시 마주하는 3대 핵심 과제(지식 검색 정확도, 도구 자율성 및 보안 거버넌스, TCO 비용 절감)를 직접 해결한 실전 아키텍처 포트폴리오입니다.

### 01. 출처 기반 사내 문서 Q&A 시스템 (Enterprise RAG)
- **진행 기간**: 2026.11 (진행 중 / 11.30 릴리즈 목표)
- **타깃 스택**: Python, LangChain / LlamaIndex, Chroma / Pinecone, Streamlit, Anthropic API
- **핵심 아키텍처**:
  - **하이브리드 검색**: 도메인 전문 용어 오인식을 막는 키워드(BM25) + 고차원 벡터 임베딩 결합
  - **출처 각주(Citations) 자동화**: 답변 생성 시 참고한 원문 파일명, 페이지 번호, 문단 메타데이터를 본문에 각주로 보존
  - **환각 방지 가드레일**: 관련 문맥이 없을 시 거짓 답변을 차단하고 신뢰할 수 있는 Fallback 제공
- **산출물**:
  - 🔗 GitHub 저장소: [HM-zy/enterprise-rag-poc](https://github.com/HM-zy) (11월 말 배포 예정)
  - 📄 아키텍처 설계서(ADR): `docs/architecture-decisions.md`

---

### 02. MCP 기반 엔터프라이즈 자율 에이전트 (Autonomous Agent)
- **진행 기간**: 2026.12 (12.31 릴리즈 목표)
- **타깃 스택**: Anthropic SDK, Model Context Protocol(MCP), Function Calling, LangGraph
- **핵심 아키텍처**:
  - **표준 MCP 연동**: C타입 단자 규격의 오픈 표준 MCP 서버를 통해 사내 데이터베이스 및 외부 API 직접 연동
  - **안전 하네스(Governance Harness)**: 파괴적 SQL 쿼리 및 권한 밖 명령어 사전 샌드박싱 차단
  - **Human-in-the-loop**: 중요 결재/수정 액션 전 사람의 명시적 승인을 거치는 분기 워크플로우
- **산출물**:
  - 🔗 사내 DB 자율 조회 및 결재 에이전트 PoV 데모

---

### 03. FinOps 스마트 라우팅 게이트웨이 (Cost-Optimized AI Gateway)
- **진행 기간**: 2027.01 (01.31 릴리즈 목표)
- **타깃 스택**: LiteLLM, GPTCache, Prometheus, Grafana
- **핵심 아키텍처**:
  - **시맨틱 캐싱(Semantic Cache)**: 유사 질문 유입 시 LLM 재호출 없이 캐시 응답 (비용 100% 절감)
  - **복잡도 기반 동적 라우팅**: 단순 요약은 경량 모델(Haiku/mini), 고난도 추론은 고성능 모델(Sonnet/o1) 분기
  - **토큰 메트릭 모니터링**: 전사 부서별 실시간 비용 집계 및 토큰 예산 한도 제어
- **산출물**:
  - 🔗 기업 TCO 70%+ 절감 게이트웨이 아키텍처

---

## 🏆 실전 AI 공모전 & 해커톤 (Competitions)

제한된 일정과 비즈니스 제약 속에서 실제 엔터프라이즈 문제를 해결한 팀/개인 챌린지 기록입니다.

- **2026.12 ~ 2027.01 출전 타깃**:
  - 생성형 AI 비즈니스 응용 경진대회 (데이콘 / 초거대 AI 공모전)
  - 엔터프라이즈 RAG 및 에이전트 결합 PoV 출전 예정

---

## 🛠️ 일일 연계 실습 아카이브 (Daily Hands-on PoVs)

매일 아침 스터디에서 학습한 핵심 개념을 100~200줄의 마이크로 코드로 직접 검증한 기술 노트 모음입니다.

- **[실습 01]** 파이썬 코사인 유사도(Cosine Similarity) 계산 및 벡터 공간 거리 측정 원리
- **[실습 02]** 사내 매뉴얼 PDF 청킹(Chunking) 및 출처(Citation) 메타데이터 인덱싱
- **[실습 03]** JSON Schema 기반 Function Calling 도구 정의 및 파라미터 파싱
- **[실습 04]** LangGraph 기반 다단계 상태 머신 에이전트 루프 제어
