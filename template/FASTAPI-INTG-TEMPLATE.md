# FastAPI-Integrated 프로젝트 재현 템플릿

> 이 문서는 프로젝트를 처음부터 동일하게 재현하기 위한 완전한 참조 문서입니다.
> 모든 파일 내용을 그대로 복사하여 생성하면 됩니다.

---

## 프로젝트 개요

- **구조**: FastAPI 백엔드 + React(Vite+TypeScript) 프론트엔드 통합 템플릿
- **배포**: 단일 컨테이너에서 nginx(프론트 서빙 + API 리버스 프록시) + uvicorn(백엔드) 실행
- **LLM 연동**: LangChain + Azure OpenAI (환경변수 미설정 시 샘플 모드 동작)

---

## 디렉토리 구조

```
fastapi-integrated/
├── entrypoint.sh
├── nginx.conf
└── src/
    ├── backend/
    │   ├── .env.example
    │   ├── main.py
    │   ├── requirements.txt
    │   └── src/
    │       ├── api/
    │       │   └── v1/
    │       │       ├── endpoints/
    │       │       │   └── chat.py
    │       │       └── router.py
    │       ├── core/
    │       │   ├── config.py
    │       │   └── llm.py
    │       ├── prompts/
    │       │   ├── prompts.py
    │       │   └── templates.py
    │       ├── services/
    │       │   └── chat_service.py
    │       └── utils/
    │           └── logger.py
    └── frontend/
        ├── .gitignore
        ├── .modules-image-tag
        ├── eslint.config.js
        ├── index.html
        ├── package.json
        ├── public/
        │   └── vite.svg
        ├── src/
        │   ├── App.css
        │   ├── App.tsx
        │   ├── components/
        │   │   └── .gitkeep
        │   ├── hooks/
        │   │   ├── useApi.ts
        │   │   └── useHealthCheck.ts
        │   ├── index.css
        │   ├── main.tsx
        │   ├── types.ts
        │   └── vite-env.d.ts
        ├── tsconfig.app.json
        ├── tsconfig.json
        ├── tsconfig.node.json
        └── vite.config.ts
```

> **참고**: `__init__.py` 파일 없음. Python 패키지 구분 없이 직접 import 사용.

---

## 루트 파일

### `entrypoint.sh`

```bash
#!/bin/bash
set -e

# Start nginx in background (serves frontend + reverse proxies /api/ to uvicorn)
nginx

# Start uvicorn in foreground (becomes PID 1 replacement via exec)
exec uvicorn main:app --host 0.0.0.0 --port 8000
```

### `nginx.conf`

```nginx
user www-data;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;

    server {
        listen 80;
        server_name _;

        root /usr/share/nginx/html;
        index index.html;

        # Health check endpoint
        location /healthz {
            return 200 'ok';
            add_header Content-Type text/plain;
        }

        # API reverse proxy to backend on localhost:8000
        location /api/ {
            proxy_pass http://localhost:8000/api/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # SPA fallback - serve index.html for all unmatched routes
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

---

## Backend (`src/backend/`)

### `.env.example`

```env
# Application Settings
APP_ENV=local
DEBUG=True
API_V1_STR=/api/v1


# Azure OpenAI Settings
LLM_BASE_URL=https://llm.lge-vs.com
LLM_MODEL=gpt-4o
LLM_API_KEY=your_azure_api_key
```

### `requirements.txt`

```
fastapi==0.109.0
httpx==0.27.0
langchain==0.2.1
langchain-community==0.2.1
langchain-openai==0.1.7
langgraph==0.4.5
langserve[all]==0.2.1
langsmith==0.1.147
numpy==1.26.4
pydantic==2.7.4
pydantic-settings==2.4.0
python-dotenv==1.0.1
tiktoken==0.12.0
uvicorn[standard]==0.27.0
```

### `main.py`

```python
"""
==========================================================================
 FastAPI-Integrated 통합 템플릿 — 메인 엔트리포인트
==========================================================================

이 파일은 FastAPI 앱의 진입점입니다.
Dockerfile의 entrypoint.sh가 `uvicorn main:app --port 8000`으로 실행합니다.

[배포 규약]
- 이 파일의 `app` 변수가 ASGI 앱 객체여야 합니다.
- 포트 8000에서 실행됩니다 (nginx가 80 → 8000 프록시).
- /health 엔드포인트는 반드시 유지하세요 (K8s 헬스체크용).

[커스터마이징 가이드]
- 새 라우터를 추가하려면 src/api/v1/endpoints/ 에 파일을 만들고
  router.py에서 include_router()로 등록하세요.
- CORS 설정은 배포 환경에 맞게 allow_origins를 수정하세요.
==========================================================================
"""
import os
import uvicorn
from dotenv import load_dotenv

