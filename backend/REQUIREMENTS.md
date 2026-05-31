# 打了吗 - 后端需求文档

## 1. 项目概述

为微信小程序**"打了吗"**提供后端 API 服务。

> "今天打了吗？" —— 一句简单的问候，背后是男人之间心照不宣的默契。

这是一款面向群友的**整活工具**，用来记录每日"手冲打卡"，统计群内月度数据，评选出当之无愧的**飞机王**👑。项目定位纯粹是朋友间的恶搞娱乐，怎么骚怎么来。

**核心原则**：
- 无登录、无鉴权，填个花名就能开冲
- 数据是玩笑，但统计是认真的
- 月底飞机王，必须卷起来

## 2. 技术栈

- **运行时**：Node.js
- **框架**：Koa2
- **数据库**：SQLite（轻量，数据量小，无需额外部署）
- **ORM**：Prisma / Sequelize
- **认证**：无。前端本地生成 UUID 作为用户标识，接口直接透传 user_id
- **部署**：云服务器 / 微信云托管

## 3. 核心设计：群组

> **群组是整个产品的核心容器**。打卡、排行、飞机王，全部发生在群内。用户必须先加入或创建一个群，才能使用任何功能。

### 3.1 群组模型

```
                         ┌──────────────────────┐
                         │       群组 Group       │
                         │  id, name, invite_code │
                         │  owner (群主)          │
                         └────┬──────────────┬────┘
                              │              │
                    M:N       │    1:N       │ 1:N
                   ┌──────────┘              └──────────┐
                   ▼                                    ▼
          ┌─────────────────┐                  打卡 Checkin
          │   用户-群 中间表  │                  (绑定 group_id + user_id)
          │  user_groups     │
          └───────┬─────────┘
                  │ M:N
                  ▼
             用户 User
             (同一个 UUID 可加入多个群)
```

- 一个用户可以加入多个群，A 群的打卡不影响 B 群
- 打卡记录绑定 user_id + group_id
- 排行/飞机王按 group_id + year + month 独立计算
- 前端维护一个"当前活跃群"，切换群时首页数据全部刷新

### 3.2 用户旅程

```
新兵入伍 → 起个花名 → 建立或加入一个"战队"
                              ↓
                    进入主战场（冲！）
                              
老兵回归 → 自动归队 → 挑一个战队 → 继续冲

转战其他群 → 切换战队 → 各群数据独立，互不干扰
```

## 4. 功能需求

### 4.1 群组模块（P0 最高优先级）

| 编号 | 功能 | 描述 |
|------|------|------|
| G-01 | 创建群组 | 用户创建群组（填群名），自动成为群主，生成 6 位数字邀请码 |
| G-02 | 加入群组 | 输入邀请码加入已有群，校验邀请码有效性 |
| G-03 | 群信息 | 查询群名称、成员列表、邀请码、创建时间 |
| G-04 | 退出群组 | 用户可主动退群，群主退群需先转让或解散 |
| G-05 | 踢出成员 | 群主可移除成员（P2） |

### 4.2 用户模块

> 无登录，前端本地生成 UUID + 填昵称。同一 UUID 可加入多个群。

| 编号 | 功能 | 描述 | 优先级 |
|------|------|------|--------|
| U-01 | 注册/获取用户 | 前端传入 user_id + nickname，不存在则创建 | P0 |
| U-02 | 我的群列表 | 返回用户加入的所有群 | P0 |
| U-03 | 修改昵称 | 用户可随时修改昵称 | P1 |

### 4.3 打卡模块（核心玩法）

| 编号 | 功能 | 描述 | 优先级 |
|------|------|------|--------|
| C-01 | 冲一发 | 用户在所在群打卡，记录精确到秒的时间戳，一天可多次（懂的都懂） | P0 |
| C-02 | 节制模式 | 群主可设每日上限（"悠着点"），默认不限，给肝帝留足发挥空间 | P1 |
| C-03 | 打卡备注 | 可选填备注，比如"今天手感不错"、"新买的纸巾到了"之类的骚话 | P2 |
| C-04 | 撤回 | 5 分钟内可反悔，谁还没有个手滑的时候 | P2 |
| C-05 | 战斗日历 | 指定月份的打卡日期，一目了然你的勤奋程度 | P1 |

### 4.4 统计模块（卷王排行）

| 编号 | 功能 | 描述 | 优先级 |
|------|------|------|--------|
| S-01 | 实时战况 | 本月群内成员打卡次数排名，谁是卷王谁是摸鱼怪一目了然 | P0 |
| S-02 | 月度飞机王 | 每月最后一天 23:59:59 结算，冠以"本月飞机王"称号 + 👑 | P0 |
| S-03 | 名人堂 | 历月飞机王榜单，让传奇永流传 | P1 |
| S-04 | 个人战绩 | 总次数、本月次数、最高纪录、最长连续打卡天数（肝帝认证） | P1 |
| S-05 | 群数据面板 | 群总次数、本月活跃人数、人均次数（看看谁是拖后腿的） | P2 |

