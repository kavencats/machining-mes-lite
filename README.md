# 机械加工生产报工系统（MES Lite）

面向中小机械加工车间的生产报工系统：建工单、按工序报工、首件三检、不良处置、统计计件工资。
单进程部署，SQLite 单文件数据库，浏览器直接用，不需要装数据库。

> 当前版本 **2.6.25**，更新记录见 [CHANGELOG.md](CHANGELOG.md)。

---

## 一、快速启动

### 环境要求

- Python 3.9+（推荐 3.10）
- 依赖：`pip install -r requirements.txt`

### 三种启动方式

```bash
python run.py                # 默认监听局域网 0.0.0.0:8080（车间其他电脑可直接访问）
python run.py --port 9000    # 换端口
python run.py --local        # 只监听 127.0.0.1（仅本机可用）
python run.py --proxy        # 只监听 127.0.0.1，由 nginx / Caddy 转发
```

启动后终端会打印可用地址，浏览器打开即可。

### 首次运行

系统初始化前除创建账号外全部拒绝（后端强制，不只是前端弹窗）。
打开页面按提示创建第一位管理员——**第一位自动成为超级管理员**，权限全开且不可被他人修改。

### 打包成 exe（Windows 免装 Python）

```bash
python build_exe.py
```

产物在 `dist/`，数据与上传目录在 exe 同级，不进临时目录（打包守卫会检查）。

### 部署脚本

`deploy/` 下提供了 `start.bat`、`start_lan.bat`、`start_lan.sh`、`backup.bat`、`backup.sh`、
`nginx.conf`、`Caddyfile`、`mes.service`，按场景选用。

---

## 二、业务流程

```
建工单 → 下达 → 现场报工 → 检验判定 → 合格数回写 → 工序完工 → 流转下一序 → 工单完工
```

**建单自动展开工序**：选零件后按工艺路线自动生成工序，不用手工逐道加。

**报工类型**（6 种）：

| 类型 | 说明 |
|---|---|
| 首件报检 | 批量加工前的准入。自检 + 互检，检验员专检合格后才可进度报工 |
| 开工 | 标记工序开始，不计产量 |
| 进度 | 报"交出多少件去检验"，随报随走，**不锁定工序** |
| 完工 | 本序最后一批，报完等完工检验 |
| 返修复检 | 返修件修好后再交检 |
| 暂停 | 记录暂停原因，暂停期间不可报工 |

**报检制**：现场只报交出数量，合格 / 报废 / 返修由检验员判定后回写。
不允许自报合格——自己报自己判等于没有检验。

**首件三检**：工序配了检验项目时填逐项实测值（自检 / 互检 / 专检各一遍）；
没配检验项目时填文字记录。**同一道工序只认其中一种形式**，按有无检验项目自动切换。

**互检人范围**：本班组在职员工，外加全部班组长与检验员（不限班组）。
班组只剩一人时仍可选班组长 / 检验员兜底，不至于卡住报检。

**报工锁定规则**（出厂默认）：

| 上一次判定 | 工序状态 | 能否继续报工 |
|---|---|---|
| 全合格 | 进行中 | ✅ 接着干 |
| 有返修 | 待返修 | ❌ 修好交回复检合格才解锁 |
| 有报废 | 报废待处置 | ❌ 检验员填不良处置才解锁 |

首件未放行、暂停、取消的工序一律不可报工。
**首件判废豁免报废锁**——首件未放行本身已挡住全部批量报工，再加一把会互相锁死。

---

## 三、功能一览

