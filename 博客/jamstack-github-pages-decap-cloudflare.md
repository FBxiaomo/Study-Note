---
title: 基于 JAMstack 架构的零成本动态博客：GitHub Pages + Decap CMS + Cloudflare Workers 实战解析
description: 用 GitHub Pages 托管静态博客、Decap CMS 实现在线写作、Cloudflare Workers 代理 GitHub
  OAuth，从架构原理到配置接入，梳理认证、发布与跨仓库部署的完整流程。
pubDate: 2026-10-24
tags:
  - JAMstack
  - GitHub Pages
  - Decap CMS
  - Cloudflare Workers
  - Astro
cover: images/screenshot_2026-10-26_162044_292.png
draft: false
slug: jamstack-github-pages-decap-cloudflare
---

# 基于 JAMstack 架构的零成本动态博客：GitHub Pages + Decap CMS + Cloudflare Workers 实战解析

## 前言：纯静态博客的痛点与破局

在静态网站生成器（SSG）盛行的当下，使用 Hugo、Hexo 或 Astro 配合 GitHub Pages 托管博客，是开发者常见的建站方案。它具备静态页面访问快、免费托管以及基于 Git 的版本控制等优点。

然而，这种架构也有一个明显的使用门槛：**缺乏开箱即用的在线内容管理系统（CMS）**。每次发布文章，往往需要在本地使用 Markdown 编辑器编写，再通过命令行执行 `git commit` 和 `git push`。

为了实现在线写作，可以租用云服务器运行数据库和后台程序。但在 JAMstack（JavaScript + APIs + Markup）理念下，也可以利用 **Git-based CMS**，让 Git 仓库承担内容存储与版本管理，以 Serverless 函数处理认证，从而免去自建服务器的成本。

本文将解析 **GitHub Pages + Decap CMS + Cloudflare Workers** 的联动逻辑，并给出在线写作环境的接入步骤。

> 本文的“动态”指在线编辑和自动发布能力，读者访问的仍是构建后的静态页面；“零成本”指使用平台免费额度、不购买服务器和自定义域名，实际额度与限制以各平台当前政策为准。

## 一、技术栈与核心架构解析

这套系统的核心在于：**将 GitHub 仓库作为内容存储，将 Serverless 边缘函数作为 OAuth 认证网关。**

### 1. 核心组件

- **存储与托管（Content Storage & Hosting）**：GitHub Repository 与 GitHub Pages。仓库存储 Markdown 源文件，GitHub Actions 负责构建与部署，Pages 对外提供静态网站。
- **内容管理前端（SPA Frontend）**：Decap CMS。它是运行在浏览器中的单页应用，通过 GitHub API 读写仓库文件，无需部署传统的 CMS 服务端。
- **认证网关（Auth Gateway）**：Cloudflare Workers。提供 OAuth 代理服务，在服务端使用 Client Secret 交换 Access Token，避免把应用密钥放进前端代码。

这里的“仓库作为数据库”是对内容存储方式的类比，并不意味着 Git 具备关系型数据库的查询和事务能力。

### 2. 系统交互逻辑：OAuth 2.0 与 API 调用

系统的联动分为两个核心流程。

**流程 A：身份认证（OAuth 2.0 授权码模式）**

1. 用户访问 `/admin/` 中的 Decap CMS，点击登录。
2. CMS 打开登录弹窗，请求配置好的 Worker `/auth` 接口。
3. Worker 生成并保存短期有效的 `state` 校验信息，再引导用户跳转至 GitHub 授权页面。
4. 用户同意后，GitHub 携带临时 `code` 和 `state`，重定向回 Worker 的 `/callback`。
5. Worker 校验 `state`，在服务端使用 `code`、Client ID 和 Client Secret 交换 Access Token，并按需验证允许登录的 GitHub 账号。
6. Worker 通过 Decap CMS 兼容的弹窗消息协议，把 Token 回传给指定来源的 CMS 页面。CMS 在浏览器中保存登录信息，随后使用 Token 访问 GitHub API。

