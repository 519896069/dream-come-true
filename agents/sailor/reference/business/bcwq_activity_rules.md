# 百场万企活动业务规则

## 一、业务概述

**业务定义**：百场万企活动是政府或机构组织的系列对接活动，旨在促进产业链上下游企业合作。每场活动归属于一个"场次(Session)"，场次决定了活动的类型和显示的字段。

**核心链路**：
```
场次(Session) → 活动(Activity) → 企业申请(Application)
```

---

## 二、业务功能点

| 功能编号 | 功能名称 | 说明 |
|---------|---------|------|
| F001 | 场次管理 | 百场万企下有多个场次，场次决定活动类型 |
| F002 | 活动列表 | 分页展示百场万企活动(plate_id=1) |
| F003 | 活动创建 | 创建活动，根据type_id显示不同表单 |
| F004 | 活动编辑 | 修改活动信息 |
| F005 | 活动删除 | 删除活动（需二次确认） |
| F006 | 查看申请 | 查看该活动的企业申请列表 |
| F007 | 申请审核 | 对企业申请进行通过/拒绝/重新审核 |
| F008 | 申请导出 | 导出该活动的所有申请数据 |
| F009 | 前台活动页 | 展示活动列表和场次信息 |
| F010 | 前台场次详情 | 按场次展示旗下所有活动 |

---

## 三、场次绑定逻辑（核心）

### 3.1 场次与活动的关系

```
bcwq_session (场次)
    │
    ├── type_id: 0 → 产业链专场对接
    │       └── activities (theme=产业链名称, company=龙头企业)
    │
    ├── type_id: 1 → 区域专场对接
    │       └── activities (region=区域划分, desc=备注)
    │
    └── type_id: 2 → 主题专场
            └── activities (theme=活动主题)
```

### 3.2 表单字段动态显示

| type_id | 场次类型 | 显示字段 |
|---------|---------|---------|
| 0 | 产业链专场对接 | theme(拟对接产业链), company(龙头企业) |
| 1 | 区域专场对接 | region(区域划分), desc(备注) |
| 2 | 主题专场 | theme(活动主题) |

---

## 四、数据库设计

### 4.1 ER 图

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  bcwq_session   │───1:N──│   activities   │───1:N──│activity_applies │
│  (场次表)        │         │  (活动表)        │         │  (申请表)        │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id              │       │ id              │       │ id              │
│ name (场次名称)  │       │ plate_id=1      │       │ activity_id     │
│ type_id         │       │ session_id FK   │       │ user_id         │
│ status          │       │ type_id         │       │ title           │
│ created_at      │       │ type (专场类型)  │       │ territory       │
└─────────────────┘       │ theme           │       │ content         │
                          │ region          │       │ annex           │
                          │ company         │       │ status          │
                          │ address         │       │ created_at      │
                          │ start_time      │       └─────────────────┘
                          │ end_time        │
                          │ banner          │
                          │ brief           │
                          │ annex           │
                          │ examine_status  │
                          └─────────────────┘
```

### 4.2 表结构

```sql
-- 场次配置表
CREATE TABLE `bcwq_session` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL COMMENT '场次名称',
  `type_id` int(11) DEFAULT '0' COMMENT '类型:0产业链,1区域,2主题',
  `status` tinyint(1) DEFAULT '1' COMMENT '状态:0禁用,1启用',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 活动主表
CREATE TABLE `activities` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) DEFAULT NULL COMMENT '创建者ID',
  `plate_id` tinyint(1) NOT NULL DEFAULT '1' COMMENT '板块:1百场万企,2揭榜挂帅',
  `type_id` int(11) DEFAULT '0' COMMENT '活动类型ID(继承自session)',
  `type` varchar(100) DEFAULT NULL COMMENT '专场活动类型',
  `title` varchar(255) NOT NULL COMMENT '活动标题',
  `theme` varchar(255) DEFAULT NULL COMMENT '主题',
  `author` varchar(255) DEFAULT NULL COMMENT '主办单位',
  `company` varchar(255) DEFAULT NULL COMMENT '龙头企业',
  `session_id` int(11) DEFAULT NULL COMMENT '所属场次ID',
  `address` varchar(255) DEFAULT NULL COMMENT '举办地址',
  `area_id` int(11) DEFAULT NULL COMMENT '地域ID',
  `start_time` datetime DEFAULT NULL COMMENT '开始时间',
  `end_time` datetime DEFAULT NULL COMMENT '结束时间',
  `banner` varchar(255) DEFAULT NULL COMMENT '横幅图片',
  `brief` text COMMENT '专场简介',
  `annex` json DEFAULT NULL COMMENT '附件',
  `examine_status` tinyint(1) DEFAULT '1' COMMENT '审核状态',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 活动申请报名表