| 页面 | 说明 |
|---|---|
| 报工台 | 按工种自动派发的待办工序，可直接报工 |
| 我的报工 | 报工流水与当日汇总，班组长可撤销误报 |
| 生产工单 | 工单按工艺路线展开工序；列表内可直接取消工单 |
| 基础数据 | 设备 / 零件 / 工艺路线 / 员工 / 班组 / 班次 / 不良原因，支持批量导入 |
| 检验工作台 | 待检 / 已判定 / 不良处置，判定后回写产量并解锁报工 |
| 统计看板 | 产量、良率、效率、设备利用率、计件工资 |
| 停机登记 | 故障、换型、待料等停机时长，计入设备利用率 |
| 权限配置 | 角色权限矩阵、数据范围、个人权限覆盖（仅管理员） |
| 审计日志 | 登录、报工、检验、工单、基础数据、权限、数据库全部操作留痕 |
| 数据库管理 | 在线备份 / 恢复 / 重置 / 完整性检查 / 优化，备份可下载 |

### 几个跨页面的机制

- **列表两种查看方式**：卡片 / 表格切换，按表格分别记忆；窄屏默认卡片。
  表格视图高度按剩余空间算，底部留空与左右留空一致。
- **宽高自适应**：标准模式有高度分档，超宽屏内容居中；大屏模式另有独立的字号与分档规则。
- **重置数据库**：强制先自动备份，备份失败则拒绝执行；可先预览将删除什么。
- **批量导入**：选完文件先解析预览，标出未识别列与缺失必填列，缺必填列时禁用导入按钮；
  失败行可导出改完重传。

---

## 四、系统结构

### 五个模块

```
公共 common     配置 / 数据库 / 模型 / 批量导入 / 图纸
用户管理 user   认证 / 员工档案 / 班组 / 权限矩阵
生产管理 prod   工单 / 报工 / 基础数据(设备零件工艺班次) / 统计
质量管理 qual   检验判定 / 不良处置 / 不良原因 / 检验项目
系统管理 sys    初始化 / 审计 / 备份 / 重置 / 维护
```

分层：**routers（接口层）→ services（业务层）→ common（模型与基础）**。
底层不依赖上层业务模块，启动自检会检查（分层无反向依赖）。

### 目录结构

```
app/
  common/    配置、数据库、数据表模型、通用服务（批量导入、图纸）
  user/      认证、员工与班组、权限判定
  production/ 工单、报工、统计、生产基础数据
  quality/   检验、不良原因、检验项目
  system/    初始化、审计、备份与重置
  static/    前端单页（index.html）
scripts/     smoke_test（后端）、ui_test（前端断言）、import_guard（启动自检）、check_bat
deploy/      启动脚本、备份脚本、nginx / Caddy 配置、systemd 单元
run.py       启动入口
build_exe.py 打包脚本
```

### 数据表（18 张）

```
employee machine part routing routing_check_item defect_reason
shift work_order wo_operation production_report inspection downtime
role_permission employee_permission role_scope employee_scope
audit_log perm_config_audit
```

---

## 五、权限体系

### 角色（6 个）

| 角色 | 定位 |
|---|---|
| 管理员 admin | 厂长 / 系统管理员，全部权限与全部数据 |
| 主管 supervisor | 统管全厂生产，可撤销报工、维护基础数据，不管权限与员工档案 |
| 计划员 planner | 建单下达、排产与进度跟踪，不改基础数据台账 |
| 班组长 leader | 建单下达、撤销报工、维护基础数据，数据限本班组 |
| 操作工 worker | 报工、报修、查看本人数据 |
| 检验员 inspector | 判定报检数量并回填合格 / 报废 / 返修 |

### 权限码（39 项）

**报工**

| 权限码 | 名称 | 默认角色 |
|---|---|---|
| `report.create` | 提交报工（含批量报工） | admin / leader / worker |
| `report.view` | 查看报工明细 | + supervisor / planner |
| `report.export` | 导出报工明细 | 同上 |
| `report.cancel` | 撤销报工并回滚累计 | admin / leader / supervisor |
| `report.resume` | 恢复暂停的工序 | admin / supervisor / leader / worker |

**生产工单**