## 5. 数据模型设计

### 5.1 用户表 (users)

```
id          VARCHAR   PK     前端生成的 UUID
nickname    VARCHAR          昵称
created_at  DATETIME
updated_at  DATETIME
```

### 5.2 群组表 (groups)

```
id          INTEGER   PK 自增
name        VARCHAR          群名称
invite_code VARCHAR   UNIQUE 邀请码（6 位数字）
owner_id    VARCHAR   FK → users.id
max_daily   INTEGER          每日打卡上限（0=不限），默认 0
created_at  DATETIME
updated_at  DATETIME
```

### 5.3 用户-群组中间表 (user_groups)

```
user_id     VARCHAR   FK → users.id   联合主键
group_id    INTEGER   FK → groups.id  联合主键
joined_at   DATETIME
```

唯一约束：`(user_id, group_id)`

### 5.4 打卡记录表 (checkins)

```
id          INTEGER   PK 自增
user_id     VARCHAR   FK → users.id
group_id    INTEGER   FK → groups.id
note        VARCHAR          备注（可选）
checkin_at  DATETIME         打卡时间（精确到秒）
created_at  DATETIME
```

索引：`(user_id, group_id, checkin_at)`, `(group_id, checkin_at)`

### 5.5 月度飞机王表 (monthly_kings)

```
id          INTEGER   PK 自增
group_id    INTEGER   FK → groups.id
user_id     VARCHAR   FK → users.id
year        INTEGER          年份
month       INTEGER          月份（1-12）
count       INTEGER          当月打卡次数
awarded_at  DATETIME         结算时间
```

唯一约束：`(group_id, year, month)`

## 6. API 接口设计

> 所有接口无鉴权，直接传 user_id。

### 6.1 群组（核心）

```
POST   /api/group              创建群组 { user_id, name } → { group, invite_code }
POST   /api/group/join         加入群组 { user_id, invite_code }
GET    /api/group/:id          群信息（群名、成员列表、邀请码等）
DELETE /api/group/:id/member   移除成员 { operator_id, target_user_id }（群主操作）
POST   /api/group/:id/leave    退出群组 { user_id }
```

### 6.2 用户

```
POST   /api/user               注册/获取用户 { user_id, nickname }
PUT    /api/user/nickname      修改昵称 { user_id, nickname }
GET    /api/user/:id/groups    获取用户加入的所有群列表
```

### 6.3 打卡

```
POST   /api/checkin            打卡 { user_id, group_id, note? }
DELETE /api/checkin/:id        撤销打卡 { user_id }
GET    /api/checkin/calendar   打卡日历 ?user_id=&group_id=&year=&month=
GET    /api/checkin/today      今日打卡状态 ?user_id=&group_id=
```

### 6.4 排行

```
GET    /api/rank/monthly       本月排行 ?group_id=&year=&month=
GET    /api/rank/kings         历史飞机王 ?group_id=
GET    /api/rank/personal      个人统计 ?user_id=&group_id=
GET    /api/rank/group-stats   群统计概览 ?group_id=
```

### 6.5 通用返回格式

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

错误码：
- `0`：成功
- `1001`：参数错误
- `1004`：资源不存在（用户、群、邀请码无效等）
- `2001`：今日已达上限
- `2002`：已过撤销时限
- `2003`：无权限（非群主操作、非本人撤销等）

## 7. 非功能需求

### 7.1 安全性
- 打卡撤销校验 user_id 是否为打卡人本人
- 群管理操作校验 operator_id 是否为群主
- 邀请码 6 位纯数字，随机生成不可预测
- 无敏感用户数据，不涉及隐私合规风险

### 7.2 性能
- 打卡接口响应 < 200ms
- 排行查询可加内存缓存（5 分钟 TTL）
- 月度结算使用 node-cron 定时任务

### 7.3 可靠性
- SQLite 数据库文件定期备份
- 打卡接口幂等：同 user_id 同一秒内重复请求只记一条
- 月度结算支持手动触发（补算）

## 8. 项目结构

```
backend/
├── src/
│   ├── app.js              # 应用入口
│   ├── config/             # 配置（数据库路径、端口等）
│   ├── models/             # 数据模型
│   ├── controllers/        # 控制器
│   ├── services/           # 业务逻辑层
│   ├── routes/             # 路由
│   ├── utils/              # 工具函数
│   └── jobs/               # 定时任务（月度结算）
├── data/                   # SQLite 数据库文件目录
├── package.json
└── .env.example
```

## 9. 里程碑

| 阶段 | 内容 |
|------|------|
| M1 | 项目初始化、建表（users, groups, user_groups, checkins, monthly_kings）、群组创建/加入/查询 |
| M2 | 用户注册/获取、打卡核心流程（打卡、撤销、日历） |
| M3 | 排行统计（实时排行、月度飞机王结算） |
| M4 | 群组管理增强（退出、踢人、设置、用户群列表） |
| M5 | 联调测试、上线 |
