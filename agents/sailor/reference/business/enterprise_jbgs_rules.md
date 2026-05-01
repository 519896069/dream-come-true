# 企业端揭榜管理业务规则

## 一、业务概述

### 1.1 业务背景

「发榜揭榜」是大中小企业融通创新行动的核心业务流程：

- **发榜**：大企业（龙头企业）发布技术创新需求（场景验证类、采购需求类、技术攻关类）
- **揭榜**：中小企业看到需求后，提交「揭榜」申请，说明自身能力、技术方案、合作方式
- **企业端**：中小企业查看需求、提交揭榜申请、管理申请状态

### 1.2 核心链路

```
场次(Session) → 活动(Activity) → 企业申请(Application)
```

---

## 二、业务功能点

| 功能编号 | 功能名称 | 说明 |
|---------|---------|------|
| F001 | 揭榜列表 | 查看本企业提交的所有揭榜申请，记录需求名称、需求方、申请书、申请时间、审核状态 |
| F002 | 修改揭榜申请书 | 修改已提交的揭榜申请书（标题、企业情况、技术能力、合作方案等）|
| F003 | 删除揭榜申请 | 删除未通过审核的揭榜申请 |
| F004 | 发榜管理 | 查看本企业发布的的需求列表 |
| F005 | 查看需求详情 | 查看已发布的的技术攻关/采购需求/场景验证需求详情 |
| F006 | 申请揭榜 | 中小企业填写揭榜申请书并提交 |
| F007 | 查看申请详情 | 查看揭榜申请书的完整内容（弹窗展示） |

---

## 三、数据流线稿

```
企业端用户操作
    ↓
前端页面
• 揭榜列表: /activity_applies/index2025.vue
• 修改申请: /activity_applies/jbgs/join2024.vue
• 发榜列表: /enterprise/activity/index2024.vue
    ↓
前端 API (docking.js / activity_applies.js)
• get2025() → GET /applies/2025
• create() → POST /activity_applies/application
• update() → PATCH /activity_applies/{id}
• delApplication() → DELETE /apply/{id}
    ↓
后端路由 → ActivityAppliesController
    ↓
Business: ActivityAppliesBusiness
    ↓
Model: apply_form, activities, docking_table
```

---

## 四、数据库设计

### 4.1 ER 图

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│     users       │──1:1──│  docking_table  │       │   activities    │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id (PK)         │       │ id (PK)         │       │ id (PK)         │
│ enterprise_name │       │ user_id (FK)    │       │ user_id (FK)    │
└─────────────────┘       │ enterprise_name │       │ title           │
        │                  │ register_loc... │       │ plate_id=2(JBGS)│
        │                  └─────────────────┘       │ gkcode          │
        │                           │               └────────┬────────┘
        │ 1:N                       │ 1:N                    │ 1:N
        ▼                           ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            apply_form                                        │
│                         (揭榜申请表 / 申请书)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ id (PK) | user_id (FK) | activity_id (FK) | title | desc | capacity | ... │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 表结构

#### activities (发榜需求表)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 发榜企业用户ID |
| title | varchar(128) | 需求名称 |
| plate_id | bigint | 专栏ID (2=揭榜) |
| gkcode | varchar(64) | 公开码 |
| brief | text | 需求简介 |
| content | longtext | 需求详情 |
| type | varchar(32) | 活动类型 |
| way | varchar(255) | 合作方式 |
| time | varchar(255) | 时间要求 |
| start_time | datetime | 需求开始时间 |
| end_time | datetime | 需求结束时间 |
| dead_line | datetime | 截止日期 |
| public_status | tinyint | 发布状态 (0不公开/1公开) |
| examine_status | tinyint | 审核状态 |

#### apply_form (揭榜申请表)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 揭榜方用户ID |
| activity_id | bigint | 关联的需求ID |
| title | varchar(32) | 事项名称 |
| desc | longtext | 中小企业基本情况简介 |
| capacity | longtext | 技术创新能力情况 |
| programme | longtext | 合作方案 |
| way | longtext | 合作方式 |
| gkcode | varchar(55) | 关联需求公开码 |
| annex | json | 附件 |
| status | tinyint | 审核状态 (0未审核/1已提交/3通过/4拒绝) |
| activity_type | tinyint | 需求类型 (0场景验证/1采购需求/2技术攻关) |
| reject_reason | varchar(55) | 拒绝原因 |