| 权限码 | 名称 | 默认角色 |
|---|---|---|
| `order.view` | 查看工单 | admin / leader / worker / supervisor / planner |
| `order.create` | 创建工单 | admin / leader / supervisor / planner |
| `order.release` | 下达工单 | 同上 |
| `order.status` | 变更状态（暂停 / 恢复 / 完工 / 关闭） | 同上 |
| `order.cancel` | **取消工单**（整单作废，不可恢复，须填原因） | admin / leader / supervisor |
| `order.report` | 工单详情内直接报工 | admin / leader / worker |

> 取消单列：取消会连带作废在制工序与待检验单，与"暂停"不是一回事。

**基础数据**

| 权限码 | 名称 | 默认角色 |
|---|---|---|
| `master.view` | 查看基础数据 | 全部业务角色 |
| `master.machine.edit` | 维护设备台账 | admin / leader / supervisor |
| `machine.status.set` | 设置设备状态 | + planner |
| `machine.status.fault` | 报修（仅故障） | + worker |
| `master.part.edit` | 维护零件 / 图纸 | admin / leader / supervisor |
| `master.routing.edit` | 维护工艺路线 | admin / leader / supervisor |
| `master.defect.edit` | 维护不良原因码 | admin / leader / supervisor |
| `master.shift.edit` | 维护班次 | admin / leader / supervisor |
| `master.team.edit` | 维护班组 | admin |
| `master.employee.edit` | 维护员工档案 | admin |
| `master.import` | 批量导入 | admin / leader / supervisor |

**检验 / 停机 / 统计 / 审计 / 数据库**

| 权限码 | 名称 | 默认角色 |
|---|---|---|
| `inspect.view` | 查看检验单 | admin / leader / inspector / worker / supervisor / planner |
| `inspect.submit` | 提交检验结果 | admin / leader / inspector / supervisor |
| `downtime.create` | 登记停机 | 全部业务角色 |
| `downtime.view` | 查看停机记录 | 全部业务角色 |
| `stats.view` | 查看统计 | 全部业务角色 |
| `stats.employee` | 人员效率 | admin / leader / worker / supervisor / planner |
| `stats.machine` | 设备利用率 | 同上 |
| `stats.wage` | 计件工资 | admin / leader / worker / supervisor |
| `audit.view` | 查看审计日志 | admin / supervisor |
| `audit.export` | 导出审计日志 | admin / supervisor |
| `audit.purge` | 清理过期日志 | admin |
| `db.view` | 查看数据库状态 | admin / supervisor |
| `db.backup` | 备份数据库 | admin / supervisor |
| `db.maintain` | 数据库维护 | admin |
| `db.restore` | 恢复数据库 | admin |
| `perm.config` | 权限配置 | admin |
| `perm.view` | 查看权限说明 | 全部角色 |

### 数据范围

三类：`all` 全部数据 / `team` 本班组 / `self` 仅本人。
四类资源分别配置：报工 `report`、统计 `stats`、工资 `wage`、员工 `employee`。

| 角色 | 报工 | 统计 | 工资 | 员工 |
|---|---|---|---|---|
| 管理员 / 主管 | 全部 | 全部 | 全部 | 全部 |
| 计划员 | 全部 | 全部 | 仅本人 | 全部 |
| 班组长 | 本班组 | 本班组 | 本班组 | 本班组 |
| 操作工 | 仅本人 | 仅本人 | 仅本人 | 仅本人 |
| 检验员 | 全部 | 全部 | 仅本人 | 全部 |

### 三条可配置线（管理端【权限配置】页）

1. **角色权限矩阵**——勾选即生效，可整体重置回默认
2. **角色数据范围**——上表四列
3. **个人权限覆盖**——对单个员工单独加权限或单独拒绝（deny 优先于角色）

按权限**隐藏**入口，不是置灰：没有权限的人看不到菜单与按钮，
后端同时校验——绕过界面直接调接口同样被拒。

---

## 六、接口清单（117 个）

### 公共（6）

