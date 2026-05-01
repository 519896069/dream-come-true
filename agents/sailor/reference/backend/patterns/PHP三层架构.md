# PHP 三层架构规范

## 目录结构

```
backend/gongxinphp/
├── controllers/    # 控制器层
├── services/      # 业务逻辑层
├── models/        # 数据模型层
├── repositories/   # 数据访问层
└── middleware/    # 中间件
```

## 分层职责

### Controller（控制器层）
- 接收 HTTP 请求
- 参数校验
- 调用 Service 层
- 返回响应

### Service（业务逻辑层）
- 业务规则处理
- 事务管理
- 组合多个 Repository 操作

### Repository（数据访问层）
- 数据库 CRUD 操作
- 复杂查询封装

## 核心原则

1. Controller 只做请求/响应处理
2. 业务逻辑在 Service 层
3. 数据库操作在 Repository 层
4. 禁止在 Controller 中直接操作数据库
