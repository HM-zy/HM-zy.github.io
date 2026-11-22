---
title: Projects
icon: fas fa-diagram-project
order: 2
---

> **AI Solution Architect Portfolio**  
> 글로벌 빅테크 Solutions Architect 면접 대비로 직접 만든 프로젝트와, 매일 아침 스터디에서 검증한 실습 코드를 기록하는 공간입니다. 성능 수치는 모두 저장소의 결과 파일(`evals/results/`)로 재현할 수 있습니다.

---

## 1. 대표 프로젝트: Red Hat Solution Sales Navigator — Claude RAG 어시스턴트 & Eval 하네스

Red Hat B2B 영업 현장의 질문("RHEL 8 지원 종료는 언제인가요?", "Satellite로 폐쇄망 패치는 어떻게 하나요?")에 **Red Hat 공식 문서 안에서만** 근거를 찾아 문장마다 출처 `[doc_id]`를 붙여 답하고, 근거가 없으면 답하지 않는 영업용 AI 어시스턴트입니다. 답이 실제로 맞는지는 정답을 적어 둔 골든셋으로 자동 채점합니다.

- 🔗 **GitHub 저장소**: [HM-zy/redhat-solution-sales-navigator](https://github.com/HM-zy/redhat-solution-sales-navigator)
- 📄 **설계·측정 글**: [공식 문서 RAG 인용 + 30문항 Evals로 영업용 AI 어시스턴트 만들기](/posts/redhat-sales-navigator-rag-evals/)
- 🧪 **구현기**: [63.6%에서 시작해 eval로 버그를 잡기까지](/posts/redhat-sales-navigator-rag-evals-build-log/)

### 무엇을 만들었나

- **프런트엔드 (React 18 · TypeScript · Vite)**: RHEL 디스커버리 8문항 → 세일즈 플레이(Upsell / Cross-sell) 7종 추천, 미팅 메모를 로컬 엑셀(`.xlsx`)로 저장, 화면 하단 RAG 질문 카드
- **RAG 파이프라인 (Python)**: Red Hat 공식 문서 14건 → **289청크**(11-17 v1·v2 측정) / **292청크**(11-19 v3 재수집 시점). 직접 구현한 **BM25**(영어 단어 + 한국어 2-gram, 영업 용어 별칭 맵, 버전·제품 부스트)
- **결정적 가드레일**: 생성 전 가격·범위 밖·근거 부족 차단, 생성 후 인용 누락·가짜 인용 검증 — 모델 판단에 맡기지 않고 코드 규칙으로 처리
- **로컬 API**: 표준 라이브러리 `http.server`만으로 `GET /api/health`, `POST /api/ask` (포트 8000), 웹은 `/api` 프록시로 호출
- **모델**: Claude Messages API (Claude Opus 5.5)

### 실측 결과 (라이브 Claude, 골든셋 30문항)

| 지표 | v1 | v2 (수정 후) |
|---|---|---|
| 문서 단위 Top-1 | 18/22 (81.8%) | 18/22 (81.8%) |
| Recall@5 | 21/22 (95.5%) | 22/22 (100.0%) |
| 거절 정확도 (거절해야 할 8문항) | 8/8 (100%) | 8/8 (100%) |
| 과잉 거절 (답할 수 있는 22문항 중) | 6건 (27.3%) | **2건 (9.1%)** |
| 인용 무결성 | 12/16 (75.0%) | **19/20 (95.0%)** |
| 핵심 사실 정확도 | 13/16 (81.2%) | 20/20 (100.0%) |

- **단위·API 테스트 49개** 통과 (`python3 -m unittest discover -s tests -t .`)
- v3에서 가격 가드레일 과잉 차단을 수정하고 회귀 확인 2문항을 더해 **골든셋 30 → 32문항**이 됐습니다(v3 수치는 32문항 기준이며 이전 버전과 1:1 비교하지 않습니다)

### 면접에서 쓰는 3단계 서사

1. **왜**: 추천의 근거를 물으면 보여 줄 문서가 없었고, 프롬프트를 고쳐도 좋아졌는지 판단할 기준이 없었다
2. **어떻게 고쳤나**: v0 Top-1 63.6% → v1 81.8% → v2 핵심 사실 100%. 인용 파싱 버그와 검색 누락을 eval이 잡아냈다
3. **한계**: 32문항 규모, 문항 초안은 Claude가 코퍼스 원문을 보고 작성해 사람 검토가 필요, 문단 단위 근거 지표는 아직 없음

---

## 2. 다음 단계 (진행 예정)

- 읽기 전용 MCP 도구 연동 + PII 마스킹 · 감사 로그 · 사람 승인(Human-in-the-loop) 워크플로우
- 시맨틱 캐싱 · 복잡도 기반 라우팅의 비용·지연 실측 비교와 임원용 1페이지 제안서
- 문단 단위 근거 지표와 held-out 검증 셋, 하이브리드 검색(BM25 + 임베딩)과 리랭커

---

## 3. 기초 실습 아카이브 — [HM-zy/AI-practice](https://github.com/HM-zy/AI-practice)

매일 아침 AI 스터디에서 개념을 직접 코드로 확인한 from-scratch 실습입니다. 위 RAG 파이프라인의 부품(청킹·유사도)은 여기서 출발했습니다.

- **[실습 01]** `cosine_similarity.py` — 코사인 유사도로 문장 벡터의 거리(각도) 재기
- **[실습 02]** `chunk_text.py` — 슬라이딩 윈도로 긴 문서를 청크로 나누고 겹침(overlap) 다루기