```
GET  /api/master/import/{entity}/fields
GET  /api/master/import/{entity}/template
POST /api/master/import/{entity}
POST /api/master/import/{entity}/parse
POST /api/master/import/{entity}/parse-text
POST /api/master/import/{entity}/text
```

### 用户管理（32）

```
认证   POST /api/auth/login | /logout | /change-password | /refresh
       GET  /api/auth/me | /permissions | /permissions/matrix
员工   GET  POST /api/master/employees
       GET  PUT DELETE /api/master/employees/{emp_id}
       GET  /api/master/employees/mutual          互检人候选
       POST /api/master/employees/{emp_id}/reset-pwd
       GET  POST /api/master/employees/{emp_id}/pin
班组   GET  /api/master/teams
       POST /api/master/teams/rename | /normalize
       DELETE /api/master/teams/{name}
权限   GET  /api/perm/overview | /check | /audit
       GET  PUT /api/perm/roles/{role}
       GET  PUT /api/perm/scopes/{role}
       GET  PUT /api/perm/employees/{emp_id}/permissions
       GET  PUT /api/perm/employees/{emp_id}/scopes
       DELETE /api/perm/employees/{emp_id}/overrides
       POST /api/perm/roles/reset
```

### 生产管理（49）

```
工单   GET  POST /api/orders
       GET  /api/orders/{order_id}
       POST /api/orders/{order_id}/release | /status | /resume
       GET  /api/orders/ops/{woo_id}
       POST /api/orders/ops/{op_id}/pause | /resume
       GET  /api/orders/tasks/my | /next-no
报工   GET  POST /api/reports
       POST /api/reports/batch
       POST /api/reports/{report_id}/cancel
       GET  /api/reports/export | /summary/today
停机   GET  POST /api/downtime
统计   GET  /api/stats/overview | /daily | /employee | /machine
       GET  /api/stats/orders | /defect | /wage | /permissions
设备   GET  POST /api/master/machines
       GET  PUT DELETE /api/master/machines/{m_id}
       POST /api/master/machines/{m_id}/status
零件   GET  POST /api/master/parts
       GET  PUT DELETE /api/master/parts/{p_id}
       GET  POST DELETE /api/master/parts/{p_id}/drawing
       GET  /api/master/parts/code-issues
       POST /api/master/parts/normalize-codes
工艺   GET  POST /api/master/routings
       GET  PUT DELETE /api/master/routings/{r_id}
班次   GET  POST /api/master/shifts
       GET  PUT DELETE /api/master/shifts/{s_id}
```

### 质量管理（14）

```
检验   GET  /api/inspection/pending | /done | /summary
       GET  /api/inspection/{insp_id} | /by-report/{report_id}
       POST /api/inspection/submit | /dispose
不良   GET  POST /api/master/defects
       GET  PUT DELETE /api/master/defects/{d_id}
检验项 GET  POST PUT /api/master/routings/{r_id}/check-items   PUT 为整表提交（提交时不含即删除）
```

### 系统管理（20）

```
初始化 GET  /api/setup/status    POST /api/setup
审计   GET  /api/audit | /meta | /stats | /export
       POST /api/audit/purge
数据库 GET  /api/db/stats | /backups | /reset-preview
       GET  /api/db/backups/{name}/download
       POST /api/db/backup | /restore | /reset | /check | /vacuum
       POST /api/db/cleanup-stray
       DELETE /api/db/backups/{name}
系统   GET  /api/system/data-stats
       POST /api/system/reset-data
健康检查 GET /api/health | /api/meta
```

> 注：`/api/auth/me`、`/api/reports/batch`、`/api/perm/check` 前端未直接调用，
> 但属基础能力（令牌校验 / 批量报工 / 权限自检），予以保留。
> `/api/auth/logout` 自 2.6.12 起前端点退出时调用（写注销审计留痕）。
> 扫码登录（`/api/auth/qrcode*`）与重复接口已在 2.6.11 下线。

---