dotenv_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env")
load_dotenv(dotenv_path=dotenv_path)

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from src.core.config import settings
from src.api.v1.router import api_router
from src.utils.logger import get_logger

logger = get_logger(__name__)

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.PROJECT_NAME,
        openapi_url=f"{settings.API_V1_STR}/openapi.json",
        debug=settings.DEBUG
    )

    # ── CORS 설정 ──────────────────────────────────────────
    # 개발 중에는 "*"로 설정하고, 운영 배포 시에는 허용할 도메인만 지정하세요.
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # ── 라우터 등록 ────────────────────────────────────────
    # src/api/v1/router.py에서 정의한 라우터를 여기서 등록합니다.
    # 새 엔드포인트를 추가할 때마다 router.py에 include_router()를 추가하세요.
    app.include_router(api_router, prefix=settings.API_V1_STR)

    # ── 헬스체크 (필수) ────────────────────────────────────
    # K8s liveness/readiness probe가 이 엔드포인트를 호출합니다.
    # 절대 삭제하지 마세요!
    @app.get("/health")
    def health_check():
        return {"status": "ok", "env": settings.APP_ENV}

    return app

app = create_app()

if __name__ == "__main__":
    # 로컬 개발 시 직접 실행: python main.py
    logger.info(f"서버를 8000번 포트에서 시작합니다. (Debug={settings.DEBUG})")
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### `src/core/config.py`

```python
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    PROJECT_NAME: str = "Sample AI Agent"
    APP_ENV: str = os.getenv("APP_ENV", "local")  # 애플리케이션 실행 환경 (local, dev, prod 등)
    DEBUG: bool = os.getenv("DEBUG", "False").lower() == "true"  # 디버그 모드 활성화 여부
    API_V1_STR: str = "/api/v1"  # API 버전 접두사

    # LLM 설정
    LLM_BASE_URL: str = os.getenv("LLM_BASE_URL", "https://llm.lge-vs.com")  # LLM API 기본 주소
    LLM_MODEL: str = os.getenv("LLM_MODEL", "gpt-4o")  # 사용할 모델 이름
    LLM_API_KEY: str = os.getenv("LLM_API_KEY", "")  # LLM API 키

    class Config:
        env_file = ".env"  # .env 파일에서 설정 로드
        case_sensitive = True

settings = Settings()
```

### `src/core/llm.py`

```python
from langchain_openai import ChatOpenAI
from src.core.config import settings

def get_llm():
    """
    설정에 따라 구성된 ChatOpenAI 인스턴스를 반환합니다.
    """
    return ChatOpenAI(
        openai_api_base=settings.LLM_BASE_URL,
        model=settings.LLM_MODEL,
        openai_api_key =settings.LLM_API_KEY,
        temperature=0.7
    )
```

### `src/prompts/prompts.py`

```python
"""
==========================================================================
 프롬프트 텍스트 관리
==========================================================================

LLM에 전달할 프롬프트 텍스트를 이 파일에서 관리합니다.
templates.py에서 이 텍스트를 PromptTemplate으로 변환합니다.

[커스터마이징 가이드]
- {question} 같은 변수를 사용하여 동적 입력을 받을 수 있습니다.
- 여러 프롬프트를 정의하고 용도별로 분리하세요.
==========================================================================
"""

# ── 기본 채팅 프롬프트 ─────────────────────────────────────
# {question} 자리에 사용자 입력이 들어갑니다.
BASIC_CHAT_PROMPT = """
당신은 도움이 되는 AI 어시스턴트입니다.
사용자의 질문에 정확하고 친절하게 답변해주세요.

사용자 질문: {question}
"""
```

### `src/prompts/templates.py`

