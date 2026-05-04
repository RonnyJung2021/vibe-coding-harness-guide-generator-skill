# 实现指南：基于 LangChain（TS/JS）的 PDF 个人知识库问答（Harness + Cursor Agent）

> **读者**：刚接到任务、要在本机从零做出「可换 PDF、可对话理解」系统的自然人。  
> **Harness 对齐**：本指南把「任务契约、中间状态落盘、上下文裁剪、工具边界、轻量验收」写进每一步；不依赖模型临场发挥，而依赖环境与可复跑流程（参见仓库内 `harness guide/` 下的 Harness 实践与原文摘录）。  
> **技术定案**：运行时 **Node.js 20+**；语言 **TypeScript**；编排 **LangChain.js**；向量检索 **内存向量库 + JSON 持久化**（零额外服务，先把 Harness 跑通）；大模型与向量模型 **火山引擎方舟（豆包）OpenAI 兼容接口**（国内线路、可替换为同兼容形态的其他厂商）。  
> **协作方式**：尽量用 **Cursor Agent** 完成编码与改文件；你在每一步只做「打开界面 + 粘贴指令 + 批准运行」。

---

## 0. 开始前 10 分钟（本机一次）

| Harness 要素 | 你要做的事 |
|--------------|------------|
| 任务契约 | 确认目标：任意新 PDF → 入库 → 中文问答理解 PDF。 |
| 鉴权/边界 | 到 [火山引擎方舟](https://console.volcengine.com/ark) 开通模型服务，拿到 **API Key**；记下 **推理接入点**（形如 `https://ark.cn-beijing.volces.com/api/v3`）及 **模型 Endpoint ID**（对话模型、多模态/文本向量模型各一个，按控制台实际开通为准）。 |
| 可复现环境 | 安装 **Node.js 20 LTS** 与 **pnpm**（若不会装：用 Agent 第一步里让它检测并给出本机安装命令）。 |

**在 Cursor 里暂不需要打开 Agent**，完成控制台与 Node 即可。

---

## 1. 打开 Cursor Agent 的标准姿势（后续每一步重复用）

以下统一称为「**发一条 Agent 任务**」：

1. 用快捷键 **`⌘ + L`**（Windows/Linux：**`Ctrl + L`**）打开 **Chat** 面板。  
2. 在 Chat 面板 **顶部或输入框附近**，将模式从 **Ask** 切到 **Agent**（若你的 Cursor 版本将 Agent 合在 **Composer** 里：**`⌘ + I`** 打开 Composer，再选 **Agent**）。  
3. 在 **底部输入框** 点击一下，**完整粘贴**本指南该步骤「**Agent 输入框粘贴**」区块中的文字。  
4. 若弹出 **允许运行终端 / 允许网络 / 写入文件**，在确认内容无害后点 **Accept / Run**（Harness 里这叫「人在环上的审批」）。  
5. 等 Agent 跑完；再进入下一步。

> 若某步失败：把 **终端红色报错全文** 复制，新开一条 Agent 消息，首行写「上一步失败，请根据报错修复」，再粘贴报错。

---

## 2. 阶段 A：仓库骨架与 Harness 契约文件

### A1. 初始化 TS 工程与目录约定

**目的**：固定「输入/输出放哪、环境变量叫什么」——对应 Harness 的 **任务表达 + 状态落盘**。

**在 Cursor 中**：左侧 **Explorer** 打开本仓库根目录 `harness-langchain-kb`（若未打开：**File → Open Folder** 选该目录）。

**发一条 Agent 任务**，粘贴：

```text
在本仓库根目录初始化一个 Node.js + TypeScript 工程（可用 pnpm），要求：
1) package.json 的 name 为 kb-rag-local；type 为 module；scripts 含 build（tsc）、ingest（tsx src/ingest.ts）、ask（tsx src/ask.ts）。
2) 目录：src/ 放源码；kb_store/ 放向量与元数据 JSON（加入 .gitignore 仅忽略 kb_store/*.json 或整个 kb_store 数据文件，但保留 kb_store/.gitkeep）；pdfs/ 放用户 PDF（加入 pdfs/.gitkeep）；根目录 .env.example 列出 ARK_API_KEY、ARK_BASE_URL、ARK_CHAT_MODEL、ARK_EMBED_MODEL。
3) tsconfig 使用 ES2022、moduleResolution bundler 或 node16、strict true。
4) 写极简 README.md（中文）：说明把 PDF 放进 pdfs/、复制 .env.example 为 .env 并填密钥、pnpm ingest、pnpm ask。
5) 不要实现业务逻辑，只要可编译的空壳与目录。
完成后执行 pnpm install 与 pnpm run build，确保通过。
```

**验收**：终端里 `pnpm run build` 无报错；磁盘上出现 `src/`、`pdfs/`、`kb_store/`、`.env.example`。

---

## 3. 阶段 B：依赖与「模型/向量」边界（工具治理）

### B1. 安装 LangChain 与 PDF、环境变量依赖

**Harness 对齐**：把「能调用外网 API」和「能读本地 PDF」变成 **显式依赖与模块边界**，避免隐式全局脚本。

**发一条 Agent 任务**，粘贴：

```text
在现有 TS 工程上添加依赖并锁定可运行组合：
- langchain @langchain/core @langchain/community
- @langchain/openai（用于 OpenAI 兼容的 Chat 与 Embeddings，指向方舟 baseURL）
- pdf-parse 以及类型 @types/pdf-parse（若需要）
- dotenv、tsx、typescript（若尚未有）

实现 src/config.ts：从 process.env 读取 ARK_API_KEY、ARK_BASE_URL（默认 https://ark.cn-beijing.volces.com/api/v3）、ARK_CHAT_MODEL、ARK_EMBED_MODEL；缺任一 throw 清晰错误。
不要写死密钥。更新 .env.example 注释为中文。
pnpm run build 必须通过。
```

**验收**：`pnpm run build` 通过；`src/config.ts` 存在且 import 不报红。

---

## 4. 阶段 C：PDF → 文本块 → 向量（状态保存与裁剪）

### C1. 入库流水线（ingest）

**Harness 对齐**：**持久化中间产物**（块与向量可审计）；**上下文裁剪**（按字符递归切分，控制单块大小）。

**发一条 Agent 任务**，粘贴：

```text
实现 PDF 入库：src/ingest.ts
行为：
1) 从命令行参数读取 PDF 路径；若为相对路径，相对仓库根目录解析。
2) 用 LangChain 的 PDFLoader（或等价）加载文本；空文档要报错说明。
3) 使用 RecursiveCharacterTextSplitter：chunkSize 800、chunkOverlap 120（中文友好可调）。
4) 使用 @langchain/openai 的 OpenAIEmbeddings，configuration.baseURL = ARK_BASE_URL，apiKey = ARK_API_KEY，model = ARK_EMBED_MODEL（方舟的 embedding 端点 ID）。
5) 使用 MemoryVectorStore.fromDocuments 生成向量库；将 store 与文档元数据（source 文件名、chunk 索引）序列化写入 kb_store/ 下固定文件名（例如 manifest.json + vectors.json 或 LangChain 推荐方式），要求下次启动可加载继续追加或覆盖策略二选一：默认「同一 source 先删除旧块再写入」避免重复。
6) 控制台打印：块数量、耗时、写入路径。

保持代码模块化：src/pdf/loadPdf.ts、src/chunk/split.ts、src/embed/arkEmbeddings.ts、src/store/localVectorStore.ts 等可按你判断拆分。

完成后我应能运行：pnpm ingest -- pdfs/sample.pdf
```

**验收**：任意小 PDF 放入 `pdfs/`，填好 `.env` 后执行 `pnpm ingest -- pdfs/xxx.pdf`，`kb_store/` 有更新，终端有块数统计。

### C2. 轻量「真做好了吗」检查（Harness 验收）

**发一条 Agent 任务**，粘贴：

```text
添加 scripts 中的 ingest:smoke：用一个最小 PDF（若没有就生成一个只含几十字中文的 PDF 到 pdfs/_smoke.pdf）跑 ingest 并在末尾打印 kb_store 文件大小与 chunk 数断言（chunk>0）。README 增加「验收」一节说明该命令。
```

---

## 5. 阶段 D：问答循环（RAG + 国内大模型）

### D1. 检索增强问答 CLI

**Harness 对齐**：**反馈回流**——把检索到的片段编号打印或写入日志，便于人眼核对「答的是不是书里的」。

**发一条 Agent 任务**，粘贴：

```text
实现问答：src/ask.ts
行为：
1) 从 kb_store 加载向量库；失败则提示先 ingest。
2) 从命令行读取用户问题字符串（支持多个词）。
3) 向量检索 topK=4，带分数阈值（例如 <0.35 可配置）过滤弱相关。
4) 将检索片段拼成 system/context，使用 ChatOpenAI（@langchain/openai）配置 baseURL=ARK_BASE_URL、apiKey、model=ARK_CHAT_MODEL，temperature 0.2。
5) 输出：先打印「引用片段摘要（文件名+chunk序号+前80字）」，再打印「回答」。
6) 支持连续对话可选：第一版可单轮；若实现多轮，用简单数组保存 history 并注意总 token 裁剪。

pnpm ask -- "这份资料的核心结论是什么？"

README 补充示例问题与常见问题（API 429、PDF 无文本只有扫描图——提示需 OCR 版 PDF）。
pnpm run build 通过。
```

**验收**：对刚入库的 PDF 提问，能看到引用片段 + 中文回答；断网或 Key 错误时有**可解释**报错（对应 Harness「错误分类」）。

---

## 6. 阶段 E：把「换 PDF」变成稳定操作（熵管理）

### E1. 约定与文档，而不是口头记忆

**发一条 Agent 任务**，粘贴：

```text
在仓库根目录新增 docs/KB_OPERATIONS.md（中文），写清：
1) 新增一本 PDF 的标准步骤（复制到 pdfs/ → pnpm ingest -- ...）。
2) 替换知识库时如何清理 kb_store（给一条 rm 或 npm script clean:kb）。
3) 环境变量说明与方舟控制台截图占位说明（文字即可）。
4) 安全：不要把 .env 提交 git；PDF 可能含隐私，kb_store 不提交。

在 package.json 增加 clean:kb 脚本。检查 .gitignore 已覆盖 .env 与 kb_store 数据文件。
```

---

## 7. 阶段 F（可选）：用 Cursor Agent 做「半自动阅读」而不是只有 CLI

### F1. 极简本地 HTTP 服务（浏览器问一句）

**发一条 Agent 任务**，粘贴：

```text
增加可选 dev 服务：使用最简依赖（如 hono 或 express 任选其一），实现 src/server.ts：
- POST /ingest body: { path: string } 调用现有入库逻辑（限制 path 必须在 pdfs/ 下，防止路径穿越）
- POST /ask body: { question: string } 返回 JSON：{ citations: [...], answer: string }
- 端口 8787，pnpm dev:server 启动
README 说明仅本机使用、无鉴权风险

注意：工具边界与路径校验写在代码里，不依赖模型自觉。
```

---

## 8. 你在日常使用中「唯一需要记住」的三句话

1. **加书**：把 PDF 丢进 `pdfs/`，Agent 或终端执行：`pnpm ingest -- pdfs/你的文件.pdf`  
2. **提问**：`pnpm ask -- "用三句话总结第二章"`  
3. **坏了先看**：`kb_store/` 是否生成、`.env` 是否四项齐全、方舟控制台模型是否欠费/未开通

---

## 9. Harness 自检表（交付前打勾）

- [ ] **任务契约**：ingest / ask 的输入输出在 README 与 `docs/KB_OPERATIONS.md` 中写死说法一致。  
- [ ] **状态落盘**：HTML 类比——PDF 原文保留在 `pdfs/`，向量与 manifest 在 `kb_store/`。  
- [ ] **上下文裁剪**：分块参数与 topK、阈值在代码或常量中可调、有注释。  
- [ ] **工具治理**：所有外网调用走 `config.ts`；可选 HTTP 服务有路径白名单。  
- [ ] **验证**：`ingest:smoke` 或等价命令能一键证明「管道末端有 chunk」。  
- [ ] **错误可解释**：Key/网络/空 PDF 分别有不同提示。  
- [ ] **安全**：`.env`、密钥、客户 PDF 不进版本库。

---

## 10. 参考与命名对齐

- 本仓库 **Harness 概念与实践**：`harness guide/juejin-post-7620226704209592360.md`、`harness guide/harness-practice-web-article-to-markdown.md`  
- LangChain JS 官方文档：以安装页与 `MemoryVectorStore`、`PDFLoader`、`ChatOpenAI` 为准（包名与 API 随版本演进，以 Agent 安装时解析结果为准）。  
- 方舟 OpenAI 兼容说明：以火山引擎控制台「API 接入」文档中的 **Base URL、鉴权头、模型 ID** 为准。

---

**文档版本**：与 2026 年 Cursor / LangChain.js 常见工作流对齐；若你使用的 Cursor 将 Agent 放在 Composer 内，只需把本文「`⌘ + L`」替换为你本机打开 **Agent/Composer** 的实际快捷键即可，**粘贴内容不变**。
