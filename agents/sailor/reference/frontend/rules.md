# 前端规范

## Vue 2 + element-ui 规范

### 组件命名
- 组件文件：PascalCase
- 组件名：PascalCase
- 样式 scoped

### Props 规范
```javascript
props: {
  title: {
    type: String,
    required: true
  },
  data: {
    type: Array,
    default: () => []
  }
}
```

### API 封装
- 统一放在 api/ 目录
- 按模块分离
- 使用 async/await

### 目录结构
```
frontend/gxadmin/
├── api/          # API 接口封装
├── components/   # 公共组件
├── views/        # 页面组件
├── router/       # 路由配置
└── utils/        # 工具函数
```

## Nuxt.js 主站规范

### 目录结构
```
frontend/gxhome/
├── pages/        # 页面
├── components/   # 组件
├── layouts/      # 布局
├── middleware/   # 中间件
├── plugins/      # 插件
└── static/       # 静态资源
```

### SSR 注意事项
- 客户端特有代码用 `process.client`
- 注意 `asyncData` 和 `fetch` 使用
- SEO 友好的 meta 管理
