# Streamlit LLM Chat App - Project Template

이 문서는 프로젝트를 처음부터 동일하게 재현하기 위한 전체 명세서입니다.

---

## 프로젝트 개요

- **목적**: OpenAI 호환 API를 사용하는 Streamlit 기반 채팅 앱
- **배포 환경**: Docker 컨테이너 → Coder 워크스페이스 (Kubernetes)
- **언어**: Python 3.12
- **프레임워크**: Streamlit
- **LLM 클라이언트**: OpenAI Python SDK (LangChain 사용하지 않음, requirements.txt에만 존재)

---

## 디렉토리 구조

```
streamlit/
├── app.py                 # 메인 앱 (UI + 채팅 로직 + 부트스트랩)
├── requirements.txt       # Python 의존성
├── .env.example           # 환경변수 예시 파일
└── AGENTS.md             # AI 가이드 (별도 관리)
```

---

## 파일별 상세 명세

### 1. `app.py` (142줄)

단일 파일에 모든 로직이 들어있다. 크게 3개 섹션으로 구성:

#### 섹션 1: 부트스트랩 블록 (1~27줄)

`python app.py` 실행 시 자동으로 `streamlit run`으로 전환하는 코드. `STREAMLIT_RUN` 환경변수로 무한 재귀 방지.

```python
import os
import subprocess
import sys

import streamlit as st
from dotenv import load_dotenv
from openai import OpenAI

# python app.py 실행 시 자동으로 streamlit run으로 재실행하는 부트스트랩 코드
# STREAMLIT_RUN 환경변수로 무한 재귀 실행 방지
if __name__ == "__main__" and not os.environ.get("STREAMLIT_RUN"):
    app_dir = os.path.dirname(os.path.abspath(__file__))
    venv_python = os.path.join(app_dir, ".venv", "bin", "python")
    python = venv_python if os.path.isfile(venv_python) else sys.executable

    env = {**os.environ, "STREAMLIT_RUN": "1"}

    subprocess.run([
        python, "-m", "streamlit", "run", os.path.abspath(__file__),
        "--server.port=8501",
        "--server.headless=true",
        "--server.address=0.0.0.0",
        "--server.enableCORS=false",
        "--server.enableXsrfProtection=false",
        "--browser.gatherUsageStats=false",
    ], env=env)
    sys.exit(0)
```

**핵심 사항**:
- venv가 있으면 venv python 사용, 없으면 시스템 python 사용
- 포트 8501 고정 (인프라와 연동)
- headless=true, CORS 비활성화 (프록시 환경)

#### 섹션 2: 환경변수 로드 및 설정 (29~50줄)

```python
# ── 메인 앱 시작 ─────────────────────────────────────────────────────────────

# .env 파일에서 환경 변수 로드 (LLM_BASE_URL, LLM_API_KEY, LLM_MODEL)
load_dotenv(os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env"))

LLM_BASE_URL = os.getenv("LLM_BASE_URL")
LLM_API_KEY  = os.getenv("LLM_API_KEY")
LLM_MODEL    = os.getenv("LLM_MODEL", "gpt-5.1-chat")

MODELS = [
    "gpt-4o-mini",
    "gpt-4.1",
    "gpt-4o",
    "claude-3-sonnet",
    "claude-3-5-sonnet",
    "nova-pro",
    "gpt-5.1-chat",
]

# ── 페이지 기본 설정 ──────────────────────────────────────────────────────────

st.set_page_config(page_title="LLM Settings Demo", page_icon="🤖", layout="wide")
st.title("🤖 LLM Settings Demo")
```

**핵심 사항**:
- `LLM_MODEL`의 기본값은 `"gpt-5.1-chat"`
- MODELS 리스트는 사이드바 드롭다운에 사용
- `layout="wide"` 사용

#### 섹션 3: 사이드바 + 채팅 UI (53~142줄)

**사이드바 구성**:
- Model 선택 (selectbox, MODELS 리스트 기반)
- Temperature 슬라이더 (0.0~2.0, 기본값 0.7, step 0.1)
- Max Tokens 슬라이더 (100~4096, 기본값 1024, step 100)
- System Prompt 텍스트 영역 (기본값: "You are a helpful AI assistant.", height=150)
- divider 후 Base URL 표시, API Key 마스킹 표시
- Clear Chat History 버튼

**채팅 UI 구성**:
- `st.session_state.messages` 리스트로 대화 기록 관리
- 이전 대화 기록을 `st.chat_message`로 렌더링
- API Key 미설정 시 경고 메시지
- `st.chat_input`으로 사용자 입력 받기
- 스트리밍 응답: `st.empty()` placeholder에 청크 단위로 출력, 커서 표시(`▌`)
- system prompt + 전체 대화 기록을 messages로 전달
- 에러 시 `❌ Error: {e}` 형태로 표시