```python
"""
==========================================================================
 프롬프트 템플릿 생성
==========================================================================

prompts.py에서 정의한 텍스트를 LangChain PromptTemplate으로 변환합니다.
서비스(services/)에서 이 템플릿을 import하여 체인에 사용합니다.
==========================================================================
"""
from langchain_core.prompts import PromptTemplate
from src.prompts.prompts import BASIC_CHAT_PROMPT

# prompts.py의 텍스트를 템플릿으로 변환
CHAT_PROMPT = PromptTemplate.from_template(BASIC_CHAT_PROMPT)

# ── 새 템플릿 추가 예시 ────────────────────────────────────
# from src.prompts.prompts import SUMMARY_PROMPT
# SUMMARY_PROMPT_TEMPLATE = PromptTemplate.from_template(SUMMARY_PROMPT)
```

### `src/services/chat_service.py`

```python
"""
==========================================================================
 채팅 서비스 — 비즈니스 로직
==========================================================================

엔드포인트(endpoints/)에서 호출하는 비즈니스 로직을 여기에 구현합니다.

[커스터마이징 가이드]
- LLM 체인을 구성하려면 src/core/llm.py의 get_llm()을 사용하세요.
- 프롬프트는 src/prompts/ 에서 관리하세요.
- 새 서비스 파일을 만들어도 됩니다 (예: summary_service.py).

[LLM 연동]
.env에 LLM_API_KEY가 설정되어 있으면 LLM 체인을 사용합니다.
미설정 시 샘플 응답을 반환합니다.
==========================================================================
"""
from langchain_core.output_parsers import StrOutputParser
from src.utils.logger import get_logger

logger = get_logger(__name__)


class ChatService:
    """
    채팅 서비스 클래스.
    LLM_API_KEY가 설정되어 있으면 LLM 체인을 사용하고,
    미설정 시 샘플 응답을 반환합니다.
    """

    def __init__(self):
        self.chain = None
        try:
            from src.core.llm import get_llm
            from src.prompts.templates import CHAT_PROMPT
            llm = get_llm()
            self.chain = CHAT_PROMPT | llm | StrOutputParser()
            logger.info("ChatService: LLM 체인이 초기화되었습니다.")
        except Exception as e:
            logger.warning(f"ChatService: LLM 초기화 건너뜀 (API 키 미설정 시 정상): {e}")
            logger.info("LLM을 사용하려면 .env에 LLM_API_KEY를 설정하세요. (샘플 모드로 실행)")

    async def get_response(self, question: str) -> str:
        """
        사용자의 질문을 처리하고 응답을 반환합니다.
        LLM 체인이 초기화되었으면 LLM 응답을, 아니면 샘플 응답을 반환합니다.
        """
        if self.chain:
            try:
                result = self.chain.invoke({"question": question})
                return result.strip()
            except Exception as e:
                logger.error(f"LLM 호출 실패: {e}")
                return f"[LLM 오류] {e}"

        logger.info(f"샘플 모드 — 질문: {question}")
        return f"[샘플 응답] '{question}' — LLM을 연동하려면 .env에 LLM_API_KEY를 설정하세요."


# 서비스 인스턴스 (엔드포인트에서 import하여 사용)
chat_service = ChatService()
```

### `src/api/v1/router.py`

```python
"""
==========================================================================
 API 라우터 등록
==========================================================================

새 엔드포인트 모듈을 만들면 여기서 include_router()로 등록합니다.

[추가 방법]
1. src/api/v1/endpoints/ 에 새 파일 생성 (예: summary.py)
2. 해당 파일에서 router = APIRouter() 생성
3. 아래에 include_router(summary.router, prefix="/summary", tags=["summary"]) 추가
==========================================================================
"""
from fastapi import APIRouter
from src.api.v1.endpoints import chat

api_router = APIRouter()

# ── 채팅 엔드포인트 ─────────────────────────────────────────
api_router.include_router(chat.router, prefix="/chat", tags=["chat"])

# ── 여기에 새 라우터를 추가하세요 ─────────────────────────────
# 예시:
# from src.api.v1.endpoints import summary
# api_router.include_router(summary.router, prefix="/summary", tags=["summary"])
```

### `src/api/v1/endpoints/chat.py`

