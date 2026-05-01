# 管理员端发榜揭榜管理业务规则

## 一、业务概述

### 1.1 业务背景

「发榜揭榜」是大中小企业融通创新行动的核心业务流程：

- **发榜**：大企业（龙头企业）发布技术创新需求（场景验证类、采购需求类、技术攻关类）
- **揭榜**：中小企业看到需求后，提交「揭榜」申请，说明自身能力、技术方案、合作方式
- **管理员**：负责审核大企业发布的需求、审核中小企业的揭榜申请

### 1.2 管理员端菜单结构

| 菜单路径 | 菜单名称 | 说明 |
|---------|---------|------|
| `/jbgs/index` | 大企业发榜列表 | 查看/审核所有大企业发布的需求 |
| `/jbgs/applies` | 揭榜列表 | 查看某条需求的揭榜申请列表 |
| `/jbgs/edit` | 添加需求 | 管理员代大企业发布需求 |
| `/jbgs/list`（隐藏） | 揭榜总览 | 全局查看所有揭榜对接表 |

---

## 二、业务功能点

### F001 | 大企业发榜列表

**路径**：`/jbgs/index` → `views/admin/activity/jbgs/index2024.vue`

管理员查看所有大企业发布的技术创新需求，包含审核操作。

**功能清单**：
- 展示需求列表（分页）
- 查看大企业技术创新需求表（弹窗）
- 审核需求：通过 / 拒绝 / 返回修改
- 导出当季度大企业需求（按日期范围导出）
- 查看需求具体内容（跳转前台页面）

**审核状态流转**（`examine_status` 字段）：
```
0-未审核 → 1-通过 → 3-返回修改
                ↘ 2-拒绝
```

### F002 | 揭榜列表（按需求查看）

**路径**：`/jbgs/applies` → `views/admin/activity/jbgs/applies.vue`

管理员查看某一条发榜需求收到的所有揭榜申请，并进行审核。

**审核状态流转**（`apply_form.status` 字段）：
```
1-未审核 → 2-通过（老版本）
         ↘ 3-拒绝（老版本）

1-未审核 → 3-通过（applies2024.vue）
         ↘ 4-拒绝（applies2024.vue）
         ↘ 1-返回修改（applies2024.vue）
```

### F003 | 揭榜总览

**路径**：`/jbgs/list`（隐藏） → `views/admin/demand/list.vue`

查看所有揭榜企业的对接表总览，按企业维度展示。

### F004 | 添加/编辑需求

**路径**：`/jbgs/edit` 或 `/jbgs/edit/:id`

管理员代大企业创建或编辑一条技术创新需求。

**字段**：
- 需求标题（`title`）
- 需求类型（`type`）：0-场景验证类 / 1-采购需求类 / 2-技术攻关类
- 需求方 / 作者（`author`）
- 需求时间要求（`time`）
- 需求合作方式（`way`）
- 需求所属行业（`industry`）
- 需求活动内容（`content`，富文本）
- 需求缩略图（`banner`）

**业务规则**：
- `plate_id` 固定为 2（JBGS 专栏）
- 创建时自动设置 `examine_status=1`（通过）、`public_status=1`（公开）
- 自动生成 `gkcode`（公开编号，按季度序列规则）
- 自动计算截止日期（`dead_line`）：下个季度末月25号

---

## 三、数据流线稿

```
【发榜流程 - 大企业需求发布】
管理员操作（/jbgs/edit）
    ↓ POST /api/activity/admin-create
    ↓ ActivityBusiness::adminCreateActivity()
    ↓ activities 表（plate_id=2, examine_status=1, public_status=1）
    ↓ 自动生成 gkcode、dead_line

【揭榜流程 - 中小企业申请】
企业端操作（前台页面）
    ↓ POST /api/docking
    ↓ DockingBusiness::create()
    ↓ docking_table 表 + apply_form 表

【管理员审核揭榜】
管理员操作（/jbgs/applies）
    ↓ PUT /api/docking/{id}/update-status
    ↓ DockingBusiness::updateStatus()
    ↓ apply_form 表 status 字段更新
```

---

## 四、数据库设计

