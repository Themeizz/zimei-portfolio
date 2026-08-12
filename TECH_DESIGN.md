# 子美个人作品集（Zimei Portfolio）· 技术设计文档

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 技术设计文档（Technical Design） |
| 文档版本 | v1.0 |
| 更新日期 | 2026-08-12 |
| 文档状态 | 待评审 |
| 关联文档 | [PRD.md](PRD.md)（产品需求文档 v1.0）、[docs/需求文档.md](docs/需求文档.md) |

> 本文档把 PRD 转化为可落地的技术方案：技术栈、项目结构、数据模型、关键技术点。版本信息以 2026-08 为基准，实际安装时以 `package.json` 锁定的版本为准。

---

## 1. 技术栈选择

### 1.1 架构总览

本项目是**内容型静态站点**：前台用 Next.js 做页面渲染，内容托管在 Sanity（云端 CMS / 内容数据库），部署在 Vercel（CDN + Serverless）。

```
┌──────────────────────────────────────────────────────────────┐
│ 内容生产（子美，技术小白）                                      │
│  Sanity Studio 可视化后台（/studio）：填表、传图、发布作品       │
└───────────────────────────┬──────────────────────────────────┘
                            │ 保存 / 发布
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ Sanity Content Lake（内容数据库 + 图片 CDN）                    │
│  结构化内容（site / project / timelineEntry）                   │
└────────────┬──────────────────────────────────┬───────────────┘
             │ GROQ 查询（只读 API）              │ 更新事件（Webhook）
             ▼                                  ▼
┌──────────────────────────┐      ┌─────────────────────────────┐
│ Next.js 16（App Router）  │      │ /api/revalidate（Serverless）│
│ SSG / ISR 渲染页面         │◀─────│ 验签后按标签失效缓存           │
└────────────┬─────────────┘      └─────────────────────────────┘
             │ 静态产物 / 增量生成
             ▼
┌──────────────────────────────────────────────────────────────┐
│ Vercel（HTTPS + CDN + 自动部署 + Serverless API）              │
│  访客访问；图片由 Sanity CDN 分发                               │
└──────────────────────────────────────────────────────────────┘
```

**更新生效链路（核心）**：子美在后台保存 → Sanity 触发 Webhook → Next.js 的 `/api/revalidate` 验签后调用 `revalidateTag()` → 相关页面下次访问时重新生成 → CDN 缓存失效。目标：后台保存后 1–5 分钟内前台可见（对应 PRD A12）。

### 1.2 技术栈清单

| 层级 | 选型 | 当前版本（2026-08） | 职责 |
| --- | --- | --- | --- |
| 前端框架 | Next.js（App Router）+ React 19 | 16.x | 页面渲染、路由、SSG/ISR、Server Components |
| 开发语言 | TypeScript | 5.x | 类型安全，前后端与内容类型对齐 |
| 样式 | Tailwind CSS + CSS 变量 | v4（CSS-first 配置） | 原子样式 + 设计令牌（换肤） |
| 内容后台 / "后端" | Sanity（Sanity Studio + Content Lake） | Studio v3 | 可视化后台、内容存储、内容 API、图片 CDN |
| 数据查询 | GROQ + next-sanity | next-sanity 9.x | 前台读取内容 |
| 图片 | Sanity Image CDN + next/image | — | 自动压缩、格式转换、懒加载 |
| 部署 / Serverless | Vercel | — | CI/CD、HTTPS、CDN、API Routes |
| 代码托管 | GitHub | — | 版本管理、自动部署触发源 |
| 统计（可选） | Umami / Plausible | — | 匿名访问统计 |

### 1.3 明确"前端 / 后端 / 数据库"分别是什么

- **前端**：Next.js 16（App Router）。所有页面默认是 Server Component，在服务器上取数、渲染成静态 HTML，再交给 CDN。
- **后端**：**不搭建自建后端**。内容侧的后端能力由 Sanity 云端承担（后台、鉴权、内容 API）；Vercel Serverless Functions 只承担少量接口：`/api/revalidate`（内容更新重验证）和未来的联系表单（`/api/contact`，P2）。
- **数据库**：**Sanity Content Lake**（结构化内容 + 图片资产库）。本地不部署数据库，不维护数据库服务器。

