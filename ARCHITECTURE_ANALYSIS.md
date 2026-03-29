# Phân Tích Kiến Trúc MiroFish

## 1. Tổng Quan

**MiroFish** là một nền tảng **Collective Intelligence Engine** (Động cơ Trí tuệ Tập thể) cho phép mô phỏng hành vi xã hội trên các nền tảng Twitter/Reddit thông qua multi-agent AI, từ đó tạo báo cáo phân tích dự đoán.

### Luồng Xử Lý Chính

```
Tải tài liệu (PDF/MD/TXT)
    ↓
Sinh Ontology (LLM phân tích → entity types + edge types)
    ↓
Xây dựng Knowledge Graph (Zep Cloud)
    ↓
Trích xuất Entities → Sinh Agent Profiles (LLM)
    ↓
Cấu hình & Chạy Mô phỏng (CAMEL OASIS - Twitter/Reddit)
    ↓
Sinh Báo cáo Phân tích (Report Agent + Tools)
    ↓
Tương tác Hỏi đáp với Agent
```

---

## 2. Kiến Trúc Tổng Thể

### 2.1 Mô Hình: Monolith với Service Layer

```
┌─────────────────────────────────────────────────┐
│                   Frontend                       │
│            Vue 3 + Vite + D3.js                 │
│              (Port 3000)                         │
├─────────────────────────────────────────────────┤
│                 REST API                         │
│           Axios → /api/* proxy                   │
├─────────────────────────────────────────────────┤
│                   Backend                        │
│              Flask (Port 5001)                   │
│  ┌───────────┬──────────────┬──────────────┐    │
│  │ graph_bp  │simulation_bp │  report_bp   │    │
│  └─────┬─────┴──────┬───────┴──────┬───────┘    │
│        │   Service Layer            │            │
│  ┌─────┴────────────┴──────────────┴───────┐    │
│  │ OntologyGenerator  │ GraphBuilder       │    │
│  │ SimulationManager  │ SimulationRunner   │    │
│  │ ProfileGenerator   │ ConfigGenerator    │    │
│  │ ReportAgent        │ ZepEntityReader    │    │
│  └─────────────────────────────────────────┘    │
│        │          Model Layer        │           │
│  ┌─────┴────────────────────────────┴──────┐    │
│  │ ProjectManager (File) │ TaskManager (RAM)│    │
│  │ SimulationManager     │ ReportManager   │    │
│  └─────────────────────────────────────────┘    │
├─────────────────────────────────────────────────┤
│              External Services                   │
│  ┌──────────────┐  ┌──────────┐  ┌───────────┐ │
│  │  Zep Cloud   │  │ LLM API  │  │ CAMEL AI  │ │
│  │ (Graph/RAG)  │  │ (OpenAI) │  │ (OASIS)   │ │
│  └──────────────┘  └──────────┘  └───────────┘ │
└─────────────────────────────────────────────────┘
```

### 2.2 Cấu Trúc Thư Mục

