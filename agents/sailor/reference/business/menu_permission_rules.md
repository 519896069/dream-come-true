# 前端菜单权限控制业务规则

## 一、业务概述

### 1.1 核心问题

**这是什么问题？**
控制台菜单需要根据登录用户的角色（管理员/企业用户/部委用户）和企业类型（大企业/小微企业）动态显示不同内容。

**谁在用？**
- 系统管理员（后台运营人员）
- 各类企业用户（大企业、小微企业）
- 部委/省级监管用户

**解决什么？**
- 权限隔离：不同角色只能看到自己权限范围内的功能
- 数据安全：防止越权访问敏感管理功能
- 个性化体验：企业用户只看业务功能，管理员只看管理功能

---

## 二、用户角色体系

### 2.1 用户角色（rule字段）

| rule值 | 角色名称 | 说明 |
|--------|----------|------|
| 0 | 企业用户 | 普通企业人员 |
| 1 | 管理员 | 系统管理员 |
| 2 | 部委用户 | 国家级监管 |
| 3 | 省级用户 | 省级监管 |

### 2.2 企业类型（enterprise_type字段）

| 值 | 类型名称 | 说明 |
|----|----------|------|
| 0 | 小微企业 | 小型微利企业 |
| 1 | 大企业 | 大型企业 |

---

## 三、权限控制逻辑

### 3.1 路由权限控制函数

位置：`frontend/gxadmin/src/permission.js`

```javascript
const permission = (perm, rule, ent_type) => {
  if (rule == 0) {  // 仅对企业用户(rule=0)做特殊限制
    if (perm == 'big' && ent_type == 1) return false    // 大企业用户不能访问big权限菜单
    if (perm == 'small' && ent_type == 0) return false  // 小微企业用户不能访问small权限菜单
    if (perm == 'admin') return false                   // 企业用户不能访问admin权限菜单
  }
  return true
}
```

### 3.2 权限标识与菜单对照

| permission值 | 可访问角色 | 典型菜单 |
|---------------|-----------|----------|
| `admin` | rule=1 (管理员) | 企业管理、供需对接管理、轮播图管理 |
| `supper` | rule=1 (管理员) | 系统设置、征集活动管理 |
| `ent` | rule=0 (企业用户) | 揭榜管理、对接供需管理 |
| `big` | rule=0 且 enterprise_type=1 (大企业) | - |
| `small` | rule=0 且 enterprise_type=0 (小微企业) | 我要加入 |
| `ministerial` | rule=2 (部委用户) | 成效汇总(管理员) |

### 3.3 路由守卫流程

```
用户访问页面
    ↓
检查 Token 存在？
  ├─ 否 → 跳转登录页
  └─ 是 → 检查用户信息是否存在？
          ├─ 否 → 调用 /info 接口获取用户信息
          └─ 是 → 执行权限校验
                  ↓
            permission(meta.permission, user.rule, user.enterprise_type)
                  ↓
            ├─ false → 跳转 /my
            └─ true → 允许访问
```

---

## 四、菜单分类

### 4.1 管理员专属菜单（rule=1）

| 菜单路径 | 菜单名称 | permission标识 |
|----------|----------|----------------|
| /enterprise | 企业管理 | admin |
| /admin_demand | 供需对接管理 | admin |
| /banner | 轮播图管理 | admin |
| /bcwq | 百场万企 | admin |
| /jbgs | 发榜揭榜 | admin |
| /activity | 活动管理 | admin |
| /article | 文章管理 | admin |
| /area | 地域管理 | admin |
| /system | 系统设置 | supper |
| /collect | 征集活动管理 | supper |

### 4.2 企业用户菜单（rule=0）

| 菜单路径 | 菜单名称 | permission标识 | 适用企业类型 |
|----------|----------|----------------|--------------|
| /my | 个人中心 | ent | 全部 |
| /activity_applies | 揭榜管理 | ent | 全部 |
| /demand | 对接供需管理 | ent | 全部 |
| /ent/activity/jbgs | 发榜管理 | ent | 全部 |
| /join | 我要加入 | small | 小微企业 |
| /activity/join | 我要参加 | - | 全部 |

---

## 五、数据流线稿

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  用户登录   │────▶│  获取Token  │────▶│  路由守卫   │────▶│  权限校验   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                   │
                           ┌───────────────────────────────────────┘
                           ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  菜单过滤   │◀────│  路由表     │◀────│  meta.permission│   │  filterRoutes│
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
      │
      ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  侧边栏    │────▶│  动态菜单   │────▶│  页面渲染   │
└─────────────┘     └─────────────┘     └─────────────┘
```

---

## 六、权限控制决策矩阵

| 用户类型 | rule | enterprise_type | admin | supper | ent | big | small | ministerial |
|----------|------|----------------|-------|--------|-----|-----|-------|-------------|
| 小微企业用户 | 0 | 0 | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ |
| 大企业用户 | 0 | 1 | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| 系统管理员 | 1 | - | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| 部委用户 | 2 | - | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| 省级用户 | 3 | - | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## 七、数据库用户表结构

### 7.1 users 表关键字段

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint | 主键 |
| email | varchar | 邮箱（登录账号） |
| password | varchar | 密码（加密存储） |
| salt | varchar | 盐值 |
| name | varchar | 姓名 |
| rule | tinyint | 角色：0=企业 1=管理员 2=部委 3=省级 |
| enterprise_type | tinyint | 企业类型：0=小微 1=大企业 |
| enterprise_name | varchar | 企业名称 |
| enterprise_status | tinyint | 企业状态 |
| enterprise_field | varchar | 所属行业 |
| enterprise_number | varchar | 统一社会信用代码 |
| is_cancel | tinyint | 是否注销：0=正常 1=已注销 |

---

## 八、路由守卫关键代码

位置：`frontend/gxadmin/src/permission.js`

```javascript
router.beforeEach(async(to, from, next) => {
  if (hasToken) {
    if (to.path === '/login') {
      next({ path: '/' })
    } else {
      const hasGetUserInfo = store.getters.name
      if (hasGetUserInfo) {
        if ((to.path === '/' || to.path === '/index') && store.getters.user.rule === 0) {
          next({ path: '/my' })  // 企业用户跳转个人中心
        } else {
          if (!permission(to.meta.permission, store.getters.user.rule, store.getters.user.enterprise_type)) {
            next({ path: '/my' })
          } else {
            next()
          }
        }
      } else {
        await store.dispatch('user/getInfo')
      }
    }
  } else {
    next(`/login?redirect=${to.path}`)
  }
})
```