这种"托管后台 + 静态前台"模式对个人作品集的收益：零运维、几乎零成本、天然静态化加载快、内容可迁移；代价是内容存在第三方（用导出备份对冲，见 4.12）。

### 1.4 为什么不自建后端 + 自建数据库

| 方案 | 优点 | 缺点 | 结论 |
| --- | --- | --- | --- |
| Sanity（本项目） | 可视化后台现成、CDN 现成、免费额度够、无需运维 | 依赖第三方、schema 受其约束 | ✅ 采用 |
| Next.js API + Prisma + Postgres | 完全自主、关系型能力强 | 要自建后台 UI、要维护数据库、要处理鉴权与部署成本 | ❌ 对个人作品集过重 |
| 纯 Markdown 文件 | 极简、Git 管理 | 没有可视化后台（违背 PRD 决策） | ❌ 不符合需求 |

---

## 2. 项目结构

### 2.1 目录树

```
zimei-portfolio/
├─ app/                                  # Next.js App Router（页面）
│  ├─ layout.tsx                         # 根布局：字体、主题、全局 metadata、Header/Footer
│  ├─ page.tsx                           # 首页（Hero + 置顶作品 + 时间线 + 联系 CTA）
│  ├─ globals.css                        # Tailwind v4 入口 + 设计令牌（@theme）
│  ├─ not-found.tsx                      # 404
│  ├─ sitemap.ts                         # 自动生成 sitemap.xml
│  ├─ robots.ts                          # robots.txt
│  ├─ projects/
│  │  ├─ page.tsx                        # /projects 作品列表
│  │  └─ [slug]/
│  │     ├─ page.tsx                     # /projects/:slug 作品详情
│  │     ├─ generateStaticParams         # 静态生成参数（与 page 同文件或同目录）
│  │     └─ not-found.tsx                # 作品不存在
│  ├─ about/page.tsx                     # /about 关于我（P1）
│  ├─ contact/page.tsx                   # /contact 联系
│  ├─ studio/[[...tool]]/page.tsx        # 嵌入的 Sanity Studio（可独立部署替代）
│  └─ api/
│     └─ revalidate/route.ts             # Sanity Webhook → 按标签失效缓存
├─ components/
│  ├─ layout/                            # Header、Footer、Navigation
│  ├─ home/                              # Hero、FeaturedProjects、TimelineSection、ContactCTA
│  ├─ projects/                          # ProjectCard、ProjectGrid、ProjectDetail、FilterBar(P1)
│  ├─ ui/                                # Button、Tag、Section、Container、GlowCard
│  └─ portable-text/                     # Portable Text 渲染映射（代码块、图片等）
├─ lib/
│  └─ sanity/
│     ├─ client.ts                       # 只读 Sanity 客户端（fetch 封装 + tag 缓存）
│     ├─ queries.ts                      # 全部 GROQ 查询集中管理
│     ├─ image.ts                        # image URL builder（尺寸/格式/焦点）
│     └─ revalidate.ts                   # Webhook 验签与标签映射
├─ sanity/
│  ├─ schema/
│  │  ├─ index.ts                        # schema 注册表
│  │  ├─ site.ts                         # 站点配置（单例）
│  │  ├─ project.ts                      # 作品
│  │  └─ timelineEntry.ts                # 时间线条目
│  ├─ lib/                               # Studio 侧专用工具（如需）
│  └─ structure.ts                       # Studio 侧边栏结构（单例/列表分组）
├─ sanity.config.ts                      # Studio 配置（项目 ID、dataset、schema、可视化编辑）
├─ sanity.cli.ts                         # CLI：typegen、dataset export/import
├─ types/
│  └─ sanity.d.ts                        # 由 sanity typegen 生成的类型（不手写）
├─ public/                               # 静态资源（favicon、og 默认图等）
├─ .env.example                          # 环境变量模板（提交仓库）
├─ .env.local                            # 本地密钥（不提交）
├─ next.config.ts
├─ package.json
├─ README.md                             # 运维文档：改内容/改样式/部署（PRD A10）
├─ PRD.md
└─ TECH_DESIGN.md
```