```python
"""
==========================================================================
 채팅 엔드포인트
==========================================================================

POST /api/v1/chat/ 엔드포인트입니다.
비즈니스 로직은 services/chat_service.py에 구현되어 있습니다.

[배포 규약]
- 프론트엔드에서 상대경로로 호출: fetch('api/v1/chat/', ...)
- nginx가 /api/ 요청을 백엔드(localhost:8000)로 프록시합니다.

[커스터마이징 가이드]
- 요청/응답 스키마를 Pydantic BaseModel로 정의하세요.
- 비즈니스 로직은 services/ 폴더에 분리하세요.
- LLM을 사용하려면 .env에 LLM_API_KEY를 설정하세요.
==========================================================================
"""
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from src.services.chat_service import chat_service

router = APIRouter()


# ── 요청/응답 스키마 ──────────────────────────────────────
# 필요에 따라 필드를 추가하세요.
class ChatRequest(BaseModel):
    query: str  # 사용자 입력 메시지


class ChatResponse(BaseModel):
    response: str  # AI 응답 메시지


# ── 엔드포인트 ────────────────────────────────────────────
@router.post("/", response_model=ChatResponse)
async def chat_endpoint(request: ChatRequest):
    """
    채팅 엔드포인트.
    .env에 LLM_API_KEY가 설정되면 LLM 응답, 미설정 시 샘플 응답을 반환합니다.
    """
    result = await chat_service.get_response(request.query)
    return ChatResponse(response=result)
```

### `src/utils/logger.py`

```python
import logging
import sys

def get_logger(name: str):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)

    # 중복 핸들러 방지
    if not logger.handlers:
        handler = logging.StreamHandler(sys.stdout)
        handler.setFormatter(logging.Formatter(
            "[%(asctime)s] %(levelname)s in %(module)s: %(message)s"
        ))
        logger.addHandler(handler)

    return logger
```

---

## Frontend (`src/frontend/`)

### `package.json`

```json
{
  "name": "fastapi-frontend",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@eslint/js": "^9.13.0",
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@vitejs/plugin-react": "^4.3.3",
    "eslint": "^9.13.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.14",
    "globals": "^15.11.0",
    "typescript": "~5.6.2",
    "typescript-eslint": "^8.11.0",
    "vite": "^5.4.10"
  }
}
```

### `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + React + TS</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### `vite.config.ts`

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  base: "./",
  server: {
    proxy: {
      "/api": "http://localhost:8000",
      "/health": "http://localhost:8000",
    },
  },
});
```

### `tsconfig.json`

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

### `tsconfig.app.json`

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

### `tsconfig.node.json`

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "ES2022",
    "lib": ["ES2023"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["vite.config.ts"]
}
```

### `eslint.config.js`

```javascript
import js from "@eslint/js";
import globals from "globals";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";
import tseslint from "typescript-eslint";

export default tseslint.config(
  { ignores: ["dist"] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ["**/*.{ts,tsx}"],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
    plugins: {
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      "react-refresh/only-export-components": [
        "warn",
        { allowConstantExport: true },
      ],
    },
  },
);
```

### `.gitignore`

```
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

node_modules
dist
dist-ssr
*.local

# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
.DS_Store
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?
```

### `.modules-image-tag`

```
mode-integrated
```

### `src/main.tsx`

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.tsx";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

### `src/App.tsx`