## 七、配置

### 环境变量

| 变量 | 说明 |
|---|---|
| `MES_DB` | 数据库路径，默认 SQLite 单文件 |
| `MES_UPLOAD_DIR` | 图纸等上传目录 |
| `MES_BACKUP_DIR` | 备份目录（建议指向独立磁盘，别和数据库同盘） |
| `MES_SECRET` | 令牌签名密钥，**生产环境务必设置** |
| `MES_HOST` / `MES_PORT` | 监听地址与端口 |
| `MES_INSPECT_TEAM_KEYWORDS` | 检验班组名关键字（逗号分隔，默认 `检验,质检,品管,品质,质量,QC,IQC`）。班组长/主管须属检验班组才能提交检验判定 |
| `MES_TOKEN_TTL_MINUTES` | 登录令牌有效期（分钟，默认 **5**；配套滑动续期，停手 5 分钟才真过期） |
| `MES_TOKEN_TTL` | 老变量（小时），优先级更高，仍兼容 |
| `MES_DEDUP_WINDOW` | 重复报工去重窗口（默认 25 秒） |
| `MES_MAX_DRAWING_MB` | 图纸大小上限 |
| `MES_ADMIN_PWD` | 出厂默认密码（默认 `l1234567`）。批量导入员工、新建账号未指定密码时都用它；**用它登录会强制要求改成自己的密码**，且不允许把新密码改回它 |
| `MES_SEED_DEMO` | 是否播种演示数据 |
| `MES_QUICK_LOGIN` | 快捷登录（仅演示环境用） |

### 启动参数

`--port` / `-p` 换端口，`--local` 仅本机，`--proxy` 走前置代理，`--lan` 兼容旧脚本（现为默认）。

---

## 八、测试

| 脚本 | 覆盖 | 当前 |
|---|---|---|
| `scripts/smoke_test.py` | 后端全链路 | **1123 项** |
| `scripts/ui_test.js` | 前端静态断言 | **969 项** |
| `scripts/page_viewport_test.js` | 页面分辨率实测（十档，含 4K / 2K） | **408 项** |
| `scripts/import_guard.py` | 启动自检（分层依赖、模块结构、索引、权限缓存、文档一致性） | **18 项** |
| `scripts/check_bat.py` | Windows 批处理字节规范 | 29 项 |

```bash
python scripts/smoke_test.py
node scripts/ui_test.js
python scripts/import_guard.py

# 分辨率实测需要先起服务
node scripts/page_viewport_test.js http://127.0.0.1:9911 工号 密码
```

分辨率实测覆盖 390 / 820 / 1366 / 1440 / 1920 / **2560×1440（2K）** /
**3840×2160（4K）** / **3840×1080（4K 拼接屏）** / 3440 / 竖屏，逐档量
横向溢出、元素越界、整页溢出与双重滚动。4K 与 4K 拼接必须分开测：同为
3840 宽，2160 高与 1080 高的排版完全不同。

启动自检在每次服务启动时自动跑一遍，失败会在日志中列出具体项。

---

## 九、编号规则

- **工单号**：`TBW` + 年月日(8 位) + 当日流水号(3 位)
- **工序号**：两位数，不足补零（10、20、30…，便于中间插序）
- **检验单号**：按日期与流水生成
- **零件图号**：支持批量整理（`/api/master/parts/normalize-codes`）

---

## 十、已知取舍

- **物料不单独建表**：采购时叫物料、加工时叫零件，建两张表就要维护两套编码，必然对不上。
  辅料（刀具、切削液）如需管理，需另行扩展。
- **大屏常亮依赖 Wake Lock API**：需要 HTTPS 或 localhost，内网 http 下会静默降级，
  表现是屏幕可能自己息屏。
- **重置数据库会清空全部数据**：虽有自动备份兜底，但备份默认与数据库同盘，
  建议用 `MES_BACKUP_DIR` 指向独立磁盘或 NAS。