**约束**：`UNIQUE(user_id, activity_id)` — 每个企业对每个需求只能提交一份申请

#### docking_table (企业对接信息表)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 用户ID |
| enterprise_name | varchar(128) | 企业名称 |
| enterprise_number | varchar(64) | 信用代码 |
| register_province | varchar(64) | 注册省份 |
| register_city | varchar(64) | 注册市区 |
| delegate_name | varchar(128) | 法人代表 |
| contact_name | varchar(128) | 联系人 |
| contact_phone | varchar(128) | 联系电话 |
| email | varchar(128) | 邮箱 |
| registered_capital | varchar(128) | 注册资本(万元) |
| revenue | varchar(128) | 上年度营收(万元) |
| people_engaged_num | varchar(128) | 从业人数 |
| enterprise_brief | longtext | 企业简介 |
| status | tinyint | 状态 (0未提交/1草稿/2发布) |

---

## 五、核心业务流程

### 5.1 发榜流程 (大企业)

```
发布需求 → 填写需求信息 → 系统自动计算周期 → 公开需求
```

### 5.2 揭榜流程 (中小企业)

```
1. 浏览需求列表 (activities WHERE plate_id=2)
2. 查看需求详情
3. 填写揭榜申请书 (apply_form)
4. 提交申请 (status=1)
5. 等待审核
```

### 5.3 审核流程 (平台/发榜方)

```
状态: 0未审核 → 1已提交 → 3通过 / 4拒绝 / 2退回修改

• 通过: 企业可查看对接结果
• 拒绝: 查看拒绝原因，可修改后重新提交
• 退回: 可修改后再次提交
```

---

## 六、GKCode 公开码生成规则

### 6.1 规则说明

GKCode（公开码）是需求的唯一标识符，用于揭榜方查找和关联需求。

| 条件 | 格式 | 示例 |
|------|------|------|
| 创建日期 ≥ 2026-04-01 且 plate_id = 2 (揭榜) | `Q{quarter}-{sequence:04d}` | `Q2-0001` |
| 否则 | `id + 300` | `301` |

### 6.2 季度截止日期计算规则

需求周期按季度计算，截止日期为**下个季度末月的25日**：

| 当前季度 | start_time | end_time | dead_line |
|---------|-----------|---------|----------|
| Q1 (1-3月) | 当年Q1首月20日 | 当年Q1末月25日 | 当年6月25日 |
| Q2 (4-6月) | 当年Q2首月20日 | 当年Q2末月25日 | 当年9月25日 |
| Q3 (7-9月) | 当年Q3首月20日 | 当年Q3末月25日 | 当年12月25日 |
| Q4 (10-12月) | 当年Q4首月20日 | 当年Q4末月25日 | **次年3月25日** |

---

## 七、API 文档

### 7.1 获取揭榜列表（2025）

- **URL**: `/api/applies/2025`
- **Method**: `GET`
- **Auth**: Bearer Token (企业用户)
- **Request Params**: `page`, `commit=true`, `plate_id=2`, `enterprise_name` (optional)

### 7.2 创建揭榜申请

- **URL**: `/api/activity_applies/application`
- **Method**: `POST`
- **Request**: `{ title, activity_id, gkcode, desc, capacity, programme, way, annex }`
- **Business Logic**: 每个企业对每个 activity_id 只能创建一条记录（存在则更新）

### 7.3 更新揭榜申请

- **URL**: `/api/activity_applies/{id}`
- **Method**: `PATCH`

### 7.4 删除揭榜申请

- **URL**: `/api/apply/{id}`
- **Method**: `DELETE`

---

## 八、相关文件索引

| 文件 | 说明 |
|------|------|
| `frontend/gxadmin/src/views/enterprise/activity_applies/index2025.vue` | 揭榜列表页 |
| `frontend/gxadmin/src/views/enterprise/activity_applies/jbgs/join2024.vue` | 修改揭榜申请书页 |
| `backend/gongxinphp/app/Http/Controllers/Api/ActivityAppliesController.php` | 活动申请控制器 |
| `backend/gongxinphp/app/Business/ActivityAppliesBusiness.php` | 活动申请业务逻辑 |
| `backend/gongxinphp/app/Services/QuarterSequenceService.php` | 季度序号服务 |