```tsx
/**
 * ==========================================================================
 *  메인 앱 컴포넌트
 * ==========================================================================
 *
 * 프론트엔드의 루트 컴포넌트입니다.
 *
 * [커스터마이징 가이드]
 * - 새 컴포넌트를 src/components/ 에 만들고 여기서 배치하세요.
 * - 페이지 레이아웃을 자유롭게 수정하세요.
 * - 스타일은 App.css에서 관리합니다.
 *
 * [배포 규약]
 * - TypeScript 에러가 있으면 npm run build가 실패하고 배포가 중단됩니다.
 * - 반드시 에러 없이 빌드되는 상태를 유지하세요.
 * ==========================================================================
 */
import { useState } from "react";
import { useHealthCheck } from "./hooks/useHealthCheck";
import { useApi } from "./hooks/useApi";
import type { ConnectionStatus } from "./types";
import "./App.css";

/** 백엔드 연결 상태 표시 컴포넌트 */
function HealthIndicator({ status }: { status: ConnectionStatus }) {
  const label =
    status === "connected"
      ? "연결됨"
      : status === "disconnected"
        ? "연결 끊김"
        : "확인 중…";
  return (
    <div className={`health-indicator health-indicator--${status}`}>
      <span className="health-dot" />
      <span className="health-label">{label}</span>
    </div>
  );
}

export default function App() {
  const healthStatus = useHealthCheck(5000);
  const { state, sendChat } = useApi();
  const [query, setQuery] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!query.trim() || state.loading) return;
    await sendChat({ query: query.trim() });
  };

  return (
    <div className="app">
      {/* ── 헤더 ─────────────────────────────────────────── */}
      <header className="app-header">
        <div className="app-header-inner">
          <div className="app-brand">
            <span className="app-logo" aria-hidden="true">
              ⚡
            </span>
            <h1 className="app-title">FastAPI Integrated</h1>
            <span className="app-version">Template</span>
          </div>
          <HealthIndicator status={healthStatus} />
        </div>
      </header>

      {/* ── 메인 콘텐츠 ─────────────────────────────────── */}
      <main className="app-main">
        <div className="app-main-inner">
          {/* 채팅 폼 (샘플) */}
          <section className="api-section">
            <div className="api-section-header">
              <h2 className="api-section-title">Chat API 샘플</h2>
              <span className="api-section-base">POST /api/v1/chat/</span>
            </div>

            <div className="endpoint-card">
              <form className="endpoint-form" onSubmit={handleSubmit}>
                <div className="form-group">
                  <label className="form-label" htmlFor="chat-input">
                    메시지 입력
                  </label>
                  <textarea
                    id="chat-input"
                    className="form-textarea"
                    value={query}
                    onChange={e => setQuery(e.target.value)}
                    placeholder="메시지를 입력하세요..."
                    rows={3}
                    disabled={state.loading}
                  />
                </div>
                <button
                  type="submit"
                  className="send-button"
                  disabled={state.loading || !query.trim()}>
                  {state.loading ? "전송 중…" : "전송"}
                </button>
              </form>

              {/* 응답 표시 */}
              {state.error && (
                <div className="response-section">
                  <div className="response-error">
                    <span className="response-error-icon">⚠</span>
                    <span>{state.error}</span>
                  </div>
                </div>
              )}
              {state.data && (
                <div className="response-section">
                  <div className="response-header">
                    <span className="response-title">응답</span>
                  </div>
                  <pre className="json-display">
                    {JSON.stringify(state.data, null, 2)}
                  </pre>
                </div>
              )}
            </div>
          </section>

          {/* ── 여기에 새 컴포넌트를 추가하세요 ──────────── */}
          {/*
            예시:
            <section className="api-section">
              <MyNewComponent />
            </section>
          */}
        </div>
      </main>

      {/* ── 푸터 ─────────────────────────────────────────── */}
      <footer className="app-footer">
        <span>
          FastAPI Integrated Template — 이 코드를 수정하여 서비스를 만드세요
        </span>
      </footer>
    </div>
  );
}
```

### `src/types.ts`

```typescript
/**
 * ==========================================================================
 *  타입 정의
 * ==========================================================================
 *
 * 프론트엔드에서 사용하는 TypeScript 타입을 여기서 관리합니다.
 *
 * [커스터마이징 가이드]
 * - 새 API 엔드포인트를 추가할 때마다 요청/응답 타입을 여기에 정의하세요.
 * - 백엔드의 Pydantic 스키마와 일치시키세요.
 * ==========================================================================
 */

/** POST /api/v1/chat/ 요청 바디 */
export interface ChatRequest {
  query: string;
}

/** POST /api/v1/chat/ 응답 바디 */
export interface ChatResponse {
  response: string;
}

/** GET /health 응답 */
export interface HealthResponse {
  status: string;
  env: string;
}

/** 백엔드 연결 상태 */
export type ConnectionStatus = "connected" | "disconnected" | "checking";
```

### `src/hooks/useApi.ts`

