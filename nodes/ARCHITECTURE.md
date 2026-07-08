# LightRAG 架构说明

> 本文档描述 LightRAG 的整体架构、模块划分、核心数据流与关键设计。
> 目录/模块层面的权威规范以仓库根目录的 `AGENTS.md` 为准。

LightRAG 是一个基于**图结构知识表示**的检索增强生成(RAG)框架。它从文档中抽取实体与关系、构建知识图谱(KG),并支持 `local` / `global` / `hybrid` / `mix` / `naive` 多种检索模式,把图谱检索与向量检索结合起来回答查询。

---

## 1. 架构总览

LightRAG 在逻辑上分为五层,自底向上依次为:存储层、核心引擎层、解析与抽取层、API 服务层、前端层。

```
┌──────────────────────────────────────────────────────────────┐
│  前端层        lightrag_webui/  (React 19 + TS + Vite + Tailwind)  │
└───────────────────────────────┬──────────────────────────────┘
                                │  HTTP / SSE
┌───────────────────────────────▼──────────────────────────────┐
│  API 服务层    lightrag/api/  (FastAPI + Gunicorn)              │
│   ├─ document_routes  (上传/扫描/删除/状态)                       │
│   ├─ query_routes     (查询/流式)                                │
│   ├─ graph_routes     (知识图谱可视化)                            │
│   └─ ollama_api       (Ollama 兼容接口)                          │
└───────────────────────────────┬──────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────┐
│  核心引擎层    lightrag/lightrag.py  LightRAG (主类,由 mixin 组装) │
│   ├─ _PipelineMixin          文档摄入管线                        │
│   ├─ _StorageMigrationMixin  存储数据迁移                        │
│   ├─ _RoleLLMMixin           角色化 LLM 配置                     │
│   └─ operate.py              实体/关系抽取 + 多模式检索           │
└───────────────────────────────┬──────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐
│ 解析与抽取层     │  │ LLM / 嵌入层      │  │ 存储层  lightrag/kg/│
│ parser/ chunker/│  │ llm/ llm_roles.py │  │ (KV/Vector/Graph/   │
│                 │  │                   │  │  DocStatus)         │
└─────────────────┘  └──────────────────┘  └────────────────────┘
```

---

## 2. 顶层目录结构

| 目录 / 文件 | 作用 |
|---|---|
| `lightrag/` | 核心 Python 包,后端全部逻辑 |
| `lightrag_webui/` | React 19 + TypeScript 前端(Bun + Vite + Tailwind) |
| `scripts/` | `test.sh` 测试运行器、`setup/` 交互式环境配置向导、`release/` 发布工具 |
| `tests/` | pytest 测试套件,子目录镜像 `lightrag/` 的功能模块 |
| `docs/` | 部署、配置、功能文档(多数有中英双语 `-zh` 版本) |
| `examples/` | 各 LLM/存储后端的示例脚本 |
| `prompts/` | 自定义提示词模板与样例 |
| `k8s-deploy/` | Kubernetes 部署清单(含数据库依赖) |
| `reproduce/` | 复现论文/基准评估的步骤脚本 |
| `env.example` | 环境变量模板(服务器、存储、查询、rerank、认证) |
| `docker-compose*.yml` / `Dockerfile*` | 容器化部署(完整版 / 精简版 / Postgres 版) |
| `AGENTS.md` | 仓库开发规范(权威来源) |

---

## 3. 核心包 `lightrag/` 模块详解

### 3.1 主类 `LightRAG`(`lightrag.py`)

`LightRAG` 是整个框架的入口与编排者。它由多个聚焦的 mixin 组装而成(此前是单一巨型类,现已拆分):

```python
@final
class LightRAG(_RoleLLMMixin, _StorageMigrationMixin, _PipelineMixin):
    ...
```

mixin 解析顺序:`LightRAG → _RoleLLMMixin → _StorageMigrationMixin → _PipelineMixin → object`。

