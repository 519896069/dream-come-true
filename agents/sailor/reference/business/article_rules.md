# 文章管理业务规则

## 一、业务功能点

| 功能编号 | 功能名称 | 说明 |
|---------|---------|------|
| F001 | 文章列表 | 分页展示文章列表，显示ID、标题、作者、标签、发布时间 |
| F002 | 文章创建 | 创建文章，支持富文本内容、附件上传、置顶设置 |
| F003 | 文章修改 | 编辑已有文章信息 |
| F004 | 文章删除 | 删除文章（需二次确认） |
| F005 | 文章预览 | 点击标题跳转到前台详情页预览 |

### 1.1 文章类型

| 类型 | 说明 | 特殊字段 |
|------|------|---------|
| type=0 | 政策文章 | - |
| type=1 | 普通文章 | rtal_type (关联类型) |
| type=2 | 活动文章 | - |

### 1.2 业务规则

- 文章支持置顶（top字段）
- 文章支持标签管理（tags数组）
- 文章支持附件上传（annex数组）
- 发布后跳转前台政策或文章详情页

---

## 二、数据流线稿

```
用户操作（列表页）
    ↓
前端页面 /admin/article/index.vue
    ↓ 调用 get(page)
前端API article.js → GET /article
    ↓
后端路由 api.php → ArticleController@index
    ↓
ArticleController → ArticleBusiness::getArticle()
    ↓
Model Article → SQL → MySQL articles表
    ↓
响应返回文章列表数据
```

---

## 三、数据库设计

### 3.1 ER 图

```
┌─────────────┐
│  articles   │
├─────────────┤
│ id (PK)     │
│ user_id     │───→ users (关联创建者)
│ title       │
│ author      │
│ type        │
│ rtal_type   │
│ sort_content│
│ content     │
│ publish_time│
│ top         │
│ tags (JSON) │
│ annex (JSON)│
│ created_at  │
│ updated_at  │
└─────────────┘
```

### 3.2 表结构

```sql
CREATE TABLE `articles` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) DEFAULT NULL COMMENT '创建者ID',
  `title` varchar(255) NOT NULL COMMENT '文章标题',
  `author` varchar(100) NOT NULL COMMENT '作者',
  `type` tinyint(1) NOT NULL DEFAULT '0' COMMENT '类型:0政策,1文章,2活动',
  `rtal_type` int(11) DEFAULT NULL COMMENT '关联类型(type=1时必填)',
  `sort_content` text COMMENT '摘要内容',
  `content` longtext NOT NULL COMMENT '正文内容',
  `publish_time` datetime NOT NULL COMMENT '发布时间',
  `top` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否置顶:0否,1是',
  `tags` json DEFAULT NULL COMMENT '标签数组',
  `annex` json DEFAULT NULL COMMENT '附件数组',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 四、API 文档

### 4.1 获取文章列表

- **URL**: `/api/article`
- **Method**: `GET`
- **Request**: `page` (int, required): 页码

- **Response**:
```json
{
  "data": [
    {
      "id": 1,
      "title": "政策标题",
      "author": "作者名",
      "type": 0,
      "tags": ["标签1", "标签2"],
      "publish_time": "2026-04-06 10:00:00",
      "top": 0
    }
  ],
  "total": 100,
  "per_page": 20,
  "current_page": 1
}
```

### 4.2 创建文章

- **URL**: `/api/article`
- **Method**: `POST`
- **Request**:
```json
{
  "title": "文章标题",
  "author": "作者",
  "type": 0,
  "rtal_type": null,
  "sort_content": "摘要",
  "publish_time": "2026-04-06 10:00:00",
  "content": "正文内容",
  "top": 0,
  "tags": ["标签1"],
  "annex": []
}
```

### 4.3 更新文章

- **URL**: `/api/article/{id}`
- **Method**: `PATCH`

### 4.4 获取文章详情

- **URL**: `/api/article/{id}`
- **Method**: `GET`

### 4.5 删除文章

- **URL**: `/api/article/{id}`
- **Method**: `DELETE`

### 4.6 获取文章详情

- **URL**: `/api/article/{id}`
- **Method**: `GET`
- **Response**:
```json
{
  "id": 1,
  "title": "文章标题",
  "author": "作者",
  "type": 0,
  "content": "正文HTML",
  "publish_time": "2026-04-06 10:00:00",
  "tags": ["标签1"],
  "annex": []
}
```
