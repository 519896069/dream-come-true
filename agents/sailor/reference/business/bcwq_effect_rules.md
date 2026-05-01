# 百场万企成效管理业务规则

## 一、业务功能点

| 功能编号 | 功能名称 | 说明 |
|---------|---------|------|
| F001 | 编写成效表 | 省级用户填写"百场万企"大中小企业融通对接活动的月度成效数据 |
| F002 | 成效专区 | 省级用户查看本人已提交的成效表列表 |
| F003 | 成效汇总 | 管理员查看所有省份提交的成效表列表 |
| F004 | 成效专区（管理员） | 管理员查看汇总数据及未提交省份名单 |
| F005 | 打印成效表 | 将单条成效表导出为 Excel 文件 |
| F006 | 打印成效汇总 | 将月度汇总数据导出为 Excel 文件 |

---

## 二、数据流线稿

```
【省级用户视角】
成效专区(scor.vue) ──GET /score/effect──→ EffectBusiness::score ──→ effect表
       │
       ↓
查看成效汇总表(popover弹窗) ──展示该用户提交的所有月份成效数据──
       │
       ↓
打印 ──GET /printing/effect-score?month=──→ EffectBusiness::printScore ──→ Excel

编写成效表(edit.vue) ──POST /effect──→ EffectBusiness::create ──→ effect表

【管理员视角】
成效专区（adminscore.vue) ──GET /admin-effect──→ EffectBusiness::adminStore
成效汇总(index.vue) ──GET /effect?plate_id=1──→ EffectBusiness::index ──→ effect表
```

---

## 三、数据库设计

### 3.1 ER 图

```
┌──────────────────────────────────────────────────────────────┐
│                          users                                │
│──────────────────────────────────────────────────────────────│
│ id(PK) │ enterprise_name │ enterprise_province_area_id │ rule│
└────────┴─────────────────┴──────────────────────────────┴─────┘
         │ 1
         │ N
         ↓
┌──────────────────────────────────────────────────────────────┐
│                         effect                               │
│──────────────────────────────────────────────────────────────│
│ id(PK) │ user_id(FK) │ effect_date │ month │ content(JSON) │ case │
└────────┴─────────────┴─────────────┴───────┴────────────────┴──────┘
```

### 3.2 表结构

**effect 表（成效表）**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 填报人ID（关联users.id） |
| effect_date | datetime | 成效表时间 |
| month | date | 月份（格式：YYYY-MM-01） |
| content | longtext | 成效内容（JSON格式） |
| case | longtext | 典型案例及做法 |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

**content JSON 结构示例**：
```json
{
  "rtdj_count": 10,
  "sjdj_count": 5,
  "sxdj_count": 3,
  "top_count": 20,
  "ddcg_zxqy_count": 50,
  "ddcg_zjtx_count": 10,
  "ddcg_xjr_count": 2,
  "ddcg_finish_count": 30,
  "ddcg_order_count": 20,
  "ddcg_order_amount": 500
}
```

---

## 四、API 文档

### 4.1 获取成效列表（省级用户）

- **URL**: `/api/score/effect`
- **Method**: GET
- **Request**: `?page=1`

### 4.2 获取成效列表（管理员）

- **URL**: `/api/admin-effect`
- **Method**: GET
- **Request**: `?page=1&month=2026-03`
- **Response**:
```json
{
  "list": [...],
  "unsubmit": ["广东省", "浙江省"],
  "month": "2026-03"
}
```

### 4.3 编写成效表（创建）

- **URL**: `/api/effect`
- **Method**: POST
- **Request**: `{ "effect_date", "content", "case" }`
- **业务逻辑**:
  - 同一用户同一月份只能提交一次
  - month 自动根据 effect_date 生成

### 4.4 修改成效表

- **URL**: `/api/effect/{id}`
- **Method**: PATCH

### 4.5 删除成效表

- **URL**: `/api/effect/{id}`
- **Method**: DELETE

### 4.6 打印单条成效表

- **URL**: `/api/printing/effect/{id}`
- **Method**: GET
- **Response**: Excel 文件下载

### 4.7 打印月度汇总表

- **URL**: `/api/printing/effect-score`
- **Method**: GET
- **Request**: `?month=2026-03`
- **Response**: Excel 文件下载

---

## 五、合作事项类别说明

| 类别代码 | 类别名称 | 说明 |
|---------|---------|------|
| ddcg | 订单采购 | 大企业采购中小企业的原材料、零部件、生产设备等产品与服务 |
| xzcx | 协作创新 | 大企业与中小企业联合研发、委托研发、创新平台共建、成果转化等 |
| zygx | 资源共享 | 大企业向中小企业开放实验室、研发平台、中试平台、仪器仪表、数据等 |
| crhz | 产融合作 | 股权投资、产业基金、信用贷款、债券融资、供应链金融等支持 |
| cyfn | 产业赋能 | 数字化、绿色化、知识产权、法律、质量认证和质量检测等咨询服务 |
| djzh | 专利对接 | 高校和科研机构存量专利向中小企业对接转化 |
| other | 其他 | 其他合作事项 |

---

## 六、关键业务规则

1. **月度唯一性**：同一用户同一月份只能有一条成效记录
2. **省级用户**：只能查看和编辑本人提交的成效表
3. **管理员**：可查看所有省份的成效数据及未提交省份名单
4. **打印功能**：支持导出 Excel 格式的统计表
