# HTTP 请求测试工具 / HTTP Request Tester

> 一个轻量级、浏览器端的 HTTP 请求调试工具，Postman / Insomnia 的简易替代品。
> A lightweight, browser-based HTTP request debugging tool — a simple alternative to Postman / Insomnia.

![Vue 3](https://img.shields.io/badge/Vue_3-4FC08D?logo=vue.js&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-646CFF?logo=vite&logoColor=white)
![Tailwind CSS v4](https://img.shields.io/badge/Tailwind_CSS_v4-06B6D4?logo=tailwindcss&logoColor=white)

---

## 功能特性 / Features

| 功能 | 说明 |
|------|------|
| **GET / POST** | 支持两种最常用的 HTTP 请求方法 |
| **自定义请求头** | 动态添加/删除请求头，支持覆盖 `Content-Type` |
| **Cookie** | 独立输入 Cookie，自动附加到请求中 |
| **表单参数** | 以 Key-Value 方式构建 `application/x-www-form-urlencoded` 请求体 |
| **JSON 参数** | 直接编辑 JSON 请求体，支持语法格式化 |
| **GET 附加查询参数** | 选择表单参数类型时，自动将参数拼接到 URL 查询字符串 |
| **CURL 导入** | 粘贴 cURL 命令即可自动填充 URL、方法、请求头、Cookie 及请求体 |
| **请求历史** | 自动保存最近 20 条请求记录，支持一键恢复 |
| **持久配置** | 可选持久化当前请求配置到 localStorage |
| **响应信息** | 展示状态码、状态文本、响应时间、响应大小、Content-Type、最终请求地址 |
| **JSON 格式化** | 请求体与响应结果均支持一键格式化 JSON |
| **一键清空** | 一键重置所有输入和返回结果 |
| **复制结果** | 一键复制响应内容到剪贴板 |
| **代理模式** | 预留代理转发接口（需配合 Node.js 代理服务） |

---

## 快速开始 / Quick Start

```bash
# 安装依赖 / Install dependencies
npm install

# 启动开发服务器 / Start dev server
npm run dev

# 构建生产版本 / Build for production
npm run build

# 预览生产构建 / Preview production build
npm run preview
```

开发服务器默认运行在 `http://localhost:5173`。

---

## 使用指南 / Usage Guide

### 基本流程 / Basic Flow

1. 输入 **请求地址**（需包含协议，例如 `https://api.example.com`）
2. 选择 **请求方式**（GET / POST）
3. 配置 **请求头**、**Cookie**、**参数**（可选）
4. 点击 **发送请求**，等待响应结果
5. 在左侧面板查看 **返回结果** 或 **请求响应信息**

### CURL 导入 / CURL Import

支持导入标准 cURL 命令，自动解析以下内容：

- URL（第一个非参数 token）
- 请求方式（`-X` / `--request`）
- 请求头（`-H` / `--header`）
- Cookie（从请求头中自动识别 `Cookie` 字段）
- 请求体（`-d` / `--data` / `--data-raw` / `--data-binary` / `--data-urlencode`）
- 自动识别 JSON 或表单格式

### 请求历史 / Request History

- 每次成功请求自动保存到历史记录（最多 20 条）
- 按 URL + 方法去重，重复请求会更新到顶部
- 点击 **恢复** 可快速填回请求配置

---

## 技术栈 / Tech Stack

- **[Vue 3](https://vuejs.org/)** — Composition API (`<script setup>`)
- **[Vite](https://vitejs.dev/)** — 构建工具 / Build tool
- **[Tailwind CSS v4](https://tailwindcss.com/)** — 样式框架 / Utility-first CSS
- **localStorage** — 状态持久化 / Persistent state storage
- **[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)** — 请求发送 / HTTP requests

---

## 项目结构 / Project Structure

```
http-client/
├── index.html           # HTML 入口
├── vite.config.js       # Vite 配置（含 Tailwind 插件）
├── package.json
├── CLAUDE.md            # AI 协作指南（仅供参考）
├── public/
│   └── favicon.svg
└── src/
    ├── main.js          # 应用入口
    ├── App.vue          # 单文件组件，包含所有逻辑与模板
    └── style.css        # 全局样式
```

项目采用 **单文件组件架构**，所有业务逻辑、状态管理、UI 模板均集中在 `src/App.vue`（约 1400 行），无需路由或状态管理库。

---

## 代理模式 / Proxy Mode

应用支持 **本地直连**（浏览器直接发送请求）和 **代理转发** 两种访问模式。

代理模式当前为预留状态，尚未接入后端服务。如需使用，您需要启动一个本地 Node.js 代理服务，将请求转发到 `http://localhost:3000/proxy`。

---

## 许可证 / License

[MIT](LICENSE)