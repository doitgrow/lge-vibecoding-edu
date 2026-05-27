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
├── Dockerfile             # 배포용 Docker 이미지
├── scripts/
│   └── coder-init.sh     # 컨테이너 초기화 스크립트 (ENTRYPOINT)
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

### 4. `Dockerfile`

```dockerfile
FROM python:3.12-slim

# 필수 시스템 패키지 설치
USER root
RUN apt-get update && apt-get install -y \
    git \
    curl \
    wget \
    ca-certificates \
    procps \
    bash \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# coder 사용자 생성 및 sudo 권한 부여
RUN useradd -m -s /bin/bash coder && \
    echo "coder ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    mkdir -p /workspaces && \
    chown -R coder:coder /workspaces

# 샘플 코드를 템플릿 위치에 복사 (PVC 마운트로 사라지지 않는 위치)
RUN mkdir -p /opt/workspace-template
COPY --chown=coder:coder . /opt/workspace-template/streamlit

# Python 패키지를 venv로 사전 빌드 후 tar로 패킹 (NFS 위 pip install 회피)
RUN python3 -m venv /workspaces/.venv/streamlit && \
    /workspaces/.venv/streamlit/bin/pip install --no-cache-dir -r /opt/workspace-template/streamlit/requirements.txt && \
    tar cf /opt/workspace-template/venv.tar -C /workspaces .venv/streamlit && \
    rm -rf /workspaces/.venv

# 이미지 태그 기록 (venv 버전 추적용)
ARG IMAGE_TAG=latest
RUN echo "${IMAGE_TAG}" > /opt/workspace-template/.image-tag

# code-server 사전 설치 (워크스페이스 시작 시간 단축)
RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --method=standalone --prefix=/tmp/code-server

# code-server 확장 사전 설치 (coder 유저로 실행)
USER coder
RUN /tmp/code-server/bin/code-server --install-extension ms-python.python
USER root

# Coder agent 초기화 스크립트 복사
ENV REPO_NAME=streamlit
COPY scripts/coder-init.sh /tmp/coder-init.sh
RUN chmod +x /tmp/coder-init.sh

USER coder
WORKDIR /workspaces

ENTRYPOINT ["/tmp/coder-init.sh"]
```

**설계 의도**:
- venv를 이미지 빌드 시 tar로 패킹 → 런타임에 NFS 위에서 pip install 안 하도록 최적화
- `/opt/workspace-template/`에 코드 보관 → PVC 마운트 시에도 손실 방지
- IMAGE_TAG를 기록하여 venv 버전 추적
- code-server + Python 확장 사전 설치로 워크스페이스 시작 시간 단축

---

### 5. `scripts/coder-init.sh`

