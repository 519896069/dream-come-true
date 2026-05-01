# 系统设置业务规则

## 一、业务功能点

| 功能编号 | 功能名称 | 说明 |
|---------|---------|------|
| F001 | 系统配置查询 | 获取系统全局配置（地域列表、会话/场次列表） |

### 1.1 配置项说明

| 配置项 | 数据来源 | 用途 |
|--------|---------|------|
| areas | areas表 | 地域下拉选择 |
| sessions | bcwq_session表 | 百场万企活动场次选择 |

---

## 二、数据流线稿

```
前端页面（全局引用）
    ↓
前端API setting.js → GET /settings
    ↓
后端路由 api.php → SettingController@settings
    ↓
SettingController 查询 Area + BcwqSession
    ↓
响应返回配置数据
```

---

## 三、数据库设计

### 3.1 ER 图

```
┌─────────────────┐
│   areas 表       │ ← 系统设置-地域配置
└─────────────────┘

┌─────────────────┐
│  bcwq_session 表 │ ← 系统设置-场次配置
└─────────────────┘
```

### 3.2 表结构

```sql
-- 地域表 (见地域管理模块)

CREATE TABLE `bcwq_session` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL COMMENT '场次名称',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 四、API 文档

### 4.1 获取系统配置

- **URL**: `/api/settings`
- **Method**: `GET`
- **Response**:
```json
{
  "areas": [
    { "id": 1, "name": "北京市" },
    { "id": 2, "name": "上海市" }
  ],
  "sessions": [
    { "id": 1, "name": "第一场" },
    { "id": 2, "name": "第二场" }
  ]
}
```

### 4.2 使用场景

| 场景 | 使用的配置 |
|------|----------|
| 活动管理-选择地域 | areas |
| 活动管理-选择场次 | sessions |
| 企业入驻-选择城市 | areas |
| 表单下拉选项 | areas / sessions |

---

## 五、与其他模块的关系

| 关联模块 | 关系说明 |
|---------|---------|
| 活动管理 | 活动创建时选择 session_id（场次） |
| 地域管理 | 活动选择地域时使用 areas 数据 |
| 文章管理 | 无直接关联 |