### 2.2 目录职责与规则

| 目录 | 职责 | 规则 |
| --- | --- | --- |
| `app/` | 路由与页面 | 数据获取只发生在 Server Component；客户端交互组件按需 `"use client"` |
| `components/` | 展示组件 | 按业务域分目录；UI 原子组件放 `ui/`；默认 Server Component |
| `lib/sanity/` | 数据访问层 | 所有 GROQ 查询收敛于此，页面不直接写查询字符串 |
| `sanity/` | 内容后台 schema | schema 与前端类型一一对应；字段校验在 schema 层定义 |
| `types/` | 生成类型 | `sanity typegen` 自动生成，禁止手改 |
| `public/` | 静态资源 | 不放用户内容（作品图片一律走 Sanity CDN） |

### 2.3 路由与页面映射

| 路由 | 页面 | 数据来源 | 渲染策略 |
| --- | --- | --- | --- |
| `/` | 首页 | site + project(featured) + timelineEntry | SSG + tag 重验证 |
| `/projects` | 作品列表 | project 全量（published） | SSG + tag 重验证 |
| `/projects/:slug` | 作品详情 | project 单个 | SSG（generateStaticParams）+ tag 重验证 |
| `/about` | 关于我 | site | SSG |
| `/contact` | 联系 | site（邮箱/社交） | SSG |
| `/studio` | 可视化后台 | Sanity 鉴权 | 客户端应用（独立构建） |
| `/api/revalidate` | Webhook | POST 请求 | Serverless |
| `/404` | 未找到 | — | 静态 |

### 2.4 组件划分

- **布局组件**：Header（导航 + 主题标识）、Footer（版权 + 社交 + 统计声明）；
- **首页组件**：Hero（姓名/简介/技能标签/CTA）、FeaturedProjects（3–5 个精选卡片）、TimelineSection、ContactCTA；
- **作品组件**：ProjectCard（封面 16:10、标题、summary、标签、日期）、ProjectGrid（响应式 1/2/3 列）、ProjectDetail（按 type 差异化组合）；
- **通用 UI**：Button、Tag、Section、Container、GlowCard（发光卡片，hover 描边 + 上浮）；
- **富文本**：portable-text 组件映射（标题、段落、列表、引用、代码块、图片、链接）。

### 2.5 代码规范

- 命名：组件/文件 PascalCase，函数 camelCase，schema 字段 snake_case（Sanity 惯例）；
- 边界：默认 Server Component，仅交互必要的叶子组件加 `"use client"`；
- 数据获取：只在 Server Component 用 `sanityFetch`，客户端不直连 Sanity；
- 错误处理：`error.tsx` + `loading.tsx` + `not-found.tsx` 三件套，页面不裸奔；
- 格式：Prettier + ESLint（Next 内置），提交前跑 `npm run lint`；
- 提交：常规提交信息（feat/fix/docs/chore）。

---

## 3. 数据模型

### 3.1 设计原则

- **Schema 即后台表单**：字段类型、校验、必填在 Sanity schema 定义，后台自动生成对应输入控件；
- **类型生成**：用 `sanity typegen` 从 schema 生成 TypeScript 类型，前端与内容强一致；
- **枚举收敛**：`type`、`status` 等用 enum，避免自由文本造成脏数据；
- **内容与展示解耦**：schema 只描述内容，展示逻辑由前端组件负责。

### 3.2 `site`（单例，全站配置）

| 字段 | 类型 | 必填 | 说明 / 校验 |
| --- | --- | --- | --- |
| name | string | 是 | 站点/作者名称 |
| headline | string | 是 | 一句话简介（Hero） |
| bio | string | 否 | 详细介绍 |
| avatar | image | 否 | 头像（需 alt） |
| email | string | 是 | 邮箱，格式校验 |
| social | object | 否 | github、weibo、email 等链接，URL 校验 |
| nav | array | 否 | 导航项（label + href） |
| footer | object | 否 | 版权文案、统计声明 |
| theme | object | 否 | 预留：后台可覆盖主题色（P2） |