```typescript
/**
 * ==========================================================================
 *  API 호출 훅
 * ==========================================================================
 *
 * 백엔드 API를 호출하는 커스텀 훅입니다.
 *
 * [배포 규약 — 중요!]
 * - fetch URL은 반드시 상대경로 + '/api/' 접두사를 사용하세요.
 *   ✅ fetch('api/v1/chat/', ...)
 *   ❌ fetch('http://localhost:8000/api/v1/chat/', ...)
 * - nginx가 /api/ 요청을 같은 컨테이너의 백엔드(localhost:8000)로 프록시합니다.
 *
 * [커스터마이징 가이드]
 * - 새 API 엔드포인트를 추가할 때 이 훅을 참고하여 새 훅을 만드세요.
 * - 또는 이 훅에 새 함수를 추가해도 됩니다.
 * ==========================================================================
 */
import { useState, useCallback } from "react";
import type { ChatRequest, ChatResponse } from "../types";

interface ApiState {
  loading: boolean;
  error: string | null;
  data: ChatResponse | null;
}

export function useApi() {
  const [state, setState] = useState<ApiState>({
    loading: false,
    error: null,
    data: null,
  });

  const sendChat = useCallback(async (request: ChatRequest) => {
    setState({ loading: true, error: null, data: null });

    try {
      // ⚠️ 상대경로 필수! 'api/v1/chat/' (앞에 / 없음)
      const res = await fetch("api/v1/chat/", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(request),
      });

      const parsed = await res.json();

      if (res.ok) {
        setState({ loading: false, error: null, data: parsed as ChatResponse });
      } else {
        setState({
          loading: false,
          error: parsed.detail || `Error ${res.status}`,
          data: null,
        });
      }
    } catch (err) {
      setState({
        loading: false,
        error: err instanceof Error ? err.message : "Network error",
        data: null,
      });
    }
  }, []);

  return { state, sendChat };
}
```

### `src/hooks/useHealthCheck.ts`

```typescript
/**
 * ==========================================================================
 *  헬스체크 훅
 * ==========================================================================
 *
 * 백엔드 /health 엔드포인트를 주기적으로 호출하여 연결 상태를 확인합니다.
 *
 * [배포 규약]
 * - fetch URL은 반드시 상대경로를 사용하세요: 'health' (NOT 'http://localhost:8000/health')
 * - nginx가 /health 요청을 백엔드로 프록시합니다.
 * ==========================================================================
 */
import { useState, useEffect, useRef } from "react";
import type { ConnectionStatus } from "../types";

export function useHealthCheck(intervalMs = 5000): ConnectionStatus {
  const [status, setStatus] = useState<ConnectionStatus>("checking");
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => {
    const check = async () => {
      try {
        const res = await fetch("health", {
          signal: AbortSignal.timeout(4000),
        });
        setStatus(res.ok ? "connected" : "disconnected");
      } catch {
        setStatus("disconnected");
      }
    };

    check();
    timerRef.current = setInterval(check, intervalMs);
    return () => {
      if (timerRef.current !== null) clearInterval(timerRef.current);
    };
  }, [intervalMs]);

  return status;
}
```

### `src/index.css`

```css
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html,
body {
  min-height: 100vh;
  min-width: 320px;
}

body {
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#root {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}
```

### `src/App.css`