```
MiroFish/
├── backend/                        # Python/Flask backend
│   ├── app/
│   │   ├── __init__.py            # App factory (Flask)
│   │   ├── config.py              # Cấu hình tập trung
│   │   ├── api/                   # REST endpoints (3 blueprints)
│   │   │   ├── graph.py           # Ontology + Graph APIs
│   │   │   ├── simulation.py      # Simulation lifecycle APIs
│   │   │   └── report.py          # Report + Chat APIs
│   │   ├── models/                # Data models & persistence
│   │   │   ├── project.py         # Project CRUD (file-based)
│   │   │   └── task.py            # Task tracking (in-memory)
│   │   ├── services/              # Business logic (13 modules)
│   │   │   ├── ontology_generator.py
│   │   │   ├── graph_builder.py
│   │   │   ├── zep_entity_reader.py
│   │   │   ├── oasis_profile_generator.py
│   │   │   ├── simulation_config_generator.py
│   │   │   ├── simulation_manager.py
│   │   │   ├── simulation_runner.py
│   │   │   ├── simulation_ipc.py
│   │   │   ├── report_agent.py
│   │   │   ├── zep_tools.py
│   │   │   └── zep_graph_memory_updater.py
│   │   └── utils/                 # Tiện ích chung
│   │       ├── llm_client.py      # OpenAI-compatible wrapper
│   │       ├── file_parser.py     # PDF/MD/TXT extraction
│   │       ├── logger.py          # Logging
│   │       ├── retry.py           # Retry mechanism
│   │       └── zep_paging.py      # Pagination helper
│   ├── scripts/                   # Script mô phỏng độc lập
│   ├── run.py                     # Entry point
│   └── pyproject.toml             # Python dependencies (uv)
├── frontend/                       # Vue 3 + Vite frontend
│   ├── src/
│   │   ├── main.js                # Entry point
│   │   ├── App.vue                # Root component
│   │   ├── views/                 # 6 page components
│   │   │   ├── Home.vue
│   │   │   ├── MainView.vue       # 5-step wizard
│   │   │   ├── SimulationView.vue
│   │   │   ├── SimulationRunView.vue
│   │   │   ├── ReportView.vue
│   │   │   └── InteractionView.vue
│   │   ├── components/            # Reusable components
│   │   │   ├── Step1GraphBuild.vue
│   │   │   ├── Step2EnvSetup.vue
│   │   │   ├── Step3Simulation.vue
│   │   │   ├── Step4Report.vue
│   │   │   ├── Step5Interaction.vue
│   │   │   ├── GraphPanel.vue     # D3.js visualization
│   │   │   └── HistoryDatabase.vue
│   │   ├── api/                   # API client layer
│   │   ├── router/                # Vue Router
│   │   └── store/                 # State management
│   └── vite.config.js             # Vite config + proxy
├── Dockerfile                      # Multi-stage build
├── docker-compose.yml              # Orchestration
├── .github/workflows/              # CI/CD
└── .env.example                    # Env template
```

---

## 3. Technology Stack

| Tầng | Công nghệ | Mục đích |
|------|-----------|----------|
| **Frontend** | Vue 3 (Composition API) | UI framework |
| | Vite 7.2 | Build tool + dev server |
| | Vue Router 4 | Client-side routing |
| | Axios | HTTP client (5min timeout, retry) |
| | D3.js 7.9 | Knowledge graph visualization |
| **Backend** | Flask 3.0 | Web framework |
| | Python 3.11+ | Runtime |
| | uv | Package manager |
| | flask-cors | CORS middleware |
| **AI/ML** | OpenAI SDK | LLM integration (OpenAI-compatible) |
| | Zep Cloud 3.13 | Knowledge graph / GraphRAG |
| | CAMEL AI 0.2.78 | Multi-agent framework |
| | CAMEL OASIS 0.2.5 | Social simulation platform |
| **Infra** | Docker + Compose | Container deployment |
| | GitHub Actions | CI/CD (build & push GHCR) |

### LLM Configuration Mặc Định
- **Primary**: Alibaba Qwen (`qwen-plus` via DashScope)
- **Boost** (optional): Hỗ trợ model thứ 2 cho tác vụ nặng
- Tương thích với bất kỳ OpenAI-format API nào

---

## 4. API Endpoints Chi Tiết

### 4.1 Graph API (`/api/graph`)

| Method | Endpoint | Chức năng |
|--------|----------|----------|
| POST | `/upload` | Upload files (PDF/MD/TXT, max 50MB) |
| POST | `/extract-text` | Trích xuất & ghép text |
| POST | `/generate-ontology` | Sinh ontology bằng LLM (async) |
| POST | `/generate-ontology/status` | Kiểm tra tiến trình |
| POST | `/build-graph` | Xây dựng graph trên Zep (async) |
| POST | `/build-graph/status` | Kiểm tra tiến trình |
| GET | `/<graph_id>` | Lấy thống kê graph |
| GET | `/<graph_id>/entities` | Lấy danh sách entities |
| GET | `/<graph_id>/relations` | Lấy danh sách relations |
| GET | `/project/<id>` | Lấy thông tin project |
| GET | `/project/list` | Danh sách projects |
| DELETE | `/project/<id>` | Xóa project |