### 3.3 `project`（作品）

| 字段 | 类型 | 必填 | 说明 / 校验 |
| --- | --- | --- | --- |
| slug | slug | 是 | 唯一；由标题生成可改；小写字母/数字/连字符 |
| type | enum | 是 | `code` / `design` / `writing` / `other` |
| title | string | 是 | 1–100 字符 |
| subtitle | string | 否 | 副标题 1–200 字符 |
| summary | string | 是 | 卡片简介 1–300 字符 |
| cover | image | 是 | 封面，建议 16:10，alt 必填 |
| gallery | array(image) | 否 | 详情图集，每张 alt 必填 |
| video | url / file | 否 | 演示视频 |
| pdf | file | 否 | 文档，≤ 20MB |
| description | portable text | 是 | 正文块：标题/段落/列表/引用/代码块/图片/链接 |
| tags | array(string) | 否 | 展示标签，单条 ≤ 20 字符 |
| tech | array(string) | 否 | 技术栈（type=code 时重点展示） |
| links | object | 否 | demo / github / article，URL 校验 |
| date | date | 是 | 排序依据 |
| featured | boolean | 否 | 置顶精选，默认 false |
| status | enum | 否 | `draft` / `published` / `hidden`，默认 draft |
| order | number | 否 | 手动排序，越小越靠前 |

**排序规则（对应 PRD FR-08）**：列表 `order asc` → `date desc`；首页置顶取 `featured && published` 前 3–5 个。

### 3.4 `timelineEntry`（时间线）

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| date | date | 是 | 事件时间 |
| title | string | 是 | 事件标题 |
| org | string | 否 | 组织/公司/学校 |
| type | enum | 是 | `work` / `education` / `event` |
| description | portable text | 否 | 描述 |
| links | object | 否 | 相关链接 |

### 3.5 类型生成

- 运行 `npx sanity typegen generate`（由 `sanity.cli.ts` 驱动），把 schema 生成到 `types/sanity.d.ts`；
- GROQ 查询建议配合类型化查询（`sanity-typed-queries` / groqd），让查询结果类型可推断；
- 开发规范：改了 schema 必须重新生成类型并跑 `npm run typecheck`，杜绝"内容类型与前端类型漂移"。

### 3.6 GROQ 查询示例

```groq
// 首页：站点配置 + 置顶作品 + 最近时间线
*[_type == "site"][0]
*[_type == "project" && status == "published" && featured == true] | order(order asc, date desc)[0...5]
*[_type == "timelineEntry"] | order(date desc)[0...5]

// 作品列表
*[_type == "project" && status == "published"] | order(order asc, date desc)

// 作品详情
*[_type == "project" && slug.current == $slug && status == "published"][0] {
  _id, title, type, summary, cover, gallery, video, pdf,
  description, tags, tech, links, date, "slug": slug.current
}
```

---

## 4. 关键技术点

### 4.1 内容更新生效链路（最重要的技术点）

**目标**：后台保存后 1–5 分钟前台可见（PRD A12），同时保持静态化性能。

**方案：tag-based on-demand revalidation（按需重验证）**

1. 所有 `sanityFetch` 查询用 `next: { tags: [type] }` 打标签（如 `project`、`site`、`timelineEntry`）；
2. Sanity 项目配置 **GROQ Webhook**：监听内容变更，POST 到 `https://<域名>/api/revalidate`，携带请求头 `x-sanity-webhook-signature`；
3. `/api/revalidate` 校验签名（防止伪造请求），再按变更的 `_type` 调 `revalidateTag(type)`；
4. 已缓存页面在下次访问时按标签重建，其余页面保持 CDN 命中。

```ts
// app/api/revalidate/route.ts（摘要）
import { revalidateTag } from "next/cache";
import { isValidSignature } from "@sanity/webhook";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get("x-sanity-webhook-signature") ?? "";
  const secret = process.env.SANITY_REVALIDATE_SECRET ?? "";

  if (!(await isValidSignature(rawBody, signature, secret))) {
    return NextResponse.json({ message: "Invalid signature" }, { status: 401 });
  }

  const body = JSON.parse(rawBody) as { _type?: string };
  if (body._type) revalidateTag(body._type); // project / site / timelineEntry
  return NextResponse.json({ revalidated: true });
}
```