**Client Secret 始终留在 Worker；Access Token 则需要交给浏览器中的编辑器使用。** 因此，回传时要校验消息来源，并限制 `postMessage` 的目标 origin。

**流程 B：内容写入与部署（Git-based CI/CD）**

1. 用户在 Decap CMS 中编写文章。
2. 点击发布后，在默认发布模式下，CMS 调用 GitHub API，把 Markdown 和图片提交到配置的仓库分支。
3. 如果文章与网站位于同一仓库，主分支的 `push` 事件可以直接触发 GitHub Actions，重新构建并部署到 Pages。
4. 如果文章与网站位于不同仓库，还需要额外触发网站仓库的构建，构建时再同步文章。本文博客采用的就是这种分离方式，后文会说明。

## 二、实战教程：接入在线写作环境

以下步骤假定你已经有一个能够正常构建并部署到 GitHub Pages 的博客，且登录账号对内容仓库具有写入权限。示例先以 **Astro 网站与文章同仓库** 为基础，目录需与你现有的内容集合配置保持一致。

### Step 1：创建 GitHub OAuth App，获取凭证

1. 登录 GitHub，进入 `Settings → Developer settings → OAuth Apps`，点击 `New OAuth App`。
2. 填写基础信息：

   | 字段 | 示例 |
   | --- | --- |
   | Application name | `My Blog CMS` |
   | Homepage URL | `https://yourname.github.io` |
   | Authorization callback URL | `https://your-worker-name.workers.dev/callback` |

3. 注册成功后，记录 **Client ID**。
4. 点击 `Generate a new client secret`，生成并妥善保存 **Client Secret**。

如果 Worker 地址尚未确定，可以先填写预期地址，部署后再回来修改。不要把 Client Secret 提交到仓库，也不要填入公开的 `config.yml`。

### Step 2：部署 Cloudflare Worker，配置 OAuth 代理