- `@final` 装饰器被保留 —— mixin 分层是**内部实现细节**,不是外部可继承的扩展点。公开 API 不变。
- 跨越多个流程、或依赖 prompt-profile 状态的方法留在 `LightRAG` 自身:`ainsert_custom_kg`、`_insert_done`、`_process_extract_entities`、`_refresh_addon_params_cache`、`addon_params` 访问器。
- 公开导出(`lightrag/__init__.py`,惰性加载):`LightRAG`、`QueryParam`、`RoleLLMConfig`、`RoleSpec`、`ROLES`。

> **关键约定**:实例化后**必须**调用 `await rag.initialize_storages()` 初始化存储后端,否则会报 `AttributeError: __aenter__` 或 `KeyError: 'history_messages'`。使用结束后调用 `await rag.finalize_storages()` 清理。

各 mixin 的职责:

| Mixin / 模块 | 文件 | 职责 |
|---|---|---|
| `_PipelineMixin` | `pipeline.py` | 文档摄入管线:入队、处理、错误重试、解析器调度(`parse_native` / `parse_mineru` / `parse_docling`)、多模态分析、worker 脚手架 |
| `_StorageMigrationMixin` | `storage_migrations.py` | `check_and_migrate_data`、实体关系数据迁移、chunk 跟踪存储迁移 |
| `_RoleLLMMixin` | `llm_roles.py` | `RoleSpec`/`RoleLLMConfig`/`ROLES` 注册表,角色归一化、builder 注册、wrapper 重建、运行时配置更新 |
| — | `operate.py` | 核心抽取与查询操作:实体/关系抽取、分块、多模式检索逻辑 |
| — | `utils_pipeline.py` | 管线用纯辅助函数:文档身份(源 key、内容哈希)、解析产物路径、多模态实体增强等 |
| — | `addon_params.py` | `ObservableAddonParams` + `default_addon_params` / `normalize_addon_params` |
| — | `base.py` | 存储后端抽象基类与 `QueryParam`/`TextChunkSchema` |

### 3.2 存储层(`lightrag/kg/`)

LightRAG 使用 **4 类存储**,各自有可插拔的后端:

| 存储类型 | 存什么 | 抽象基类(`base.py`) |
|---|---|---|
| **KV_STORAGE** | LLM 响应缓存、文本 chunk、文档信息 | `BaseKVStorage` |
| **VECTOR_STORAGE** | 实体/关系/chunk 的向量嵌入 | `BaseVectorStorage` |
| **GRAPH_STORAGE** | 实体-关系图结构 | `BaseGraphStorage` |
| **DOC_STATUS_STORAGE** | 文档处理状态跟踪 | `BaseDocStatusStorage` |

所有存储后端继承自 `StorageNameSpace`,约定统一的 `initialize()` / `finalize()` / `index_done_callback()` / `drop()` 生命周期接口。

**后端实现**(`kg/*_impl.py`):

- 键值:`json_kv_impl`、`postgres_impl`、`mongo_impl`、`redis_impl`
- 向量:`nano_vector_db_impl`、`faiss_impl`、`milvus_impl`、`qdrant_impl`、`postgres_impl`、`opensearch_impl`
- 图:`networkx_impl`、`neo4j_impl`、`memgraph_impl`、`postgres_impl`、`mongo_impl`
- 文档状态:`json_doc_status_impl`、`postgres_impl`、`mongo_impl`、`redis_impl`

后端注册表 `STORAGE_IMPLEMENTATIONS` / `STORAGES` 位于 `kg/__init__.py`;`kg/factory.py::get_storage_class()` 按配置名解析具体类。

**Workspace 数据隔离**:每个 `LightRAG` 实例可传 `workspace` 参数做数据隔离,实现方式因后端而异——文件型用 `working_dir` 子目录;集合型用 collection 名前缀;关系型 DB 用 workspace 列过滤;Qdrant 用 payload 分区。

`kg/shared_storage.py` 维护管线并发用的共享状态(`pipeline_status` 等跨 workspace 共享 dict)。

### 3.3 LLM 与嵌入层(`lightrag/llm/` + `llm_roles.py`)