**注意事项**

- 验签必须用**原始请求体**（不能先 JSON.parse 再比对）；
- `revalidateTag` 只在下次访问时重建，不是主动推送；若需要更即时体验，可叠加 Sanity Visual Editing / 预览模式（P1 可选）；
- `site` 变更要单独重验证（导航/页脚会影响所有页面）；
- slug 修改后旧 URL 会 404：详情页 `not-found.tsx` 提供返回入口，必要时用重定向或别名（P2）；
- 兜底策略：`export const revalidate = 60`（ISR）作为 webhook 失效时的第二道保险，避免长期不更新内容。

### 4.2 图片优化与 CDN

- 作品图片一律存 Sanity 图片资产（自带 CDN、版本管理、裁剪元数据）；
- `next/image` 配置 `remotePatterns` 只允许 `cdn.sanity.io`，防止开放代理滥用（PRD NFR-S5）；
- 用 `next-sanity` 的 image URL builder 输出 `w`、`q=75-80`、`auto=format`、`fit` 参数；封面统一 16:10 裁剪；
- 每张图必填 alt（schema 校验），装饰图用空 alt；
- 布局稳定性：容器固定 `aspect-ratio`，避免 CLS（PRD NFR-P2）；
- 懒加载：`next/image` 默认 lazy；首屏图 `priority`。

### 4.3 Portable Text 渲染与安全

- 富文本正文用 `next-sanity` 的 `PortableText` 组件 + 自定义 `components` 映射（代码块高亮、图片、外链 `target="_blank" rel="noopener"`）；
- 安全：Portable Text 是结构化 JSON，渲染走白名单组件，React 默认转义文本，天然防 HTML 注入（PRD NFR-S4）；禁止引入 raw HTML 块；
- 代码块样式：暗黑主题下用等宽字体 + 语法高亮（如 `shiki` 或轻量方案，控制 JS 体积）。

### 4.4 暗黑主题与设计令牌

- Tailwind v4 用 `@theme` 定义设计令牌（对应 PRD 4.2 色板），生成 `bg-base`、`text-primary`、`text-accent-*` 等工具类：

```css
/* app/globals.css */
@import "tailwindcss";

@theme {
  --color-bg-base: #05070d;
  --color-bg-surface: #0b101c;
  --color-bg-elevated: #121a2b;
  --color-text-primary: #e6edf3;
  --color-text-secondary: #8b98a9;
  --color-text-muted: #5b6b82;
  --color-accent-cyan: #22d3ee;
  --color-accent-violet: #8b5cf6;
  --font-sans: var(--font-inter), var(--font-noto-sans-sc), system-ui, sans-serif;
  --font-mono: var(--font-jetbrains-mono), ui-monospace, monospace;
}
```

- **换肤 = 改 `@theme` 变量**（PRD FR-33）；浅色模式（P2）预留 `html[data-theme="light"]` 覆盖，用 `@theme inline` 引用运行时变量实现动态切换；
- 字体用 `next/font` 自托管（Space Grotesk / Inter / Noto Sans SC / JetBrains Mono），避免外部字体请求拖慢首屏；
- **深色 FOUC 预防**：默认就是深色，无切换开关，天然无闪烁；若未来做浅色切换，需在 `head` 内联脚本按系统偏好先设置 `data-theme`。

### 4.5 SEO 与社交分享

- 用 Next.js Metadata API：`layout.tsx` 设站点级默认，页面/作品用 `generateMetadata()` 动态生成；
- 默认规则（PRD FR-29）：`{作品名} - 子美` + 摘要 + 作品封面作为 OG 图；
- `sitemap.ts` + `robots.ts` 自动生成；canonical 指向正式域名；
- JSON-LD：首页输出 `Person`，作品页输出 `CreativeWork`（带 author、url、image）；
- 分享到微信/QQ 时 OG 图可能不生效：额外提供 `twitter:card` 与 favicon；微信需审核域名（后续如需要再处理）。