### 4.2 Simulation API (`/api/simulation`)

| Method | Endpoint | Chức năng |
|--------|----------|----------|
| POST | `/create` | Tạo simulation mới |
| POST | `/prepare` | Chuẩn bị (entities → profiles → config) |
| POST | `/prepare/status` | Kiểm tra tiến trình chuẩn bị |
| POST | `/start` | Khởi chạy mô phỏng |
| POST | `/stop` | Dừng mô phỏng |
| GET | `/<id>` | Lấy trạng thái simulation |
| GET | `/<id>/profiles` | Agent profiles (Twitter/Reddit) |
| GET | `/<id>/config` | Simulation config |
| GET | `/<id>/run-status` | Trạng thái real-time |
| GET | `/<id>/posts` | Bài đăng được tạo |
| GET | `/<id>/timeline` | Timeline theo round |
| GET | `/<id>/actions` | Lịch sử hành động agent |
| GET | `/<id>/agent-stats` | Thống kê per agent |
| POST | `/interview/batch` | Phỏng vấn agents |

### 4.3 Report API (`/api/report`)

| Method | Endpoint | Chức năng |
|--------|----------|----------|
| POST | `/generate` | Sinh báo cáo (async) |
| POST | `/generate/status` | Kiểm tra tiến trình |
| GET | `/<id>` | Lấy báo cáo |
| GET | `/<id>/download` | Tải Markdown |
| GET | `/<id>/sections` | Các section (streaming) |
| GET | `/<id>/progress` | Tiến trình real-time |
| POST | `/chat` | Chat với Report Agent |
| GET | `/<id>/agent-log` | Agent execution log |
| DELETE | `/<id>` | Xóa báo cáo |

---

## 5. Data Models & Persistence

### 5.1 Chiến Lược Lưu Trữ

MiroFish **không dùng database truyền thống**. Thay vào đó:

| Loại dữ liệu | Phương thức lưu | Đặc điểm |
|---------------|-----------------|----------|
| **Projects** | File JSON trên disk | Bền vững, CRUD đầy đủ |
| **Tasks** | In-memory (singleton) | Tạm thời, mất khi restart |
| **Simulations** | In-memory + file output | State tạm, kết quả bền |
| **Reports** | File JSON + Markdown | Bền vững |
| **Knowledge Graph** | Zep Cloud (external) | Bền vững, managed service |

### 5.2 Project Model

```python
Project:
  project_id: str         # "proj_xxx"
  name: str
  status: ProjectStatus   # CREATED → ONTOLOGY_GENERATED → GRAPH_BUILDING → GRAPH_COMPLETED | FAILED
  files: List[Dict]       # [{original_filename, size}]
  ontology: Dict           # {entity_types, edge_types}
  graph_id: str           # Zep graph identifier
  simulation_requirement: str
  chunk_size: int         # Default 500
  chunk_overlap: int      # Default 50

# Lưu tại: uploads/projects/{project_id}/project.json
```

### 5.3 Task Model

```python
Task:
  task_id: str            # UUID
  task_type: str          # graph_build | prepare_simulation | run_simulation | generate_report
  status: TaskStatus      # PENDING → PROCESSING → COMPLETED | FAILED
  progress: int           # 0-100
  message: str
  result: Optional[Dict]
  error: Optional[str]

# Lưu tại: In-memory (TaskManager singleton, thread-safe)
```

### 5.4 Simulation State

```python
SimulationState:
  simulation_id: str
  project_id: str
  graph_id: str
  status: str             # CREATED → PREPARING → READY → RUNNING → COMPLETED | STOPPED | FAILED
  enable_twitter: bool
  enable_reddit: bool
  entities_count: int
  profiles_count: int
  current_round: int
  config_generated: Dict  # LLM-generated params

# Lưu tại: In-memory (SimulationManager singleton)
```

### 5.5 Cấu Trúc File Storage

```
backend/uploads/
├── projects/
│   └── {project_id}/
│       ├── project.json         # Metadata
│       ├── extracted_text.txt   # Aggregated text
│       └── files/               # Uploaded files
│           └── {filename}
└── simulations/
    └── {simulation_id}/         # Kết quả mô phỏng
```