```bash
#!/bin/bash
set -e

echo "=== Coder Agent Initialization ==="

# --- Section 1: Template copy ---
# 첫 실행 시 샘플 코드를 /workspaces로 복사
if [ ! -d "/workspaces/${REPO_NAME}" ]; then
    echo "Copying sample code to /workspaces/${REPO_NAME}..."
    cp -r /opt/workspace-template/${REPO_NAME} /workspaces/
    echo "Sample code copied successfully!"
fi

# --- Section 2: Virtualenv setup ---
setup_venv() {
  VENV_PATH="/workspaces/.venv/${REPO_NAME}"
  IMAGE_TAG_FILE="/opt/workspace-template/.image-tag"
  VENV_TAG_FILE="${VENV_PATH}/.image-tag"
  REQ_FILE="/opt/workspace-template/${REPO_NAME}/requirements.txt"
  
  echo "Setting up Python virtual environment..."
  
  # 3a. Check Python version compatibility
  if [ -d "$VENV_PATH" ]; then
    SYSTEM_PY_VERSION=$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')
    if [ -f "$VENV_PATH/bin/python3" ]; then
      VENV_PY_VERSION=$("$VENV_PATH/bin/python3" -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")' 2>/dev/null || echo "")
      if [ -n "$VENV_PY_VERSION" ] && [ "$VENV_PY_VERSION" != "$SYSTEM_PY_VERSION" ]; then
        echo "Python version changed (venv: $VENV_PY_VERSION, system: $SYSTEM_PY_VERSION), recreating venv..."
        rm -rf "$VENV_PATH"
      fi
    fi
  fi
  
  # 3b. Create venv if it doesn't exist
  if [ ! -d "$VENV_PATH" ]; then
    echo "Creating new virtual environment at $VENV_PATH..."
    mkdir -p "$(dirname "$VENV_PATH")"
    
    VENV_TAR="/opt/workspace-template/venv.tar"
    if [ -f "$VENV_TAR" ]; then
      echo "Extracting pre-built venv from image..."
      tar xf "$VENV_TAR" -C /workspaces || return 1
    else
      echo "No pre-built venv found, building from scratch..."
      python3 -m venv "$VENV_PATH" || return 1
      "$VENV_PATH/bin/pip" install --no-cache-dir -r "$REQ_FILE" || return 1
    fi
    
    # Copy image tag to venv
    if [ -f "$IMAGE_TAG_FILE" ]; then
      cp "$IMAGE_TAG_FILE" "$VENV_TAG_FILE" 2>/dev/null || echo "unknown" > "$VENV_TAG_FILE"
    else
      echo "unknown" > "$VENV_TAG_FILE"
    fi
    echo "Virtual environment created successfully!"
  else
    # 3c. Sync venv if image tag changed
    CURRENT_TAG="unknown"
    NEW_TAG="unknown"
    
    if [ -f "$VENV_TAG_FILE" ]; then
      CURRENT_TAG=$(cat "$VENV_TAG_FILE")
    fi
    
    if [ -f "$IMAGE_TAG_FILE" ]; then
      NEW_TAG=$(cat "$IMAGE_TAG_FILE")
    fi
    
    if [ "$CURRENT_TAG" != "$NEW_TAG" ]; then
      echo "Image tag changed (venv: $CURRENT_TAG, image: $NEW_TAG), syncing venv..."
      VENV_TAR="/opt/workspace-template/venv.tar"
      if [ -f "$VENV_TAR" ]; then
        echo "Replacing venv from pre-built tar..."
        rm -rf "$VENV_PATH"
        tar xf "$VENV_TAR" -C /workspaces || return 1
      else
        "$VENV_PATH/bin/pip" install --no-cache-dir -r "$REQ_FILE" || return 1
      fi
      
      # Update tag file
      if [ -f "$IMAGE_TAG_FILE" ]; then
        cp "$IMAGE_TAG_FILE" "$VENV_TAG_FILE" 2>/dev/null || echo "unknown" > "$VENV_TAG_FILE"
      else
        echo "unknown" > "$VENV_TAG_FILE"
      fi
      echo "Virtual environment synced successfully!"
    else
      echo "Venv is up to date (tag: $CURRENT_TAG), skipping sync"
    fi
  fi
  
  # 3d. Activate venv for this process
  export VIRTUAL_ENV="$VENV_PATH"
  export PATH="$VENV_PATH/bin:$PATH"
  echo "Virtual environment activated for current process"
  
  # 3e. Setup pip cache on PVC
  export PIP_CACHE_DIR="/workspaces/.pip-cache"
  mkdir -p "$PIP_CACHE_DIR"
  
  # 3f. Setup .bashrc for new terminal sessions
  WORKSPACES_BASHRC="/workspaces/.bashrc"
  
  # Create /workspaces/.bashrc if not exists
  if [ ! -f "$WORKSPACES_BASHRC" ]; then
    echo "# Coder workspace configuration" > "$WORKSPACES_BASHRC"
  fi
  
  # Check if this repo's venv config already exists in /workspaces/.bashrc
  if ! grep -q "VIRTUAL_ENV=\"/workspaces/.venv/${REPO_NAME}\"" "$WORKSPACES_BASHRC" 2>/dev/null; then
    echo "" >> "$WORKSPACES_BASHRC"
    echo "# Auto-activate venv for ${REPO_NAME}" >> "$WORKSPACES_BASHRC"
    echo "export VIRTUAL_ENV=\"/workspaces/.venv/${REPO_NAME}\"" >> "$WORKSPACES_BASHRC"
    echo "export PATH=\"/workspaces/.venv/${REPO_NAME}/bin:\$PATH\"" >> "$WORKSPACES_BASHRC"
    echo "export PIP_CACHE_DIR=\"/workspaces/.pip-cache\"" >> "$WORKSPACES_BASHRC"
  fi
  
  # Ensure ~/.bashrc sources /workspaces/.bashrc (for coder user)
  USER_BASHRC="/home/coder/.bashrc"
  if [ -f "$USER_BASHRC" ]; then
    if ! grep -q "source /workspaces/.bashrc" "$USER_BASHRC" 2>/dev/null && ! grep -q ". /workspaces/.bashrc" "$USER_BASHRC" 2>/dev/null; then
      echo "" >> "$USER_BASHRC"
      echo "# Source workspace configuration" >> "$USER_BASHRC"
      echo "if [ -f /workspaces/.bashrc ]; then" >> "$USER_BASHRC"
      echo "    source /workspaces/.bashrc" >> "$USER_BASHRC"
      echo "fi" >> "$USER_BASHRC"
    fi
  fi
  
  echo "Virtual environment setup complete!"
  return 0
}

# Call setup_venv with error handling
setup_venv || echo "WARNING: venv setup failed, continuing without venv..."

# --- Section 3: Environment variable checks ---
# 환경 변수 확인
if [ -z "$CODER_AGENT_URL" ]; then
    echo "ERROR: CODER_AGENT_URL not set"
    echo "Keeping container alive for debugging..."
    sleep infinity
fi

if [ -z "$CODER_AGENT_TOKEN" ]; then
    echo "ERROR: CODER_AGENT_TOKEN not set"
    echo "Keeping container alive for debugging..."
    sleep infinity
fi

# --- Section 4: Coder agent download and exec ---
# Coder agent 다운로드
BINARY_DIR="/tmp/coder"
BINARY_NAME="coder"
mkdir -p "$BINARY_DIR"
cd "$BINARY_DIR"

BINARY_URL="${CODER_AGENT_URL}/bin/coder-linux-amd64"
echo "Downloading Coder agent from: $BINARY_URL"

# 재시도 로직
MAX_RETRIES=5
RETRY_COUNT=0
while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if curl -fsSL "$BINARY_URL" -o "$BINARY_NAME"; then
        echo "Download successful!"
        break
    fi
    
    RETRY_COUNT=$((RETRY_COUNT + 1))
    if [ $RETRY_COUNT -lt $MAX_RETRIES ]; then
        echo "Download failed, retrying in 5 seconds... ($RETRY_COUNT/$MAX_RETRIES)"
        sleep 5
    else
        echo "ERROR: Failed to download after $MAX_RETRIES attempts"
        sleep infinity
    fi
done

chmod +x "$BINARY_NAME"

# Agent 실행
echo "Starting Coder agent..."
export CODER_AGENT_AUTH="token"
exec ./"$BINARY_NAME" agent
```

**설계 의도**:
- Section 1: 첫 실행 시만 코드 복사 (이미 있으면 건드리지 않음)
- Section 2: venv 관리 (Python 버전 호환성 체크, 이미지 태그 기반 동기화, .bashrc 자동 설정)
- Section 3: 필수 환경변수 없으면 `sleep infinity`로 디버깅 가능하게 유지
- Section 4: Coder agent 바이너리 다운로드 + 재시도 로직 + exec으로 PID 1 대체

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