### 4.6 混合作品类型的差异化渲染

- 详情页按 `project.type` 组合不同区块，而不是一个组件里堆满条件分支：

```
<ProjectDetail project={project}>
  {project.type === "design" && <ImageGallery images={project.gallery} />}
  {project.type === "code" && <TechStack items={project.tech} />}
  <Links links={project.links} />
  <PortableText value={project.description} />
</ProjectDetail>
```

- 组件通过 props 组合，新增类型 = 新增一个区块组件，不改动核心布局；
- 列表卡片统一结构，保证视觉一致性。

### 4.7 环境变量与密钥安全

**变量清单（`.env.example`）：**

```bash
# 前台只读（NEXT_PUBLIC_ 会被打进浏览器，只能放公开信息）
NEXT_PUBLIC_SANITY_PROJECT_ID=your-project-id
NEXT_PUBLIC_SANITY_DATASET=production
NEXT_PUBLIC_SANITY_API_VERSION=2026-08-12

# 服务端专用（绝不带 NEXT_PUBLIC_，只配在 Vercel / .env.local）
SANITY_API_READ_TOKEN=sk_...          # 只读 token，前台 fetch 用
SANITY_REVALIDATE_SECRET=...          # webhook 验签

# Studio 端（嵌入 /studio 时）
SANITY_STUDIO_PROJECT_ID=your-project-id
SANITY_STUDIO_DATASET=production
```

规则：

- `.env.local` 与 `*.env` 已在 `.gitignore`，绝不提交（PRD NFR-S3）；
- `NEXT_PUBLIC_*` 只放非敏感内容；Sanity 写 token 永远只存在于 Studio 登录流程（用户浏览器 OAuth），不落在代码里；
- 在 Vercel 后台配置环境变量，不同环境（production/preview）可独立配置；
- 若 dataset 设为私有，前台必须用 `SANITY_API_READ_TOKEN`；公开 dataset 可省，但建议一律使用 token 以便审计。

### 4.8 性能预算

| 指标 | 目标 | 手段 |
| --- | --- | --- |
| LCP | < 2.5s | SSG/ISR、CDN、next/image、字体子集化 |
| JS 体积 | 首页 gzip < 200KB | Server Components 默认、路由级分包、不引入重型动画库 |
| CLS | < 0.1 | 固定 aspect-ratio、字体大小预留、图片尺寸显式 |
| Lighthouse | ≥ 90（移动端） | 以上全部 + 持续用 Vercel Analytics 观测 |

### 4.9 类型安全与数据一致性

- `sanity typegen` 生成 schema 类型，前端禁用 `any` 处理内容；
- slug 唯一性：schema 层做唯一校验提示；发布时 `generateStaticParams` 只取 published；
- status 过滤在**查询层**统一做（`status == "published"`），杜绝某个页面漏过滤导致草稿泄漏；
- 查询集中在 `lib/sanity/queries.ts`，方便统一加 tag 和过滤。

### 4.10 无障碍与减少动效

- 语义标签 + 键盘可达 + 焦点样式（PRD 4.7）；
- 动效全部走 CSS，`prefers-reduced-motion: reduce` 时禁用（PRD 4.5）；
- 对比度按 PRD 色板控制（正文 ≥ 4.5:1）。

### 4.11 部署与 CI

- Vercel 连接 GitHub 仓库：push `main` 自动构建发布，PR 自动生成预览链接（PRD FR-27）；
- 构建命令：`next build`；环境变量在 Vercel 项目设置注入；
- 可选 GitHub Actions：`lint + typecheck + typegen 校验`，作为合并门槛；
- dataset 规划：建议 `production`（正式）+ `development`（本地测试），避免误操作污染线上内容。

### 4.12 数据备份与恢复

```bash
# 备份（建议每月或大改动后手动执行；README 记录步骤）
npx sanity dataset export production ./backups/production-YYYYMMDD.tar.gz

# 恢复
npx sanity dataset import ./backups/production-YYYYMMDD.tar.gz production
```