Cloudflare Workers 免费计划当前提供每日 10 万次 Worker 请求额度，个人博客的登录代理通常只会消耗其中很少一部分。具体限制见 [Cloudflare Workers 官方文档](https://developers.cloudflare.com/workers/platform/limits/)。

1. 登录 Cloudflare，进入 `Workers & Pages`，创建 Worker。
2. 部署与 Decap CMS 登录协议兼容的 Worker 代码。可以参考本博客的 [Worker 实现](https://github.com/FBxiaomo/FBxiaomo.github.io/blob/17f7eb5e805f30318b51a2d8f51ccb1938e3c51f/workers/study-note-oauth/worker.mjs)，它包含 `/auth`、`/callback`、`/health` 和用于请求网站构建的 `/publish` 路由。
3. 进入 Worker 的 `Settings → Variables and Secrets`，配置以下变量：

   | 名称 | 类型 | 内容 |
   | --- | --- | --- |
   | `GITHUB_CLIENT_ID` | Secret 或文本 | OAuth App 的 Client ID |
   | `GITHUB_CLIENT_SECRET` | Secret | OAuth App 的 Client Secret |
   | `CMS_ORIGIN` | 文本 | CMS 所在网站的来源，例如 `https://yourname.github.io`，不含路径 |
   | `ALLOWED_GITHUB_LOGIN` | 文本 | 允许登录的 GitHub 用户名 |
   | `ALLOWED_GITHUB_ID` | 文本，可选 | 对应账号的数字 ID，用于额外校验 |

4. 部署后，访问 `https://your-worker-name.workers.dev/health`，检查配置是否就绪。
5. 回到 GitHub OAuth App，将回调地址设置为实际的 `https://your-worker-name.workers.dev/callback`。

参考实现的 `/publish` 目标仓库、工作流和分支固定在服务端。复用时需要把其中的 `FBxiaomo/FBxiaomo.github.io`、`deploy.yml` 和 `main` 改成自己的配置；仅使用同仓库的 `push` 自动部署时，可以不接入这个发布接口。

这个实现默认申请 `public_repo` 权限，适用于公开仓库。私有内容仓库需要重新评估 OAuth 权限范围及部署端的读取凭证，不能直接照搬公开仓库配置。

> `state` 校验、短期有效的认证 Cookie、账号限制和明确的回传 origin，都是认证流程的一部分。不要只实现“拿到 code 就换 Token”的简化版本。

### Step 3：配置 Decap CMS 前端接入

#### 3.1 创建后台入口

Decap CMS 至少需要一个加载编辑器的 HTML 入口和一份配置文件。**只创建 `config.yml` 不会自动生成编辑器页面。**

在 Astro 中，创建以下目录：

```text
public/
└── admin/
    ├── index.html
    └── config.yml
```

`public/admin/index.html` 内容如下。示例固定使用本博客已经接入的版本，升级时应验证兼容性。

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="robots" content="noindex, nofollow" />
    <title>博客写作后台</title>
  </head>
  <body>
    <script src="https://cdn.jsdelivr.net/npm/decap-cms@3.16.3/dist/decap-cms.js"></script>
  </body>
</html>
```

不同生成器的静态目录并不相同：Astro 使用 `public/admin/`，Hugo 通常使用 `static/admin/`，Hexo 通常使用 `source/admin/`。最终发布出来的地址应为 `/admin/`，详见 [Decap CMS 安装说明](https://decapcms.org/docs/install-decap-cms/)。

如果已经用 `src/pages/admin/index.astro` 创建了后台页面，就沿用该入口，不要再创建相同路由的静态 HTML。

#### 3.2 编写配置文件

以下示例的文章目录为 `src/content/blog`，图片目录为 `public/images/uploads`。请根据自己项目的内容加载路径调整。

```yaml
# public/admin/config.yml
backend:
  name: github
  repo: your-username/your-repo-name
  branch: main
  base_url: https://your-worker-name.workers.dev
  auth_endpoint: auth

locale: 'zh_Hans'
site_url: https://yourname.github.io
media_folder: "public/images/uploads"
public_folder: "/images/uploads"

collections:
  - name: "posts"
    label: "博客文章"
    folder: "src/content/blog"
    create: true
    slug: "{{year}}-{{month}}-{{day}}-{{slug}}"
    fields:
      - {label: "标题", name: "title", widget: "string"}
      - {label: "摘要", name: "description", widget: "text"}
      - {label: "日期", name: "pubDate", widget: "datetime"}
      - {label: "标签", name: "tags", widget: "list", required: false}
      - {label: "草稿", name: "draft", widget: "boolean", default: false}
      - {label: "正文", name: "body", widget: "markdown"}
```

注意三个对应关系：

- `repo` 和 `folder` 指向 **CMS 要写入的仓库与目录**，不一定是网站源码仓库。
- `fields` 必须匹配静态生成器的文章 schema。例如项目使用 `pubDate`，就不能只写一个 `date` 字段；`draft` 也需要由网站构建逻辑实际过滤。
- `media_folder` 是图片在仓库中的路径，`public_folder` 是构建后网页里的访问路径，两者不能混用。上面的图片路径以根域名部署为前提，项目站点还要考虑 `base` 路径。

提交并部署后，访问博客的 `/admin/`，即可加载编辑器并尝试 GitHub 登录。[GitHub backend 文档](https://decapcms.org/docs/github-backend/) 说明了账号写入权限等要求。

## 三、进阶优化与常见排坑指南

### 1. 为什么界面的右下角会出现“更新网站”提示？

本博客右下角的“更新网站”按钮是自定义功能，不是安装 Decap CMS 后自动出现的标准界面。

原因在于，本博客采用了两个仓库：

- `Study-Note` 的 `博客/` 目录保存 Markdown 原稿。
- `FBxiaomo.github.io` 保存 Astro 网站源码，并在构建时同步前者的文章。

**向内容仓库提交文章，不会自动触发另一个仓库的 `push` 工作流。** 因此，后台监听 Decap CMS 的 `postSave` 事件，再调用 Worker 的 `/publish` 接口，请求网站仓库运行 `workflow_dispatch`；删除文章后则手动点击“更新网站”。网站另有定时同步作为补充。

这里还需要满足两个条件：网站工作流声明了 `workflow_dispatch`，用于调用 GitHub API 的账号和 Token 具有触发目标工作流的权限。目标仓库与工作流固定在服务端，避免浏览器任意指定发布位置。

按钮成功提示只表示构建请求已经发出。**文章已保存、构建已启动、Pages 已部署完成，是三个不同的状态。** 刷新页面无法代替构建，应先到 GitHub Actions 查看构建和部署结果。

本博客的实际配置由 `scripts/cms-config.mjs` 生成，文章写入 `Study-Note/博客`，图片写入 `博客/images`；构建时再解析仓库相对图片链接。它与前面的同仓库 Astro 示例在目录和发布方式上有所不同。

### 2. Token 失效问题

这里创建的是 **GitHub OAuth App**。不要把 GitHub App 的 `Expire user access tokens` 设置直接套用到 OAuth App 上，两者的 Token 机制不同。

OAuth Token 仍可能因用户撤销授权、长时间未使用或泄露被平台撤销等原因失效。遇到 `401 Unauthorized` 时，应检查授权状态并重新登录；`403` 还可能意味着仓库权限不足、组织策略或 API 访问限制，不能一概归因于过期。具体规则见 [GitHub Token 失效与撤销说明](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation)。

重新登录前，先复制备份尚未保存的正文。已经成功提交到 GitHub 的文章可以重新读取，但不能保证所有未提交的本地草稿都不会丢失。

### 3. 登录卡住、跨域与 404 错误

如果登录一直停留在“正在登录……”，可以按 `F12` 检查控制台和网络请求：

- `base_url` 应是 Worker 的 HTTPS 根地址，`auth_endpoint` 应对应实际登录路由，例如 `auth`。
- GitHub OAuth App 的回调地址应与 Worker 的 `/callback` 一致。
- `CMS_ORIGIN` 应是 CMS 页面实际所在的来源，不要误填成 Worker 地址。
- 检查 Worker 变量名称、大小写，以及登录弹窗是否被浏览器阻止。
- 检查 `state` 校验 Cookie 是否有效，回调页与主窗口的消息握手和来源校验是否成功。
- 如果 `/admin/` 或配置请求返回 404，先检查静态目录是否部署正确；如果只有图片 404，再检查媒体路径和站点 `base`。

OAuth 登录主要依赖弹窗跳转和消息回传，不是给所有响应加上 `Access-Control-Allow-Origin: *` 就能修好。自定义 `/publish` 接口则需要正确处理指定来源的 CORS 预检。

### 4. 本地开发与预览

调试 CMS 编辑功能时，可以在本地专用配置中添加：

```yaml
local_backend: true
```

在对应的本地仓库根目录运行：

```bash
npx decap-server
```

另开终端启动 Astro 开发服务器，再从本地 `/admin/` 进入。这样可以绕过远程 OAuth，在本地测试内容编辑；操作会影响本地文件，发布到远程仍需要提交代码并完成部署。

如果文章与网站分属两个仓库，本地代理的启动目录也必须与 CMS 的 `folder`、媒体目录相匹配。该模式用于本地调试，不能验证线上 OAuth 回调是否正确；生产配置应关闭本地代理模式。参见 [Decap 本地代理文档](https://decapcms.org/docs/decap-proxy/)。

## 结语

通过 **GitHub Pages + Decap CMS + Cloudflare Workers** 的组合，可以在免费额度内，搭建一个支持在线编辑与自动发布的博客系统。

它保留了静态网站的部署方式和 Git 的版本管理能力，同时把写作入口搬进浏览器。理解认证、内容提交和网站部署之间的边界，才能让这套方案既方便使用，也便于后续维护和排查问题。
