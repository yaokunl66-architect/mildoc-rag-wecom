# mildoc-rag-wecom

基于 RAG(检索增强生成)的企业知识库系统:文档放进对象存储后自动解析、向量化并写入 Milvus,用户通过企业微信客服提问,系统检索知识库并由大模型生成回答,同时提供一个 Web 管理后台管理文档。

## 功能

- **文档自动入库**:监听 MinIO / 阿里云 OSS 的文件事件,解析 PDF、Word、Excel、PPT、Markdown、文本等文件,切分后向量化写入 Milvus。
- **三种索引模式**:`listen`(实时增量)、`full-refresh`(全量重建)、`backfill`(补建遗漏)。
- **企业微信客服问答**:接收企业微信客服回调,检索知识库、重排序后调用大模型回答,并附带来源文档。
- **场景识别与监控**:对用户问题做客服场景分类,可选接入 Langfuse 追踪调用。
- **管理后台**:登录后浏览、上传、下载、删除文档,创建/删除目录,查看文件详情。

## 架构

```
文档 ──> MinIO / 阿里云 OSS ──> mildoc_index(解析 + 向量化) ──> Milvus
                                      ^                              |
                          mildoc_admin(上传/管理)                    v
                                                  mildoc_wxkf(检索 + 重排 + 大模型) <── 企业微信客服
```

| 目录 | 说明 | 默认端口 |
| --- | --- | --- |
| `mildoc_index/` | 文档解析与向量化入库服务(LangChain、markitdown,可选 MinerU) | - |
| `mildoc_wxkf/` | 企业微信客服回调与 RAG 问答服务(Flask + gunicorn) | 8890 |
| `mildoc_admin/` | 文档管理后台(Flask + Flask-Login) | 8870 |
| `mildoc_milvus/` | Milvus 的 docker-compose 配置(本地 / MinIO / OSS 三种存储),以及 Attu 可视化脚本 | 19530 |

## 技术栈

Python 3.12、uv、Flask、gunicorn、LangChain、Milvus、MinIO / 阿里云 OSS、阿里云百炼(OpenAI 兼容接口,含 Embedding 与 Rerank)、Langfuse(可选)。

## 快速开始

### 1. 准备配置

每个服务目录下都有 `.env.example`,复制为 `.env` 并填写实际值。`.env` 已被 `.gitignore` 排除,请勿提交。

```bash
cp mildoc_index/.env.example mildoc_index/.env
cp mildoc_wxkf/.env.example  mildoc_wxkf/.env
cp mildoc_admin/.env.example mildoc_admin/.env
```

需要填写的主要内容:MinIO / OSS 访问凭据、Milvus 密码、百炼 API Key(LLM、Embedding、Rerank)、企业微信的 `CORP_ID` / `TOKEN` / `ENCODING_AES_KEY` 等、管理后台账号密码与 `FLASK_SECRET_KEY`。

### 2. 启动 Milvus

选择一种存储方案,进入对应目录(先在该目录的 `.env` 里设置 `DOCKER_VOLUME_DIRECTORY`):

```bash
cd mildoc_milvus/milvus_local   # 或 milvus_minio / milvus_oss
docker compose up -d
```

可选:运行 `mildoc_milvus/run_attu` 启动 Attu 查看向量库。

### 3. 启动各服务

需要先安装 [uv](https://docs.astral.sh/uv/) 和 Python 3.12。

```bash
# 文档索引(MinIO 实时监听)
cd mildoc_index && ./run_mildoc_index
# 使用阿里云 OSS 时:./run_mildoc_index_oss

# 企业微信客服问答
cd mildoc_wxkf && ./run_mildoc_wxkf

# 管理后台
cd mildoc_admin && ./run_mildoc_admin
```

索引服务也可以手动指定模式:

```bash
uv run main.py --provider minio --mode full-refresh   # 全量重建
uv run main.py --provider minio --mode backfill       # 补建遗漏
uv run main.py --provider minio --mode listen         # 实时监听
```

### 4. 配置企业微信回调

在企业微信客服后台把回调地址设置为 `http(s)://<你的域名>:8890/callback/command`,Token 和 EncodingAESKey 与 `mildoc_wxkf/.env` 保持一致。健康检查接口为 `/health`。

## 说明

- 示例文档 `mildoc_index/data/HR.pdf` 可用于测试解析与问答。
- 启用 MinerU 解析 PDF 需要额外安装依赖,见 `mildoc_index/.env.example` 中的注释。
- 本仓库不包含任何密钥;使用前请自行准备并妥善保管各项凭据。
