# AGENTS.md — 子美个人作品集（Zimei Portfolio）开发指令

> 本文件是仓库内所有代码代理（Codex 等）与开发者的工作指令。**动工前必须先通读 [PRD.md](PRD.md) 与 [TECH_DESIGN.md](TECH_DESIGN.md)**，本文档是两者的可执行摘要。

## 项目概述

- **一句话定位**：一个部署在 Vercel、后台用 Sanity 的暗黑科技感个人作品集，传图填表即可更新作品，换色板即可换风格。
- **技术栈**：前端 Next.js 16（App Router）+ React 19 + TypeScript；样式 Tailwind CSS v4（CSS-first，设计令牌）；内容后台/数据库 Sanity（Studio v3 + Content Lake）；部署 Vercel。
- **架构要点**：内容与代码分离——所有内容由子美在 Sanity Studio 可视化后台维护，代码只负责渲染；前台以 SSG/ISR 静态化为主，后台发布后经 Webhook → `/api/revalidate` → `revalidateTag` 按标签失效缓存（目标 1–5 分钟生效）。
- **页面范围**：首页（Hero + 置顶作品 + 时间线 + 联系 CTA）、作品列表、作品详情（混合作品类型差异化呈现）、关于我、联系、404。当前不做博客。
- **内容模型**：`site`（单例）、`project`（含 type/status 枚举）、`timelineEntry`，字段与校验见 TECH_DESIGN.md 第 3 节。

## 开发规范

- **组件风格**：函数式组件 + Hooks；默认 Server Component，仅交互必要的叶子组件加 `"use client"`。
- **类型安全**：TypeScript 严格模式；Sanity schema 类型由 `npx sanity typegen generate` 生成到 `types/sanity.d.ts`，禁止手改；改了 schema 必须重新生成类型并跑 `npm run typecheck`。
- **样式规范**：一律使用 Tailwind v4 + 设计令牌（`app/globals.css` 的 `@theme`），禁止硬编码颜色值；换肤只改令牌。
- **组件复用**：组件放 `components/`，按业务域分组（`layout/`、`home/`、`projects/`、`ui/`、`portable-text/`）；通用原子组件收敛到 `ui/`。
- **数据访问**：所有 GROQ 查询收敛到 `lib/sanity/queries.ts`，页面组件不得内联查询字符串；查询统一带缓存标签并过滤 `status == "published"`。
- **注释要求**：关键逻辑、复杂组件、GROQ 查询、环境变量用途必须写中文注释；代码结构清晰优先于注释数量。
- **错误处理**：页面配套 `error.tsx` / `loading.tsx` / `not-found.tsx`，不允许裸报错。
- **命名**：组件/文件 PascalCase，函数/变量 camelCase，Sanity schema 字段 snake_case。
- **提交前**：`npm run lint` + `npm run typecheck` 必须通过；提交信息用 `feat/fix/docs/chore` 前缀；push 到 GitHub 需提权确认。

## 设计要求

- **深色主题**：背景与文字一律使用设计令牌（`--color-bg-base` 当前 `#05070D`，接近纯黑 `#0a0a0a`；`--color-text-primary` 当前 `#E6EDF3`，接近纯白 `#ffffff`）；色值以 PRD/TECH_DESIGN 设计令牌为准，不单独定义。
- **渐变强调色**：主强调色 `--color-accent-cyan`（`#22D3EE`）→ `--color-accent-violet`（`#8B5CF6`）的渐变用于 CTA、标题强调与 hover 发光，单屏不超过两个强调色。
- **平滑滚动动画**：克制的 fade + 上移入场、卡片 hover 发光上浮，全部用 CSS 实现；必须尊重 `prefers-reduced-motion: reduce`，禁用非必要动画。
- **移动端适配**：响应式断点手机 < 640px / 平板 640–1024px / 桌面 > 1024px；卡片 1/2/3 列；移动端无横向滚动、正文不小于 16px。
- **简洁克制**：不做过度设计，不引入重型动画库；内容优先，装饰不得影响可读性；对比度满足正文 ≥ 4.5:1。

## 注意事项

- **性能**：图片必须走 `next/image` + Sanity CDN 懒加载，容器固定 `aspect-ratio`（防 CLS）；预算：LCP < 2.5s、首页核心 JS < 200KB gzip、Lighthouse ≥ 90（移动端）。
- **链接**：所有链接可点击且语义正确；外链加 `target="_blank" rel="noopener"`；图片必须有 alt（装饰图用空 alt）。
- **安全**：`.env*` 不提交；`NEXT_PUBLIC_*` 只放公开信息，密钥只进 Vercel 环境变量；富文本用 Portable Text 白名单渲染，禁止 raw HTML 注入。
- **内容更新链路**：改动 Sanity schema 或查询标签时，必须同步检查 Webhook 重验证逻辑（`app/api/revalidate/route.ts`），确保"后台保存 → 前台可见"不失效。
- **线上内容**：子美通过 Sanity Studio 维护内容；开发不得直接修改线上数据，除非用户明确要求。
- **文档同步**：改动架构、schema、目录结构时，同步更新 TECH_DESIGN.md；改动产品范围时同步 PRD.md。