---

## 6. Service Layer - Chi Tiết

### 6.1 Pipeline Xử Lý Văn Bản

```
FileParser.extract_text()       # PDF (PyMuPDF) / MD / TXT
    ↓                           # Multi-encoding detection
TextProcessor.split_text()      # Chunk (500 chars, 50 overlap)
    ↓
OntologyGenerator.generate()    # LLM → 10 entity types + edge types
    ↓
GraphBuilder.build()            # Zep Cloud: create graph → set ontology → batch episodes
```

**OntologyGenerator** đặc biệt:
- Bắt buộc sinh đúng 10 entity types (8 chuyên biệt + Person + Organization)
- LLM phân tích nội dung + simulation requirement
- Output: `{entity_types[], edge_types[], analysis_summary}`

### 6.2 Graph Building (Zep Integration)

```python
GraphBuilderService:
  create_graph(name)           # Tạo graph mới trên Zep
  set_ontology(graph_id, ...)  # Dynamic Pydantic class creation
  add_text_batches(...)        # Batch episode ingestion + progress callback
  _wait_for_episodes(...)      # Poll Zep cho tới khi processing xong
  get_graph_data(graph_id)     # Truy xuất nodes + edges với pagination
```

**Điểm đáng chú ý**: Ontology được chuyển thành **Pydantic models động** (tạo class runtime) để Zep validate.

### 6.3 Simulation Pipeline

```
ZepEntityReader.read_entities()
    ↓ (filter by type)
OasisProfileGenerator.generate()
    ↓ (LLM enhance, parallel x3)
    ↓ Output: Twitter CSV + Reddit JSON
SimulationConfigGenerator.generate()
    ↓ (LLM reasoning → parameters)
    ↓ Output: rounds, duration, activity, events
SimulationRunner.start()
    ↓ (subprocess + IPC)
    ↓ Real-time: actions[], round_summaries[]
ZepGraphMemoryUpdater.update()
    ↓ Sync kết quả → Zep knowledge graph
```

**SimulationRunner** chạy mô phỏng trong **subprocess riêng** với IPC:
- Tránh block Flask main thread
- Signal handler cleanup khi shutdown
- Platform-specific action parsing (Twitter vs Reddit)

### 6.4 Report Agent

```python
ReportAgent:
  Tools:
    - insight_forge          # Phân tích insight
    - relationship_finder    # Tìm quan hệ
    - timeline_builder       # Xây timeline
    - search_graph           # Semantic search (Zep)
    - get_statistics         # Graph statistics

  Generation:
    - Sinh report section-by-section
    - Reflection mechanism (tối đa 2 rounds)
    - Tool call limit (configurable, default 5)
    - Streaming sections cho frontend
```

---

## 7. Frontend Architecture

### 7.1 Routing

```javascript
/                              → Home.vue (Upload + requirement)
/process/:projectId            → MainView.vue (5-step wizard)
/simulation/:simulationId      → SimulationView.vue
/simulation/:simulationId/start → SimulationRunView.vue
/report/:reportId              → ReportView.vue
/interaction/:reportId         → InteractionView.vue
```

### 7.2 State Management

**Không dùng Vuex/Pinia** - Approach nhẹ:
- **pendingUpload.js**: Reactive store đơn giản cho file upload state giữa các trang
- **Component local state**: Mỗi component tự quản lý reactive data
- **URL params**: project_id, simulation_id, report_id qua router props

### 7.3 API Client Layer

```javascript
// api/index.js - Axios instance
- Timeout: 5 phút (cho long-running operations)
- Request/Response interceptors
- requestWithRetry(): Exponential backoff (max 3 attempts)

// api/graph.js, simulation.js, report.js
- Tách module theo domain
- Mỗi function map 1:1 với backend endpoint
```

### 7.4 Graph Visualization

**GraphPanel.vue** sử dụng D3.js:
- Force-directed layout cho knowledge graph
- Interactive: zoom, pan, node click
- Color-coded theo entity type

---