```python
# ── 사이드바: LLM 파라미터 설정 ───────────────────────────────────────────────

with st.sidebar:
    st.header("⚙️ LLM Settings")

    # 사용할 모델 선택
    model = st.selectbox(
        "Model",
        options=MODELS,
        index=MODELS.index(LLM_MODEL) if LLM_MODEL in MODELS else 0,
    )

    # 창의성 조절 (0=일관성, 2=창의적)
    temperature = st.slider("Temperature", min_value=0.0, max_value=2.0, value=0.7, step=0.1,
                            help="값이 높을수록 더 창의적인 응답을 생성합니다.")

    # 응답 최대 길이 설정
    max_tokens = st.slider("Max Tokens", min_value=100, max_value=4096, value=1024, step=100,
                           help="생성할 최대 토큰 수를 설정합니다.")

    # 모델 역할과 행동 방식 정의
    system_prompt = st.text_area("System Prompt", value="You are a helpful AI assistant.", height=150,
                                 help="모델의 역할과 행동을 정의합니다.")

    st.divider()
    st.caption(f"**Base URL:** {LLM_BASE_URL}")
    st.caption("**API Key:** " + "*" * 20)

    # 대화 기록 초기화
    if st.button("🔄 Clear Chat History"):
        st.session_state.messages = []
        st.rerun()

# ── 세션 상태: 대화 기록 초기화 ───────────────────────────────────────────────

# session_state는 새로고침 없이 상태를 유지하는 Streamlit의 핵심 기능
if "messages" not in st.session_state:
    st.session_state.messages = []

# ── 채팅 UI ───────────────────────────────────────────────────────────────────

# 이전 대화 기록 렌더링
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# API Key 미설정 시 경고, 정상이면 입력창 표시
api_key_valid = bool(LLM_API_KEY and LLM_API_KEY != "your_azure_api_key")

if not api_key_valid:
    st.warning("⚠️ .env 파일에서 LLM_API_KEY를 설정해주세요.")
elif prompt := st.chat_input("메시지를 입력하세요..."):
    # 사용자 메시지 저장 및 표시
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    # AI 응답 생성 (스트리밍)
    with st.chat_message("assistant"):
        placeholder = st.empty()
        full_response = ""

        try:
            client = OpenAI(base_url=LLM_BASE_URL, api_key=LLM_API_KEY)

            # system 메시지 + 전체 대화 기록을 함께 전달 (컨텍스트 유지)
            messages = [{"role": "system", "content": system_prompt}] + st.session_state.messages

            stream = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens,
                stream=True,
            )

            # 청크 단위로 수신하며 실시간으로 화면에 출력
            for chunk in stream:
                if chunk.choices[0].delta.content:
                    full_response += chunk.choices[0].delta.content
                    placeholder.markdown(full_response + "▌")

            placeholder.markdown(full_response)

        except Exception as e:
            full_response = f"❌ Error: {e}"
            placeholder.markdown(full_response)

    # AI 응답 저장 (다음 턴의 컨텍스트로 활용)
    st.session_state.messages.append({"role": "assistant", "content": full_response})
```

---

### 2. `requirements.txt`

```
streamlit==1.42.2
openai==1.68.2
langchain==0.2.1
langchain-openai==0.1.7
python-dotenv==1.0.1
```

> 참고: langchain, langchain-openai는 코드에서 직접 사용하지 않지만 requirements에 포함되어 있음.

---

### 3. `.env.example`

```
# Application Settings
APP_ENV=local
DEBUG=True
API_V1_STR=/api/v1


# Azure OpenAI Settings
LLM_BASE_URL=https://llm.lge-vs.com
LLM_MODEL=gpt-4o
LLM_API_KEY=your_azure_api_key
```

---

## 아키텍처 요약

```
[사용자 브라우저]
      │
      ▼ (port 8501)
[Streamlit App (app.py)]
      │
      ▼ (OpenAI SDK, streaming)
[LLM API Server (LLM_BASE_URL)]
```

- 단일 파일 구조 (`app.py`)
- 상태 관리: `st.session_state.messages` (리스트 of `{"role": ..., "content": ...}`)
- LLM 호출: OpenAI SDK 직접 사용 (LangChain 미사용)
- 스트리밍: `stream=True` + 청크별 placeholder 업데이트
- 배포: Docker → Coder 워크스페이스 (K8s PVC 마운트 환경)

---

## 재현 시 주의사항

1. **부트스트랩 블록은 반드시 `app.py` 최상단 import 직후에 위치**해야 함 (Streamlit import 후, 앱 로직 전)
2. **포트 8501 고정** - 인프라(Dockerfile, K8s)와 연동되어 변경 금지
3. **환경변수는 절대 하드코딩 금지** - `os.getenv()`로만 접근
4. **MODELS 리스트와 LLM_MODEL 기본값**이 일치해야 selectbox index 계산이 정상 동작
5. **스트리밍 커서 `▌`** 문자를 사용하여 타이핑 효과 구현
6. **에러 핸들링**: try/except로 감싸고 에러 메시지도 assistant 응답으로 저장
7. **API Key 검증**: `"your_azure_api_key"` 문자열과의 비교로 미설정 감지
