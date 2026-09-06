# Milvus RAG 示例项目

这是一个用于学习 RAG（Retrieval-Augmented Generation，检索增强生成）的示例项目。

项目使用：

- Milvus 保存并检索文本向量；
- 阿里云百炼 `text-embedding-v4` 生成文本向量；
- 通义千问根据检索结果生成答案；
- FastAPI 提供向量检索和 RAG 对话接口；
- MinIO、etcd 和 Attu 提供 Milvus 所需的存储、元数据和可视化管理能力。

## 系统结构

```text
用户请求
   ↓
RAG 服务（8001 或 8002）
   ↓
Milvus API（8000）
   ↓
Milvus（19530）
   ├── etcd：保存元数据
   ├── MinIO：保存对象数据
   └── Attu：可视化管理界面（3000）
```

## 运行阶段与启动顺序

项目分为“构建知识库”和“在线问答”两个阶段，各部分通过 Milvus 和 HTTP API 串联。

### 阶段一：构建知识库

`milvus_insert.py` 负责读取知识库、切分文本、生成向量，并将数据写入 Milvus。这一步通常只需要在首次运行、知识库发生变化或本地数据库被清空后执行。

```text
知识库文件（milvus.doc/）
          ↓
文本读取与切块
          ↓
生成 Embedding 向量
          ↓
写入 Milvus Collection
```

### 阶段二：在线问答

`milvus_api.py` 负责检索 Milvus，RAG Service 负责接收用户问题、获取相关知识片段并调用大模型生成答案。

```text
用户问题
   ↓
RAG Service（8001 或 8002）
   ↓ HTTP
Milvus API（8000）
   ↓
Milvus 向量检索（19530）
   ↓
相关知识片段
   ↓
大模型生成最终回答
```

### 正确启动顺序

首次运行：

```text
1. 启动 Docker 中的 Milvus 等服务
2. 测试 Milvus 连接
3. 运行 milvus_insert.py 构建知识库
4. 运行 milvus_api.py 启动检索服务
5. 运行 rag_service_simple.py 或 rag_service.py
6. 开始用户问答
```

后续运行且知识库已经导入：

```text
1. 启动 Docker 中的 Milvus 等服务
2. 运行 milvus_api.py
3. 运行 rag_service_simple.py 或 rag_service.py
4. 开始用户问答
```

## 重要文件

| 文件 | 作用 |
|---|---|
| `docker-compose.yml` | 启动 Milvus、etcd、MinIO 和 Attu |
| `milvus_config.py` | Milvus、Embedding 和索引配置 |
| `milvus_insert.py` | 切分知识库、生成向量并写入 Milvus |
| `milvus_query.py` | 直接测试向量、关键词和混合检索 |
| `milvus_api.py` | 将 Milvus 检索能力封装为 FastAPI 服务 |
| `rag_service_simple.py` | 简单版 RAG 服务，端口 8002 |
| `rag_service.py` | 增强版 RAG 服务，端口 8001 |
| `milvus.doc/` | 原始知识库文件 |
| `test_milvus.py` | 测试 Python 是否能连接 Milvus |
| `test_rag_simple.py` | 测试简单版 RAG |
| `test_rag.py` | 测试增强版 RAG |

## 一、新电脑首次运行

### 1. 安装前置软件

需要提前安装：

- Git
- Conda（Miniconda 或 Anaconda）
- Docker Desktop

建议为 Docker Desktop 分配至少 8 GB 内存。

### 2. 克隆项目

```bash
git clone https://github.com/daniel-czz/milvus-rag.git
cd milvus-rag
```

### 3. 创建 Python 环境

```bash
conda create -n milvus-rag python=3.9 -y
conda activate milvus-rag
pip install -r requirements.txt
```

终端前缀出现 `(milvus-rag)`，表示环境已经激活。

### 4. 配置百炼 API Key

在项目根目录创建 `.env`：

```dotenv
DASHSCOPE_API_KEY=你的百炼API密钥
```

`.env` 包含敏感信息，已被 `.gitignore` 排除，不会上传到 Git。

### 5. 启动 Milvus

先打开 Docker Desktop，等待 Docker Engine 启动完成，然后执行：

```bash
docker compose pull
docker compose up -d
docker compose ps
```

当前项目通过 `docker.1ms.run` 拉取 Docker Hub 上的 Milvus、MinIO 和 Attu 镜像，以改善部分网络环境中的下载问题。etcd 仍从 `quay.io` 拉取。

`docker compose ps` 中的服务刚启动时可能显示：

```text
health: starting
```

这不是错误。等待几十秒后再次执行 `docker compose ps`，Milvus、MinIO 和 etcd 应逐渐变为 `healthy`。

### 6. 测试 Milvus 连接

```bash
python test_milvus.py
```