## 8. Design Patterns

| Pattern | Vị trí | Mô tả |
|---------|--------|--------|
| **App Factory** | `app/__init__.py` | `create_app()` khởi tạo Flask |
| **Blueprint** | `app/api/` | 3 blueprint modules tách routing |
| **Singleton** | TaskManager, SimulationManager | In-memory state management |
| **Async Task + Polling** | Toàn bộ long-running ops | Trả task_id → client poll status |
| **Service Layer** | `app/services/` | Business logic tách khỏi API |
| **IPC (subprocess)** | SimulationRunner | Simulation chạy process riêng |
| **Callback** | GraphBuilder, ProfileGenerator | Progress callbacks cho async ops |
| **Retry + Backoff** | LLM calls, API client | Tự động retry khi lỗi |

---

## 9. Bảo Mật & Cấu Hình

### 9.1 Bảo Mật Hiện Tại

- **Không có authentication/authorization** - giả định mạng tin cậy
- CORS mở: `origins: "*"`
- File upload: chỉ cho phép PDF/MD/TXT, max 50MB
- Filename sanitization: UUID-based để chống path traversal
- API keys lưu trong `.env`, không commit vào git

### 9.2 Biến Môi Trường

```env
# Bắt buộc
LLM_API_KEY=              # API key cho LLM service
LLM_BASE_URL=             # OpenAI-compatible endpoint
LLM_MODEL_NAME=           # Model name (e.g., qwen-plus)
ZEP_API_KEY=              # Zep Cloud API key

# Tùy chọn
LLM_BOOST_API_KEY=        # Model thứ 2 cho tác vụ nặng
LLM_BOOST_BASE_URL=
LLM_BOOST_MODEL_NAME=
FLASK_HOST=0.0.0.0
FLASK_PORT=5001
FLASK_DEBUG=false
```

---

## 10. Deployment

### Docker (Khuyến nghị)

```yaml
# docker-compose.yml
services:
  mirofish:
    image: ghcr.io/666ghj/mirofish:latest
    env_file: .env
    ports:
      - "3000:3000"   # Frontend
      - "5001:5001"   # Backend
    volumes:
      - ./backend/uploads:/app/backend/uploads
    restart: unless-stopped
```

### CI/CD Pipeline

```
Git tag push / Manual dispatch
    ↓
GitHub Actions: docker-image.yml
    ↓
Build Docker image (Python 3.11 + Node.js 18)
    ↓
Push to GHCR: ghcr.io/666ghj/mirofish
```

---

## 11. Điểm Mạnh & Hạn Chế

### Điểm Mạnh

1. **Luồng xử lý rõ ràng**: 5 bước tuần tự, mỗi bước có service riêng
2. **LLM-Centric**: Mọi quyết định quan trọng đều qua LLM reasoning
3. **Async-First**: Tất cả tác vụ nặng chạy async, không block user
4. **Extensible**: Dễ thêm platform mới (ngoài Twitter/Reddit)
5. **Knowledge Graph**: Zep cung cấp context phong phú cho cả simulation và report
6. **Subprocess isolation**: Simulation chạy process riêng, an toàn

### Hạn Chế

1. **Không có authentication**: Cần bổ sung cho production
2. **In-memory state**: Tasks và Simulations mất khi restart
3. **Không có test coverage**: Chỉ có pytest config, chưa có test thực
4. **Single container**: Backend + Frontend cùng container, khó scale riêng
5. **File-based persistence**: Không phù hợp cho multi-instance/HA
6. **Không có WebSocket**: Dùng polling thay vì real-time push
7. **CORS mở hoàn toàn**: `origins: "*"` là rủi ro bảo mật

### Khuyến Nghị Cải Thiện

1. Thêm JWT/OAuth2 authentication
2. Chuyển task/simulation state sang Redis hoặc database
3. Bổ sung unit tests + integration tests + CI test pipeline
4. Tách frontend/backend thành containers riêng
5. Thêm WebSocket cho real-time updates thay vì polling
6. Cấu hình CORS chặt chẽ hơn
7. Thêm rate limiting cho API endpoints