- **Provider 绑定**:每个 `llm/<provider>.py` 提供异步的 LLM 调用与嵌入函数,均带缓存支持。支持的 provider:OpenAI、Ollama、Azure、Gemini、Bedrock、Anthropic、HuggingFace、Jina、NVIDIA、VoyageAI、智谱(zhipu)、LMDeploy、Lollms、llama_index。
- **角色化配置**(`llm_roles.py`):`ROLES` 注册表 + `RoleSpec`/`RoleLLMConfig` 定义不同角色(如抽取、摘要、问答)使用不同模型。角色相关行为应集中在本模块,而不是散落到各 provider。

> **嵌入函数包装**:自定义嵌入函数用 `@wrap_embedding_func_with_attrs` 装饰;对已装饰函数再次包装时需访问 `.func` 取底层函数,不能重复包装。**切换嵌入模型时必须清空数据目录**(可保留 `kv_store_llm_response_cache.json`),否则旧向量与新模型空间不匹配。

### 3.4 解析与分块

**解析层(`lightrag/parser/`)** —— 统一解析入口:

- `routing.py`:根据引擎与文件名提示,在 `legacy` / `native` / `mineru` / `docling` 四条流程间路由。
- `docx/`:原生 DOCX 格式解析器(子包)。
- `external/`:基于 HTTP 的外部适配器(`mineru`、`docling`),共享助手在 `external/_common.py`、`_manifest.py`、`_zip.py`。
- `legacy/`、`markdown/`:传统与 Markdown 解析。
- `debug.py` + `cli.py`:`python -m lightrag.parser.cli` 的离线调试入口(不需启动完整 LightRAG)。

**分块策略(`lightrag/chunker/`)**:

| 文件 | 策略 |
|---|---|
| `token_size.py` | 按 token 大小切分 |
| `recursive_character.py` | 递归字符切分 |
| `semantic_vector.py` | 语义向量切分 |
| `paragraph_semantic.py` | 段落语义切分 |

### 3.5 API 服务层(`lightrag/api/`)

- `lightrag_server.py`:FastAPI 应用入口,挂载 REST 路由与 Ollama 兼容 API。
- `routers/`:`document_routes`(上传/扫描/删除/状态)、`query_routes`(查询/流式)、`graph_routes`(知识图谱可视化)、`ollama_api`(Ollama 兼容)。
- `auth.py` / `passwords.py`:API Key 与账户认证。
- `gunicorn_config.py` / `run_with_gunicorn.py`:Gunicorn 多 worker 启动器。
- `webui/`:打包好的 WebUI 静态产物;`static/`:Swagger 资源。

启动方式:`lightrag-server`(生产)、`uvicorn lightrag.api.lightrag_server:app --reload`(开发)、`lightrag-gunicorn`(多 worker)。

---

## 4. 核心数据流

### 4.1 文档插入流(`ainsert`)

```
文本/文件
   │
   ▼
apipeline_enqueue_documents        写入 doc_status(PENDING),入队
   │
   ▼
解析 (parser/routing)              legacy / native / mineru / docling
   │
   ▼
分块 (chunker)                     token / recursive / semantic / paragraph
   │
   ▼
_process_extract_entities          LLM 抽取实体与关系
   │  ├─ KV_STORAGE   缓存抽取结果
   │  ├─ VECTOR_STORAGE  实体/关系向量
   │  └─ GRAPH_STORAGE   写入节点/边
   ▼
_insert_done                       刷新索引、更新文档状态(PROCESSED)
```

**管线并发契约**(`pipeline.py`,通过 `pipeline_status` 共享状态协调并发写入者):

- `busy`:任何管线忙碌状态。**单独为 True 不阻塞入队**。
- `destructive_busy`:`/documents/clear` 或 `/documents/{doc_id}` 删除等破坏性作业专属,会 DROP 存储并删输入文件;此时入队会被拒绝,避免写入正在拆除的存储。
- `scanning` / `scanning_exclusive`:`/documents/scan` 任务运行中;`scanning_exclusive` 仅在分类阶段为 True,会拒绝入队,处理阶段前清除以允许并发上传。
- `pending_enqueues`:已预约但未完成的入队计数,供 scan 端点判断是否拒绝启动。
- `request_pending`:给运行中处理循环的"再查一次"信号。