```css
/* ─── Design tokens ─────────────────────────────────────────── */
:root {
  --bg-base: #0d0f12;
  --bg-surface: #141720;
  --bg-elevated: #1c2030;
  --bg-input: #111318;
  --bg-hover: #1f2436;

  --border-subtle: #272c3a;
  --border-default: #333a50;
  --border-focus: #4d7cfe;

  --text-primary: #e2e6f0;
  --text-secondary: #8892a8;
  --text-muted: #555e74;
  --text-code: #cdd7f0;

  --accent: #4d7cfe;
  --accent-dim: #2a4aaa;
  --accent-glow: rgba(77, 124, 254, 0.18);

  --green: #2dd882;
  --green-dim: rgba(45, 216, 130, 0.15);
  --orange: #f5a623;
  --orange-dim: rgba(245, 166, 35, 0.15);
  --red: #f04d5a;
  --red-dim: rgba(240, 77, 90, 0.15);

  --font-ui: "Outfit", "DM Sans", ui-sans-serif, system-ui, sans-serif;
  --font-mono:
    "JetBrains Mono", "Fira Code", ui-monospace, "Cascadia Code", monospace;

  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  --spacing-1: 4px;
  --spacing-2: 8px;
  --spacing-3: 12px;
  --spacing-4: 16px;
  --spacing-5: 20px;
  --spacing-6: 24px;
  --spacing-8: 32px;

  --transition: 150ms ease;
}

/* ─── Reset / base ──────────────────────────────────────────── */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  font-size: 14px;
}

body {
  font-family: var(--font-ui);
  background: var(--bg-base);
  color: var(--text-primary);
  line-height: 1.6;
  min-height: 100vh;
  -webkit-font-smoothing: antialiased;
}

#root {
  display: contents;
}

/* ─── App shell ─────────────────────────────────────────────── */
.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* ─── Header ────────────────────────────────────────────────── */
.app-header {
  position: sticky;
  top: 0;
  z-index: 100;
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border-subtle);
  backdrop-filter: blur(12px);
}

.app-header-inner {
  max-width: 900px;
  margin: 0 auto;
  padding: var(--spacing-3) var(--spacing-6);
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-4);
}

.app-brand {
  display: flex;
  align-items: center;
  gap: var(--spacing-2);
}

.app-logo {
  font-size: 1.25rem;
  line-height: 1;
}

.app-title {
  font-size: 1.05rem;
  font-weight: 700;
  letter-spacing: -0.01em;
  color: var(--text-primary);
}

.app-version {
  font-size: 0.7rem;
  font-family: var(--font-mono);
  color: var(--text-muted);
  background: var(--bg-elevated);
  border: 1px solid var(--border-subtle);
  padding: 1px 6px;
  border-radius: var(--radius-sm);
}

/* ─── Health indicator ──────────────────────────────────────── */
.health-indicator {
  display: flex;
  align-items: center;
  gap: var(--spacing-2);
  font-size: 0.78rem;
  font-weight: 500;
  color: var(--text-secondary);
  padding: var(--spacing-1) var(--spacing-3);
  border-radius: var(--radius-md);
  border: 1px solid var(--border-subtle);
  background: var(--bg-elevated);
  transition:
    border-color var(--transition),
    background var(--transition);
}

.health-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--text-muted);
  transition:
    background var(--transition),
    box-shadow var(--transition);
}

.health-indicator--connected .health-dot {
  background: var(--green);
  box-shadow: 0 0 6px var(--green);
}
.health-indicator--connected .health-label {
  color: var(--green);
}

.health-indicator--disconnected .health-dot {
  background: var(--red);
  box-shadow: 0 0 6px var(--red);
}
.health-indicator--disconnected .health-label {
  color: var(--red);
}

.health-indicator--checking .health-dot {
  background: var(--orange);
  animation: pulse 1.2s ease infinite;
}
.health-indicator--checking .health-label {
  color: var(--orange);
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.35;
  }
}

/* ─── Main layout ───────────────────────────────────────────── */
.app-main {
  flex: 1;
  padding: var(--spacing-8) var(--spacing-6);
}

.app-main-inner {
  max-width: 900px;
  margin: 0 auto;
}

/* ─── API section ───────────────────────────────────────────── */
.api-section-header {
  display: flex;
  align-items: baseline;
  gap: var(--spacing-3);
  margin-bottom: var(--spacing-5);
}

.api-section-title {
  font-size: 0.7rem;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: var(--text-muted);
}

.api-section-base {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: var(--text-muted);
}

/* ─── Endpoint card ─────────────────────────────────────────── */
.endpoint-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

/* ─── Form ──────────────────────────────────────────────────── */
.endpoint-form {
  padding: var(--spacing-5);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-4);
  border-bottom: 1px solid var(--border-subtle);
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-2);
}

.form-label {
  display: flex;
  align-items: center;
  gap: var(--spacing-2);
  font-size: 0.78rem;
  font-weight: 600;
  color: var(--text-secondary);
  font-family: var(--font-mono);
}

.form-textarea {
  font-family: var(--font-mono);
  font-size: 0.85rem;
  line-height: 1.6;
  color: var(--text-code);
  background: var(--bg-input);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  padding: var(--spacing-3) var(--spacing-4);
  resize: vertical;
  min-height: 100px;
  width: 100%;
  transition:
    border-color var(--transition),
    box-shadow var(--transition);
  outline: none;
}

.form-textarea::placeholder {
  color: var(--text-muted);
}

.form-textarea:focus {
  border-color: var(--border-focus);
  box-shadow: 0 0 0 3px var(--accent-glow);
}

.form-textarea:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

/* ─── Send button ───────────────────────────────────────────── */
.send-button {
  align-self: flex-start;
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-2);
  padding: var(--spacing-2) var(--spacing-5);
  font-family: var(--font-ui);
  font-size: 0.82rem;
  font-weight: 600;
  color: #fff;
  background: var(--accent);
  border: none;
  border-radius: var(--radius-md);
  cursor: pointer;
  transition:
    background var(--transition),
    opacity var(--transition),
    transform 80ms ease;
}

.send-button:hover:not(:disabled) {
  background: #5d8dff;
}

.send-button:active:not(:disabled) {
  transform: scale(0.97);
}

.send-button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

/* ─── Response section ──────────────────────────────────────── */
.response-section {
  padding: var(--spacing-5);
  border-bottom: 1px solid var(--border-subtle);
}

.response-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--spacing-3);
}

.response-title {
  font-size: 0.7rem;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: var(--text-muted);
}

.response-error {
  display: flex;
  align-items: flex-start;
  gap: var(--spacing-3);
  background: var(--red-dim);
  border: 1px solid rgba(240, 77, 90, 0.3);
  border-radius: var(--radius-md);
  padding: var(--spacing-3) var(--spacing-4);
  color: var(--red);
  font-size: 0.82rem;
  line-height: 1.5;
}

.response-error-icon {
  flex-shrink: 0;
  font-size: 1rem;
  margin-top: 1px;
}

/* ─── JSON display ──────────────────────────────────────────── */
.json-display {
  font-family: var(--font-mono);
  font-size: 0.82rem;
  line-height: 1.7;
  color: var(--text-code);
  background: var(--bg-input);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  padding: var(--spacing-4);
  overflow-x: auto;
  tab-size: 2;
  white-space: pre;
}

/* ─── Footer ────────────────────────────────────────────────── */
.app-footer {
  padding: var(--spacing-4) var(--spacing-6);
  border-top: 1px solid var(--border-subtle);
  text-align: center;
  font-size: 0.7rem;
  color: var(--text-muted);
  font-family: var(--font-mono);
}

/* ─── Scrollbar ─────────────────────────────────────────────── */
::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background: var(--border-default);
  border-radius: 3px;
}
::-webkit-scrollbar-thumb:hover {
  background: var(--text-muted);
}

/* ─── Selection ─────────────────────────────────────────────── */
::selection {
  background: var(--accent-glow);
  color: var(--text-primary);
}
```

