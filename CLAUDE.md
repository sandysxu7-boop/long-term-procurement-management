# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

长周期采购项目动态监测系统。前端为单文件 GitHub Pages 静态站点，数据存储于企业自托管 Teable，通过阿里云 FC3 函数做 SSO 认证中转和 Teable API 代理。

- **前端地址**: `https://sandysxu7-boop.github.io`
- **FC 后端地址**: `https://sso-backend-vaxetvknoh.cn-hangzhou.fcapp.run`
- **Teable 实例**: `https://yach-teable.zhiyinlou.com`（知音楼企业自托管，不是 app.teable.io）

## 代码结构

整个前端是 `index.html` 一个文件（~1600 行），含 HTML / CSS / JS。FC 后端在独立路径：

```
index.html                          # 完整前端（GitHub Pages 部署）
/Users/tal/.claude/skills/company-sso-login/fc-backend/
  index.js                          # FC3 Node.js 函数（阿里云 Function Compute）
  package.json
/tmp/sso-fc-update/
  index.js                          # 部署用（从 fc-backend/ 复制过来）
  s.yaml                            # Serverless Devs 部署配置
```

## 部署命令

**前端**（push 到 main 即自动发布到 GitHub Pages）：
```bash
git add index.html && git commit -m "..." && git push
```

**FC 后端**（修改 fc-backend/index.js 后）：
```bash
cp /Users/tal/.claude/skills/company-sso-login/fc-backend/index.js /tmp/sso-fc-update/index.js
cd /tmp/sso-fc-update && s deploy -a default -y
```

## 架构要点

### SSO 登录流程

1. `initSSO()` — 检查 localStorage JWT → 有则直接走 `checkPermission`
2. URL 有 `?token=` 参数时，调 FC 后端 `GET /auth/callback?token=xxx`
3. FC 后端向 `sso.100tal.com` 验证 token，颁发内部 JWT（含 `account_id`, `name`, `workcode`, `email`）
4. JWT 存入 `localStorage['lcpm_sso_jwt']`，TTL 2 小时
5. `checkPermission()` 查权限表，对应 `approved` 才进入系统

### Teable API 代理

所有 Teable 请求不直接从前端发出，而是走 FC 后端代理（携带内部 JWT 进行身份验证）：

| 前端路径 | 说明 |
|---------|------|
| `FC_BASE/teable/data/*` | 主数据表（只读 PAT，主表用） |
| `FC_BASE/teable/perm/*` | 权限表（全权 PAT，权限表用） |

前端 Teable 操作通过 `teableGet/teablePatch/teablePost/teableDelete` 四个封装函数。

### FC3 事件格式关键点

FC3（nodejs18 HTTP 触发器）的事件 `v1` 格式中：
- **HTTP 方法在 `req.requestContext.http.method`**，`req.method` 和 `req.httpMethod` 均为 undefined
- CORS preflight 通过 `Access-Control-Request-Method` 请求头检测（FC3 不可靠地传递 OPTIONS method）

### 权限分层（3级）

权限信息存储在 Teable 权限表（`tbldOAnHF4nSNj6kMb1`），通过 `角色` 和 `备注` 两个字段编码：

| 权限级别 | 角色字段 | 备注字段格式 | 可见数据 |
|---------|---------|------------|---------|
| 超级管理员 | `管理员` | `工号`（无 `\|`） | 所有项目，权限管理面板 |
| 组管理员 | `管理员` | `工号\|项目XX组` | 本组全部项目 |
| 普通采购员 | `普通用户` | `工号` | 仅自己负责项目（buyerTitle 末位工号匹配） |

- `isSuperAdmin()` / `isGroupAdmin()` / `canManageRecord(r)` 是核心权限判断函数
- 数据加载后 `loadAllRecords()` 内对非超管用户执行 `all.filter(canManageRecord)` 过滤
- `采购员` 字段格式为 `{title: "姓名-工号"}`，用 `.split('-').pop()` 取工号匹配

### 关键数据

| 名称 | 值 |
|------|---|
| 主数据表 ID | `tblwotOBEtoOOchp8Sj` |
| 权限表 ID | `tbldOAnHF4nSNj6kMb1` |
| Bootstrap 超管 account_id | `97ccf113-02b4-1080-98db-5ba2ff0416c8`（徐明阳的 SSO UUID） |
| 权限表所在 Space | `spcWqVp3nfAwPG0dN0W`（"采购四组"，用户是 owner） |
| 主数据表所在 Space | 集团采购空间（用户是 viewer，**不能创建字段**） |

## 主数据表字段

| 字段 | 类型 | 说明 |
|------|------|------|
| 项目组 | Single select | `项目一组` / `项目二/三组` / `项目四组` |
| 一级品类 | Single select | 网络通讯/物业服务/人力外包/印刷制品/办公用品/仓储物流 |
| 项目进度 | Single select | `待启动` / `寻源中` / `✅️完成签约` |
| 项目名称 | Text | |
| 采购员 | User reference | `{id, email, title, avatarUrl}`，title 格式 `姓名-工号` |
| 预算/合同金额（单位：元）| Number | |
| 合同起始/到期日期 | Date | ISO 格式 |
| 检索ID | Text | 合同编号 |
| 辅助列-到期前进展 | Single select | 只读，返回数组 |

## 权限管理操作（超管）

在"权限管理"面板中：
- 审核新用户：批准 / 拒绝
- 设置角色：下拉选择 `超级管理员 / 项目一组管理员 / 项目二/三组管理员 / 项目四组管理员 / 普通用户`
- 设置组管理员时，系统将 `备注` 字段更新为 `workcode|管辖组`（`adminSetRole` 函数）

## FC 后端路由

| 路由 | 说明 |
|------|------|
| `GET /health` | 健康检查 |
| `GET /auth/callback?token=xxx` | SSO token 验证，返回内部 JWT |
| `* /teable/data/*` | 代理到 Teable（用主数据 PAT，需内部 JWT） |
| `* /teable/perm/*` | 代理到 Teable（用权限表 PAT，需内部 JWT） |

FC 后端环境变量在 `/tmp/sso-fc-update/s.yaml` 中维护（含所有 token 和密钥）。
