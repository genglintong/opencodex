# macOS Menubar 挂件技术栈选型调研

> 调研日期：2026-07-24
> 目的：为 opencodex menubar 挂件选择最佳技术栈
> 约束：前期仅 macOS，对齐前沿实现，现有 GUI 是 React + Vite + TypeScript

## 候选方案对比

| 维度 | Tauri v2 (Tray) | SwiftUI MenuBarExtra | Electron Tray | Wails v3 |
|------|-----------------|---------------------|---------------|-----------|
| **包体积** | ~3-8 MB | ~1-2 MB | ~150-200 MB | ~10-15 MB |
| **内存占用** | ~30-60 MB | ~15-30 MB | ~200-400 MB | ~40-80 MB |
| **启动速度** | <100ms | <50ms | ~1-2s | <200ms |
| **macOS 原生体感** | 良好（NSPopover + vibrancy） | 完美（纯原生） | 一般（Chromium 渲染） | 良好（WKWebView） |
| **React/TS 复用** | ✅ 直接复用现有组件 | ❌ 需重写为 SwiftUI | ✅ 直接复用 | ✅ 直接复用 |
| **与 Bun 后端桥接** | HTTP 调用管理 API | HTTP 调用管理 API | HTTP 调用管理 API | HTTP 调用管理 API |
| **自动更新** | ✅ tauri-updater 内置 | 需 Sparkle/自建 | ✅ electron-updater | ✅ 内置 |
| **社区活跃度** | ⭐ 90k+ stars，非常活跃 | Apple 官方，稳定 | ⭐ 115k+，成熟 | ⭐ 25k+，成长中 |
| **开发效率** | 高（Web 前端 + Rust 后端） | 中（需学 SwiftUI） | 高（纯 Web） | 高（Web + Go） |
| **代码签名/公证** | 支持 | 支持 | 支持 | 支持 |

## 各方案详细分析

### Tauri v2（推荐）

**优势：**
- Tray Icon API 成熟，支持 macOS NSStatusItem + NSPopover
- 前端直接用 React/Vite，可复用现有 GUI 组件（QuotaBars、格式化函数等）
- 包体积极小（~5MB），内存占用低
- 内置 tauri-updater，支持 GitHub Releases 自动更新
- Rust 后端可做本地计算（QPS 聚合、延迟统计）
- 2025-2026 年生态已非常成熟，plugin 丰富（notification、global-shortcut、autostart）
- 支持 macOS vibrancy/毛玻璃效果（window-vibrancy plugin）

**劣势：**
- 需要 Rust 工具链（但后端逻辑简单，主要是 HTTP 调用）
- 原生体感不如纯 SwiftUI（但 95% 用户无法区分）
- macOS 权限（辅助功能等）处理比原生稍复杂

### SwiftUI MenuBarExtra

**优势：**
- 完美原生体验，系统级动画和毛玻璃
- 包体积最小，内存最低
- 无需 Web 运行时

**劣势：**
- 无法复用现有 React 组件，UI 需完全重写
- 需要 Swift/Objective-C 开发经验
- 与 TypeScript 生态割裂，维护成本高
- 自动更新需额外集成 Sparkle
- 仅 macOS，未来跨平台需完全重写

### Electron Tray

**优势：**
- 生态最成熟，文档丰富
- 完全复用现有 React 代码
- 跨平台一致

**劣势：**
- 包体积巨大（~200MB），作为 menubar 挂件不可接受
- 内存占用高（~300MB），常驻后台体验差
- 启动慢
- 2026 年已不推荐用于轻量工具类应用

### Wails v3

**优势：**
- Go 后端 + Web 前端，包体积适中
- 支持 macOS tray
- 可复用 React 组件

**劣势：**
- 社区活跃度不如 Tauri
- macOS tray 支持相对较新，成熟度待验证
- Go 后端与现有 TypeScript 生态有距离
- 自动更新方案不如 Tauri 成熟

## 最终推荐：Tauri v2

**理由：**

1. **技术栈契合**：前端直接复用 React + Vite + TypeScript，现有 GUI 的 QuotaBars、format-tokens、format-bytes 等组件和工具函数可以直接迁移
2. **轻量常驻**：~5MB 包体积 + ~40MB 内存，适合作为 menubar 常驻工具
3. **原生体感足够好**：NSPopover + vibrancy 效果，与系统原生挂件视觉一致
4. **自动更新内置**：tauri-updater 直接对接 GitHub Releases，与 opencodex 现有更新通道对齐
5. **生态成熟**：2026 年 Tauri v2 已稳定，tray/notification/autostart/global-shortcut 等 plugin 齐全
6. **未来可扩展**：如果后续需要 Windows/Linux 支持，Tauri 天然跨平台

**架构建议：**

```
menubar/                    # monorepo 内新目录
├── src-tauri/              # Rust 后端（tray 管理、HTTP 客户端、本地计算）
│   ├── src/main.rs
│   └── tauri.conf.json
├── src/                    # React 前端（复用 gui/ 的组件和工具）
│   ├── App.tsx             # 面板主组件
│   ├── components/         # 从 gui/src/components 迁移/共享
│   └── hooks/              # 数据轮询 hooks
├── package.json
└── vite.config.ts
```

**与管理 API 的通信：**
- 纯 HTTP 轮询（5-10s 间隔），调用现有 `/api/settings`、`/api/provider-quotas`、`/api/combos`、`/api/usage` 端点
- 认证：首次启动时配置 API token，存储在 Tauri 的 secure store 中