首次启动时显示集合数量为 `0` 是正常的，表示 Milvus 已运行，但知识库尚未导入。

### 7. 导入知识库

```bash
python milvus_insert.py
```

该脚本会：

1. 连接 Milvus；
2. 创建 `ai_knowledge_base` Collection；
3. 创建 `IVF_FLAT` 向量索引；
4. 读取知识库文件；
5. 调用 Embedding 模型生成向量；
6. 将文本、向量和元数据写入 Milvus。

当前脚本默认导入：

```text
milvus.doc/law.md
```

如需导入其他 Markdown 文件，可以修改 `milvus_insert.py` 中的 `file_path`。

## 二、启动简单版 RAG

简单版适合学习最基础的 RAG 流程：检索知识片段，然后让大模型根据知识片段回答问题。

### 终端 1：启动 Milvus API

```bash
conda activate milvus-rag
cd milvus-rag
python milvus_api.py
```

接口文档：<http://localhost:8000/docs>

### 终端 2：启动简单版 RAG

```bash
conda activate milvus-rag
cd milvus-rag
python rag_service_simple.py
```

接口文档：<http://localhost:8002/docs>

### 测试简单版 RAG

打开第三个终端：

```bash
conda activate milvus-rag
cd milvus-rag
python test_rag_simple.py
```

也可以发送 HTTP 请求：

```bash
curl -X POST http://localhost:8002/chat \
  -H "Content-Type: application/json" \
  -d '{
    "question": "法律服务行业有哪些发展趋势？",
    "top_k": 5
  }'
```

## 三、启动增强版 RAG

增强版增加了意图识别、查询扩展、来源引用、流式输出和多轮对话等功能。

先保持 `milvus_api.py` 运行，再打开新终端：

```bash
conda activate milvus-rag
cd milvus-rag
python rag_service.py
```

接口文档：<http://localhost:8001/docs>

测试：

```bash
python test_rag.py
python test_rag_enhanced.py
```

## 四、一键启动

完成 Python 环境和 `.env` 配置后，也可以使用项目脚本：

```bash
chmod +x start_rag_all.sh stop_rag_all.sh
./start_rag_all.sh
```

脚本会依次：

1. 检查 Docker、Python 和 `.env`；
2. 启动 Milvus；
3. 询问是否导入知识库；
4. 启动 Milvus API；
5. 启动增强版 RAG 服务。

首次启动时，询问是否导入数据应选择 `y`。已有数据时选择 `n`，避免重复导入。

## 五、服务地址

| 服务 | 地址 | 说明 |
|---|---|---|
| Attu | <http://localhost:3000> | Milvus 可视化管理 |
| MinIO Console | <http://localhost:9001> | MinIO 管理界面 |
| Milvus | `localhost:19530` | Python SDK/gRPC 连接地址 |
| Milvus 健康接口 | <http://localhost:9091/healthz> | Milvus 健康检查 |
| Milvus API | <http://localhost:8000/docs> | 向量检索 API 文档 |
| 增强版 RAG | <http://localhost:8001/docs> | 增强版问答 API 文档 |
| 简单版 RAG | <http://localhost:8002/docs> | 简单版问答 API 文档 |

## 六、停止和恢复服务

临时停止 Docker 容器：

```bash
docker compose stop
```

恢复已停止的容器：

```bash
docker compose start
```

停止并删除容器和 Compose 网络：

```bash
docker compose down
```

`docker compose down` 不会删除项目目录中的 `volumes/` 数据。之后可以使用以下命令重新创建容器：

```bash
docker compose up -d
```

不要随意执行：

```bash
docker compose down -v
```

也不要删除 `volumes/`，否则本地 Milvus 数据可能丢失。

## 七、未提交到 Git 的文件

以下文件由 `.gitignore` 排除：

| 内容 | 原因 | 新电脑如何处理 |
|---|---|---|
| `.env` | 包含 API Key | 手动创建 |
| `*.ipynb` | Notebook 较大，非运行依赖 | 不需要恢复 |
| `volumes/` | 本地数据库运行数据 | 启动并重新导入知识库 |
| `logs/` | 运行日志 | 自动生成 |
| `pids/` | 后台进程编号 | 自动生成 |
| `__pycache__/` | Python 缓存 | 自动生成 |

因此，从 GitHub 克隆项目后，只需要重新创建 `.env`、启动 Milvus 并运行一次 `milvus_insert.py`，即可重建本地运行环境。

## 八、更多文档

- [完整 RAG 使用指南](README_RAG.md)
- [Milvus 使用说明](README_MILVUS.md)
- [快速启动指南](QUICK_START.md)
- [简单版与增强版对比](RAG_COMPARISON.md)
- [Attu 使用指南](ATTU_GUIDE.md)