契约允许**并发入队 + 处理**:新上传文档在循环处理批次中途落入 `doc_status`,循环在当前批次后看到 `request_pending`,重新查询并拾取新的 PENDING 行。

### 4.2 查询流(`aquery`)

```
查询 (QueryParam.mode)
   │
   ▼
operate.py 多模式检索
   ├─ local   聚焦特定实体的上下文检索
   ├─ global  基于社区/摘要的广域知识检索
   ├─ hybrid  local + global
   ├─ naive   不走图,直接向量搜索
   └─ mix     KG + 向量融合(推荐配合 reranker)
   │
   ▼
(可选)rerank         rerank.py
   │
   ▼
LLM 生成最终回答 / 流式返回
```

`QueryParam`(`base.py`)控制检索行为:`mode`、`top_k`、`chunk_top_k`、`max_entity_tokens`、`max_relation_tokens`、`max_total_tokens`、`enable_rerank`、`user_prompt`、`stream` 等。

---

## 5. 其他模块

- **评估(`lightrag/evaluation/`)**:基于 RAGAS 的 RAG 质量评估(`eval_rag_quality.py`)与离线检索检查(`offline_retrieval_check.py`),含样例数据集。
- **运维工具(`lightrag/tools/`)**:重建向量库(`rebuild_vdb.py`)、清理/迁移 LLM 缓存、哈希密码、Qdrant 旧数据迁移、可视化器(`lightrag_visualizer/`)。
- **Sidecar(`lightrag/sidecar/`)**:Sidecar 格式的占位符、中间表示(`ir.py`)、回填(`backfill.py`)、写入(`writer.py`)。
- **提示词(`lightrag/prompt.py`、`prompt_multimodal.py`)**:核心提示词与多模态提示词;`prompts/` 顶层目录供用户自定义。

---

## 6. 前端 WebUI(`lightrag_webui/`)

React 19 + TypeScript,使用 Bun + Vite + Tailwind。`src/` 结构:

| 目录 | 作用 |
|---|---|
| `components/` | 通用 UI 组件 |
| `features/` | 功能模块 |
| `api/` + `services/` | 接口调用层 |
| `stores/` | 状态管理(设置项 schema 在 `stores/settings.ts`,localStorage 持久化键 `settings-storage`) |
| `contexts/` | React Context |
| `hooks/` | 自定义 hooks |
| `locales/` | i18next 多语言资源 |

前端测试用 Bun 内置测试运行器(**非** Vitest/Jest)。

---

## 7. 部署方式

- **本地 / 开发**:`uv sync` 安装,`lightrag-server` 或 `uvicorn ... --reload` 启动。
- **Docker**:`Dockerfile`(完整)、`Dockerfile.lite`(精简)、`Dockerfile.postgres`;`docker-compose.yml` / `docker-compose-full.yml` / `docker-compose.podman.yml`。
- **Kubernetes**:`k8s-deploy/`(含数据库依赖清单与安装脚本)。
- **配置向导**:`scripts/setup/`(`make env-*` 目标,生成 `.env` 与 `docker-compose-final.yml`)。

存储与 LLM 后端均通过环境变量或构造参数配置(详见 `env.example`)。

---

## 8. 关键设计要点与易错点

1. **必须初始化存储**:`await rag.initialize_storages()` 是最常被遗漏的一步,缺失会触发 `AttributeError: __aenter__` / `KeyError: 'history_messages'`。
2. **`@final` 的 `LightRAG`**:mixin 组装是内部细节,不要试图子类化;通过公开 API(`ainsert`/`aquery`/`ainsert_custom_kg` 等)使用。
3. **嵌入模型切换**:必须清空数据目录(可保留 LLM 缓存),否则向量空间不匹配。
4. **管线并发**:破坏性作业(clear/delete)与 scan 分类阶段会独占存储,入队需遵循 `pipeline_status` 契约,避免静默丢文档。
5. **角色化 LLM**:不同角色用不同模型的能力由 `llm_roles.py` 统一管理,不要把角色逻辑塞进 provider 模块。
6. **代码风格**:后端注释/日志用英文;前端用 i18next 多语言;Python 遵循 PEP 8 + 类型注解 + `lightrag.utils.logger`,全异步。
