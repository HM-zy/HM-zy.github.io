---
title: "[AI 기초 10] 100줄 코드로 사내 시연용 RAG 웹 데모 배포하기 (Streamlit 실무)"
date: 2026-11-17 08:00:00 +0900
categories: [기초·개념]
tags: [Streamlit, RAG, WebUI, PoV, Prototyping, Architecture, AI기초, AI]
---

Solution Architect(SA)가 고객사 기술 검증(PoV) 미팅에 들어갈 때, 터미널 검은 화면에서 코드를 돌려 보여주면 C-Level 임원들의 마음을 사로잡기 어렵습니다.

*"그래서 우리 직원들이 실제로 쓰는 화면은 어떻게 생겼나요?"*

복잡한 프론트엔드(React, Vue)를 몰라도, 순수 파이썬 코드 100줄 내외로 깔끔한 사내 챗봇 웹 UI를 찍어낼 수 있는 도구가 바로 **스트림릿(Streamlit)**입니다.

---

## 🎨 SA 시연용 웹 데모에 꼭 들어가야 할 3가지 필수 요소

1. **대화 이력 유지 (Session State)**
   - 이전 질문과 답변이 휘발되지 않고 메신저처럼 위로 쌓여야 자연스럽습니다.
2. **실시간 스트리밍 출력 (`st.write_stream`)**
   - 3~5초 동안 멍하니 기다리게 하지 않고, 단어가 타이핑되듯 주르륵 흘러나오게 만들어야 체감 지연시간(Latency)이 짧게 느껴집니다.
3. **출처 접기/펼치기 아코디언 (`st.expander`)**
   - 답변 밑에 *"참고한 원문 문서 보기"*를 토글로 넣어, 임원들이 직접 원문 조항과 AI의 답변을 비교해 볼 수 있게 만듭니다.

---

## 💻 5분 실습: Streamlit RAG 챗봇 기본 뼈대 코드

```python
# app.py 로 저장 후 'streamlit run app.py' 실행
import streamlit as st
import time

st.set_page_config(page_title="사내 문서 Q&A AI 비서", page_icon="🤖")
st.title("🤖 사내 규정 Q&A 검색 시스템 (PoV Demo)")

# 1. 대화 세션 상태 초기화
if "messages" not in st.session_state:
    st.session_state.messages = []

# 2. 기존 대화 기록 화면 출력
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.write(msg["content"])
        if "source" in msg:
            with st.expander("📄 참고 원문 출처"):
                st.info(msg["source"])

# 3. 사용자 질문 입력
if prompt := st.chat_input("사내 규정에 대해 궁금한 점을 입력하세요"):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.write(prompt)

    # 가상 RAG 답변 생성 시뮬레이션
    sample_answer = f"'{prompt}'에 대한 사규 확인 결과, 취업규칙 제14조에 따라 유급 지원됩니다."
    sample_source = "취업규칙.pdf (페이지 14, 제3문단)"

    with st.chat_message("assistant"):
        message_placeholder = st.empty()
        full_response = ""
        for chunk in sample_answer.split():
            full_response += chunk + " "
            time.sleep(0.05)
            message_placeholder.markdown(full_response + "▌")
        message_placeholder.markdown(full_response)
        
        with st.expander("📄 참고 원문 출처"):
            st.info(sample_source)

    st.session_state.messages.append({
        "role": "assistant", 
        "content": full_response, 
        "source": sample_source
    })
```

---

## 🎯 Solution Architect 관점 한 줄 요약
> **"고객 PoV에서 가장 중요한 것은 완벽한 풀스택 웹이 아니라, 고객의 질문이 들어가서 신뢰할 수 있는 출처와 함께 답이 나오는 과정을 10분 안에 눈으로 증명하는 민첩성이다."**