- 代码与文档全部在 Git，天然可回溯；
- Sanity 面板可查看文档历史并回滚单条内容（PRD FR-13 / NFR-R4）。

### 4.13 国际化预留（P2）

- 目录预留 `locales/`（zh/en）；路由层可扩展 `[locale]` 前缀（或默认 zh 不占路径）；
- schema 文本字段预留 `{ en, zh }` 结构（按需迁移），先保持单语言不阻塞 MVP。

### 4.14 安全补充

- 全站 HTTPS（Vercel 默认）；
- 后台只允许 Sanity 认证用户访问（Studio 自身鉴权），前台无写接口；
- 未来联系表单：`/api/contact` 加 honeypot 字段 + 提交频率限制（PRD NFR-S7）；
- 依赖审计：`npm audit` 纳入发布前检查；Next.js 大版本升级前看迁移指南。

### 4.15 风险与备选方案

| 风险 | 对策 |
| --- | --- |
| Sanity 免费额度变化 | 定期导出备份；schema 简单，迁移成本可控 |
| Webhook 丢失导致内容不更新 | ISR `revalidate=60` 兜底 + 手动重验证入口 |
| Next.js 大版本升级成本 | 锁定版本 + 依赖升级节奏化；内容与代码解耦不受影响 |
| 后台误删作品 | 删除需确认 + Sanity 历史版本 + 定期导出 |
| 备选：若未来不需要可视化后台 | 可迁移到 Astro + Decap CMS（内容存 Git，见 PRD 5.4） |

---

## 5. 开发环境与脚本

### 5.1 package.json 脚本

| 脚本 | 命令 | 说明 |
| --- | --- | --- |
| dev | `next dev` | 本地开发（Turbopack） |
| build | `next build` | 生产构建（SSG/ISR 输出） |
| start | `next start` | 本地运行生产构建 |
| lint | `next lint` | 代码检查 |
| typecheck | `tsc --noEmit` | 类型检查 |
| typegen | `sanity typegen generate` | schema → 类型 |
| backup | `sanity dataset export ...` | 内容备份 |

### 5.2 本地启动步骤（摘要）

1. `npm install`；
2. 复制 `.env.example` 为 `.env.local`，填入 Sanity 项目信息；
3. `npx sanity@latest init`（首次：创建项目/dataset、生成 schema 骨架）；
4. `npm run dev`，访问 `http://localhost:3000`；后台 `http://localhost:3000/studio`。

---

## 6. 里程碑技术任务拆分

| 里程碑 | 技术任务 |
| --- | --- |
| M1 · MVP | 项目初始化（Next 16 + TS + Tailwind v4）；Sanity 接入与 schema（site/project/timelineEntry）；首页、作品列表、详情、联系；tag 重验证 webhook；Vercel 部署；README |
| M2 · v1.1 | 关于我页；标签筛选；图片/多媒体优化；SEO 全套（metadata/sitemap/JSON-LD）；自定义域名；内容导出备份；Studio 结构优化 |
| M3 · v2.0 | 浅色模式；联系表单（/api/contact）；国际化；简历页；版本历史入口文档；访问统计 |

---

## 7. 附录

### 7.1 术语表

| 术语 | 说明 |
| --- | --- |
| SSG / ISR | 静态站点生成 / 增量静态再生（Next.js 渲染策略） |
| GROQ | Sanity 的查询语言 |
| Portable Text | Sanity 富文本的结构化 JSON 格式 |
| Content Lake | Sanity 的内容存储与 API 服务 |
| Webhook | 内容变更时 Sanity 主动通知我们的回调 |
| Server Component | Next.js 在服务器渲染的组件，默认形式 |

### 7.2 参考文档

- Next.js：https://nextjs.org/docs
- Sanity + Next.js：https://www.sanity.io/docs/nextjs/introduction
- Sanity Webhook 重验证：https://www.sanity.io/docs/nextjs/validating-sanity-webhooks-nextjs
- Tailwind CSS v4：https://tailwindcss.com/docs
- Vercel：https://vercel.com/docs
