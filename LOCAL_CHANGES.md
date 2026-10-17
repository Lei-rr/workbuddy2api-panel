# 本地改动说明（fork: Lei-rr/workbuddy2api-panel）

> 本文件随 fork 仓库维护（官方 `.gitignore` 忽略 `*.md`，本文件用 `git add -f` 强制跟踪），
> 删掉本地源码副本后重新 clone 仍能读到。新会话接手时先读这一页，不要推翻已有决策。

## 基线

- 上游：`linguo2625469/workbuddy2api-panel`，基线 commit `6b31db6`（PR #152 合并后）
- 本地分支：`minimal2`，HEAD `31b5f1b`，已推到 `fork`（`Lei-rr/workbuddy2api-panel`）的 `main`
- 原则：**只做增量，尽量不删官方代码**；整池「积分构成」那套代码原样保留，只断入口

## 改动清单（相对 6b31db6）

| 文件 | 改动 | 原因 |
|---|---|---|
| `internal/panel/app.js` | `TRANGE_DEFAULT='today'`；账号池客户端筛选（搜索/状态/版本）+ 分页（50/100/200/全部）；二级 tab（`accTab`/`switchAccTab`/`expBoxShow`）；账号行「积分」按钮 + `pkVeil` 单号弹窗（复用官方 `pk*` 辅助函数）；「重新扫描」按钮事件；`TITLES` 去掉 `packages`；删 `btnExp` 绑定 | 1000+ 账号时页面可用性；整池逐号查上游有风控风险 |
| `internal/panel/index.html` | 筛选栏 `#accFilter`；`#accTabs` 二级 tab；`#accBox`/`#mlBox` id；`#pkVeil` 弹窗；`#btnRescan` 按钮；删导航「积分构成」项与到期卡「检查」按钮 | 同上 |
| `internal/panel/panel.go` | `packages` 收紧为 `?uid=` 必填（无 uid → 400，删除整池全量模式）；`packagesOne` 单号查询（未知 uid 404、上游失败 200+error）；`accountsRescan`（`auth.LoadDir` + `pool.SyncToDir`，扫到 0 文件而池非空时 409 拒绝） | 单号按需查；auths/ 目录热对齐 |
| `internal/panel/packages_test.go` | 新增：未知 uid 404、无 uid/空白 uid 一律 400 | 钉住单号模式 |
| `internal/panel/rescan_test.go` | 新增：0 文件保护拒绝、空池放行 | 防止挂载丢失清空整池 |
| `internal/panel/frontend_test.go` | +6 行：`expBoxShow` 桩（切片环境无 tab 状态）、`defaultPreset:"today"` 断言 | 测试跟随实现，防静默回退 |
| `.github/workflows/docker-ghcr.yml` | 构建平台仅 `linux/amd64`，去掉 QEMU | 部署机 x86_64，多架构每次多等 10 分钟 |

核心包 `internal/pool`、`server`、`upstream`、`scheduler`、`auth` **零改动**。

## 整池「积分构成」入口是怎么断的（三层，代码保留）

1. 导航删掉「积分构成」项
2. `TITLES` 去掉 `packages` → 直链 `#packages` 自动回落账号池，`loadPackages()` 永不被调用
3. 到期卡「检查」按钮及其绑定删除 → 第二个整池查询入口也封死

后端 `/panel/api/packages` 无 uid 直接 400，全量模式代码已删（`sort` import 一并移除）。

`renderPackages`/`loadExpiry`/`renderExpiry`/`view-packages`/`pk-expiry-*` CSS 仍在文件里，不可达。

## 部署链路

- 源码：`/guolei/workbuddy2api-panel-src`
- 部署：`/guolei/workbuddy2api-panel/docker-compose.yml`，镜像 `ghcr.io/lei-rr/workbuddy2api-panel:latest`（已从官方 linguo 镜像切过来）
- 发布：`git push fork HEAD:main` → GitHub Action（amd64，约 2.5 分钟）→ `docker pull` → `docker compose up -d`
- 注意：**fork 首次需 `PUT /repos/{owner}/{repo}/actions/workflows/{id}/enable`**，否则 push 不触发 workflow（本项目已 enable）
- 账号热更新：面板「重新扫描」按钮 = `auth.LoadDir` + `pool.SyncToDir`，1146 文件约 160ms；外部脚本往 `auths/` 丢文件后点一次即可

## 实测基线（1146 账号）

- `/panel/api/overview`：689 KB / 13 ms，前端 5s 轮询只渲染当前页 100 行
- 容器：CPU 0.04%，内存 16.4 MB
- `packages?uid=<真实>` 200 / 未知 404 / 无 uid 400
- `accounts/rescan`：`scanned=1146, added=0, removed=0`，160 ms

## 已知注意点

- 页面顶部「刷新」按钮 = `balance_all` → `RunBalanceRefreshNow()`，**对全池无并发上限地打上游**（1146 个并发请求）+ 昵称同步。后台 `balance_refresh_enabled=false`，手动点这个按钮反而更激进，建议不点。
- 积分靠 9/21 点签到更新（签到流程内含查余额 + 写回 `credits`/到期批次）；签到是**串行**执行，1146 号一轮约 1 小时。
- `state.json` 会落后于池（官方设计：新增账号不置 `dirty`，只有剔除才落盘），重启后按零值重建，不影响使用。
- `auths/` 里有 52 个文件属主是 `root:root`（其余 `10001`），容器可读不可写；若出现 token 刷新失败，先 `chown -R 10001:10001 auths`。

## 重建环境（本地源码副本丢失时）

```bash
git clone https://github.com/Lei-rr/workbuddy2api-panel.git
cd workbuddy2api-panel
git log --oneline -3          # 确认 HEAD 与本文档一致
```

部署目录只需 `docker-compose.yml`（镜像 `ghcr.io/lei-rr/workbuddy2api-panel:latest`，
容器名 `workbuddy2api-panel`，`7800:7863`，挂载 `./auths`、`./data`、`./config.json`，
`user: 10001:10001`，`TZ=Asia/Shanghai`）——`auths/`、`data/`、`config.json` 是运行数据，
必须保留，不在仓库里。

## 跟随上游

```bash
git fetch origin main
git rebase origin/main      # 预计冲突：TITLES 一行、renderAccounts 头尾、renderExpirySnapshot 末尾、index.html 账号池段、packages 函数
git push -f fork HEAD:main
```

rebase 时留意：若官方把 `loadExpiry` 挂到新按钮上，整池查询会复活，需要重新断入口。

## 已回退 / 不要重复加

这些做过又按「最小化」要求撤掉了，除非用户明确要，不要重新引入：

- 首帧主题脚本 `theme.js`（刷新时浅色会闪一帧深色）
- 自定义分组 + 按分组/交集批量暂停选号
- 手机端窄屏自适应（表格列收敛、模块贴边、导航横排）
- `packages` 全量模式（已彻底删除，不是隐藏）
