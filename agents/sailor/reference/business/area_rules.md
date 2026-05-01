# 地域管理业务规则

## 一、业务功能点

| 功能编号 | 功能名称 | 说明 |
|---------|---------|------|
| F001 | 地域列表 | 分页展示地域列表，显示ID、地域名称、添加时间 |
| F002 | 地域创建 | 新增地域 |
| F003 | 地域修改 | 编辑地域名称 |
| F004 | 地域删除 | 已禁用（页面注释掉删除按钮） |
| F005 | 全域查询 | 获取所有地域数据（下拉框等场景使用） |

### 1.1 业务规则

- 地域数据被活动管理引用，作为活动的地域属性
- 地域名称唯一
- 删除功能已禁用，防止关联数据孤岛

---

## 二、数据流线稿

```
用户操作（地域管理后台）
    ↓
前端页面 /admin/area/index.vue
    ↓ 调用 get(page)
前端API area.js → GET /area
    ↓
后端路由 api.php → AreaController@index
    ↓
AreaController → AreaBusiness::getArea()
    ↓
Model Area → SQL → MySQL areas表
    ↓
响应返回地域列表数据
```

---

## 三、数据库设计

### 3.1 ER 图

```
┌─────────────┐       ┌─────────────┐
│    areas     │──1:N──│  activities │
└─────────────┘       └─────────────┘
```

### 3.2 表结构

```sql
CREATE TABLE `areas` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL COMMENT '地域名称',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 四、API 文档

### 4.1 获取地域列表

- **URL**: `/api/area`
- **Method**: `GET`
- **Request**: `page` (int, required): 页码

- **Response**:
```json
{
  "data": [
    { "id": 1, "name": "北京市", "created_at": "2026-01-01 10:00:00" }
  ],
  "total": 34,
  "per_page": 20,
  "current_page": 1
}
```

### 4.2 获取全部地域

- **URL**: `/api/areas/all`
- **Method**: `GET`
- **Response**:
```json
[
  { "id": 1, "name": "北京市" },
  { "id": 2, "name": "上海市" }
]
```

### 4.3 创建地域

- **URL**: `/api/area`
- **Method**: `POST`
- **Request**: `{ "name": "地域名称" }`

### 4.4 更新地域

- **URL**: `/api/area/{id}`
- **Method**: `PATCH`
- **Request**: `{ "name": "新地域名称" }`

### 4.5 获取地域详情

- **URL**: `/api/area/{id}`
- **Method**: `GET`

### 4.6 删除地域

- **URL**: `/api/area/{id}`
- **Method**: `DELETE`
- **注意**: 后端接口存在，但前端页面已禁用删除按钮