### 4.1 ER 图

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────┐
│  activities     │       │  apply_form       │       │  docking_table  │
│  (发榜需求表)   │──1:N──│  (揭榜申请表)     │───────│  (企业对接表)   │
└─────────────────┘       └──────────────────┘       └─────────────────┘
                                   │
                                   │ N:1（通过 user_id）
                                   ↓
                            ┌─────────────────┐       ┌─────────────────┐
                            │  users          │       │  enterprise_info│
                            │  (用户表)        │───────│  (企业详情表)   │
                            └─────────────────┘       └─────────────────┘
```

### 4.2 表结构

#### activities（活动/需求表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 发布人ID |
| author | varchar | 作者/需求方名称 |
| title | varchar | 需求标题 |
| content | longtext | 需求内容（富文本） |
| plate_id | bigint | 专栏ID，2=JBGS发榜揭榜 |
| type | varchar | 需求类型：0/1/2 |
| examine_status | tinyint | 审核状态：0未审核/1通过/2拒绝/3返回修改 |
| public_status | tinyint | 发布状态：0不公开/1公开 |
| gkcode | varchar | 公开编号 |
| industry | varchar | 所属行业 |
| way | varchar | 合作方式 |
| time | varchar | 时间要求 |
| banner | varchar | 缩略图 |
| start_time | datetime | 开始时间 |
| end_time | datetime | 结束时间 |
| dead_line | datetime | 截止时间 |
| area_id | bigint | 区域ID |
| reject_reason | varchar | 拒绝理由 |
| annex | json | 附件 |

#### apply_form（揭榜申请表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 申请人用户ID |
| activity_id | bigint | 关联的需求/活动ID |
| title | varchar | 需求标题 |
| desc | longtext | 企业基本情况简介 |
| capacity | longtext | 创新能力情况 |
| programme | longtext | 技术攻关方案 |
| way | longtext | 合作方式 |
| gkcode | varchar | 活动代码（gk+序号） |
| status | tinyint | 审核状态：0未审核/1已提交未审核/3通过/4拒绝 |
| activity_type | tinyint | 需求类型：0/1/2 |
| reject_reason | varchar | 拒绝原因 |
| annex | json | 附件 |

#### docking_table（企业对接表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 用户ID |
| enterprise_name | varchar | 企业名称 |
| enterprise_number | varchar | 统一社会信用代码 |
| register_province | varchar | 注册省份 |
| register_city | varchar | 注册市区 |
| delegate_name | varchar | 法人代表 |
| contact_name | varchar | 联系人 |
| contact_phone | varchar | 联系人电话 |
| email | varchar | 邮箱 |
| registered_capital | varchar | 注册资本（万元） |
| revenue | varchar | 上年度营业收入（万元） |
| people_engaged_num | varchar | 从业人数 |
| enterprise_brief | longtext | 企业概况 |
| status | tinyint | 提交状态：0未提交/1草稿/2发布 |

---

## 五、API 文档

### 5.1 获取大企业发榜列表（2024）

- **URL**: `GET /api/activity2024`
- **Params**: `plate_id=2`, `page`

### 5.2 审核需求

- **URL**: `PATCH /api/activity/examine/{id}`
- **Body**: `{ "examine_status": 1, "reject_reason": "" }`

### 5.3 获取揭榜申请列表

- **URL**: `GET /api/docking`
- **Params**: `activity_id`, `plate_id=2`, `commit=true`, `enterprise_name`, `page`

### 5.4 更新揭榜申请状态

- **URL**: `PUT /api/docking/{id}/update-status`
- **Body**: `{ "status": 3 }`

### 5.5 管理员创建需求

- **URL**: `POST /api/activity/admin-create`
- **Body**: `{ "title", "author", "type", "banner", "content", "industry", "way", "time", "plate_id": 2 }`
- **说明**: 自动设置 examine_status=1, public_status=1, 生成 gkcode

### 5.6 获取揭榜总览（2024）

- **URL**: `GET /api/docking/2024`
- **Params**: `commit=true`, `plate_id=2`

### 5.7 导出揭榜申请

- **URL**: `GET /api/docking/exportapply`
- **Params**: `status=apply`

### 5.8 导出需求（按日期）

- **URL**: `GET /api/activity/export`
- **Params**: `start_date`, `end_date`, `token`