### `src/vite-env.d.ts`

```typescript
/// <reference types="vite/client" />
```

### `src/components/.gitkeep`

빈 파일 (디렉토리 유지용)

### `public/vite.svg`

Vite 기본 로고 SVG 파일 (기본 생성 파일 그대로 사용)

---

## 재현 절차

```bash
# 1. 루트 디렉토리 생성
mkdir -p fastapi-integrated

# 2. 루트 파일 생성
# entrypoint.sh, nginx.conf 생성 후:
chmod +x fastapi-integrated/entrypoint.sh

# 3. 백엔드 구조 생성
mkdir -p fastapi-integrated/src/backend/src/{api/v1/endpoints,core,prompts,services,utils}
# 각 파일 생성 (위 내용 참고)
# .env.example을 복사하여 .env 생성

# 4. 프론트엔드 구조 생성
mkdir -p fastapi-integrated/src/frontend/{public,src/{components,hooks}}
# 각 파일 생성 (위 내용 참고)
touch fastapi-integrated/src/frontend/src/components/.gitkeep

# 5. 프론트엔드 의존성 설치
cd fastapi-integrated/src/frontend
npm install

# 6. 백엔드 의존성 설치
cd ../backend
pip install -r requirements.txt

# 7. 환경변수 설정
cp .env.example .env
# .env에서 LLM_API_KEY 설정 (선택)

# 8. 로컬 실행
# 터미널 1 - 백엔드:
cd src/backend && python main.py

# 터미널 2 - 프론트엔드:
cd src/frontend && npm run dev
```

---

## 핵심 아키텍처 규칙

1. **단일 컨테이너 배포**: nginx(포트 80) + uvicorn(포트 8000) 동시 실행
2. **API 프록시**: nginx가 `/api/` 요청을 `localhost:8000`으로 프록시
3. **프론트엔드 fetch**: 반드시 상대경로 사용 (`'api/v1/chat/'`, `'health'` — 앞에 `/` 없음)
4. **헬스체크**: `/health` 엔드포인트 필수 유지 (K8s probe용)
5. **LLM 연동**: `LLM_API_KEY` 환경변수 미설정 시 자동으로 샘플 모드 동작
6. **`__init__.py` 없음**: Python 패키지 마커 파일 미사용