CREATE TABLE `activity_applies` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `activity_id` int(11) NOT NULL COMMENT '活动ID',
  `user_id` int(11) DEFAULT NULL COMMENT '申请人ID',
  `enterprise_id` int(11) DEFAULT NULL COMMENT '企业ID',
  `title` varchar(255) DEFAULT NULL COMMENT '申请标题',
  `territory` varchar(100) DEFAULT NULL COMMENT '对接领域',
  `content` longtext COMMENT '申请内容',
  `annex` json DEFAULT NULL COMMENT '附件',
  `status` tinyint(1) DEFAULT '0' COMMENT '状态:0待审核,1通过,2拒绝',
  `commit` tinyint(1) DEFAULT '1' COMMENT '是否提交',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 五、完整表单字段

| 字段名 | 必填 | 说明 | 备注 |
|--------|------|------|------|
| title | 是 | 活动标题 | 完整标题如"【产业链专场对接】xxx" |
| author | 是 | 主办单位 | 逗号分隔多个单位 |
| type_id | 是 | 场次类型 | 0产业链/1区域/2主题(自动设置) |
| theme | 条件 | 拟对接产业链 或 活动主题 | type_id=0或2时必填 |
| company | 否 | 拟邀请龙头企业 | type_id=0时显示 |
| region | 否 | 区域划分 | type_id=1时显示 |
| desc | 否 | 备注 | type_id=1时显示 |
| address | 是 | 举办省市 | 文本输入 |
| session_id | 是 | 专场活动 | 下拉选择场次 |
| month | 是 | 专场时间 | 月份选择器(2026-07) |
| start_time_desc | 否 | 专场时间描述 | 文本描述 |
| brief | 是 | 专场简介 | textarea |
| banner | 是 | 缩略图 | 上传图片 |
| content | 是 | 专场内容 | 富文本编辑器 |
| annex | 否 | 事项附件 | 多文件上传 |

---

## 六、API 文档

### 6.1 获取百场万企活动列表

- **URL**: `/api/activity`
- **Method**: `GET`
- **Request**: `GET /activity?page=1&plate_id=1`

### 6.2 创建/更新活动

- **URL**: `POST /api/activity` 或 `PATCH /api/activity/{id}`

### 6.3 按场次获取活动

- **URL**: `/api/activity/by-session`
- **Method**: `GET`
- **Request**: `GET /activity/by-session?plate_id=1&session_id=1`

### 6.4 获取活动申请列表

- **URL**: `/api/activity_applies`
- **Method**: `GET`
- **Request**: `GET /activity_applies?activity_id=1&plate_id=2&commit=true`

### 6.5 审核申请

- **URL**: `/api/activity_applies/{id}`
- **Method**: `PATCH`
- **Request**: `{"status": 1}` 或 `{"status": 2}` 或 `{"status": 0}`

### 6.6 获取场次列表

- **URL**: `/api/settings`
- **Method**: `GET`

---

## 七、业务流程

### 7.1 活动创建流程

```
1. 管理员进入百场万企编辑页
2. 选择"专场活动"(session)
3. 系统自动设置 form.type_id
4. 页面根据 type_id 动态显示相关字段
5. 管理员填写表单
6. 提交时前端传 month(2026-07)
7. 后端转换为 start_time 和 end_time
8. 保存活动
```

### 7.2 企业申请流程

```
1. 企业在前台浏览百场万企活动
2. 点击"我要参与"
3. 跳转管理后台申请页 /activity/join/bcwq/{activity_id}
4. 填写申请信息（标题、对接领域、内容、附件）
5. 提交申请（status=0待审核）
6. 管理员在后台审核（通过/拒绝）
```

---

## 八、路由对照表

### 8.1 后端API路由

| Method | URL | Controller@Method | 说明 |
|--------|-----|-------------------|------|
| GET | /api/activity | ActivityController@index | 获取活动列表 |
| POST | /api/activity | ActivityController@store | 创建活动 |
| PATCH | /api/activity/{id} | ActivityController@update | 更新活动 |
| DELETE | /api/activity/{id} | ActivityController@destroy | 删除活动 |
| GET | /api/activity_applies | ActivityAppliesController@index | 获取申请列表 |
| PATCH | /api/activity_applies/{id} | ActivityAppliesController@update | 审核申请 |
| GET | /api/bcwq | BcwqController@index | 前台首页数据 |
| GET | /api/bcwq/{id} | BcwqController@detail | 前台活动详情 |
| GET | /api/bcwq/session/{id} | BcwqController@demandSessionDetail | 前台场次详情 |
