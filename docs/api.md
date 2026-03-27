# TL比价模块 — 接口文档（含JSON示例）

---

## 接口0：添加仓库
- 方法：`POST`
- 路由：`/tl/add_warehouse`
- 传入：仓库名
- 输出：仓库id、是否新建
- 逻辑说明：按名称查 `dict_warehouses`，已存在则返回现有id，不存在则自动新建
- 模拟请求JSON：
```json
{ "仓库名": "北京仓" }
```
- 模拟返回JSON（已存在）：
```json
{ "code": 200, "msg": "仓库已存在", "仓库id": 101, "新建": false }
```
- 模拟返回JSON（新建）：
```json
{ "code": 200, "msg": "仓库新建成功", "仓库id": 103, "新建": true }
```

---

## 接口1：获取仓库列表
- 方法：`GET`
- 路由：`/tl/get_warehouses`
- 传入：无
- 输出：仓库id、仓库名
- 数据来源：`dict_warehouses` 表（`is_active=1`）
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": [
    { "仓库id": 101, "仓库名": "仓1" },
    { "仓库id": 102, "仓库名": "仓2" }
  ]
}
```

---

## 接口2：获取冶炼厂列表
- 方法：`GET`
- 路由：`/tl/get_smelters`
- 传入：无
- 输出：冶炼厂id、冶炼厂名
- 数据来源：`dict_factories` 表（`is_active=1`）
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": [
    { "冶炼厂id": 201, "冶炼厂": "华北冶炼厂" },
    { "冶炼厂id": 202, "冶炼厂": "华东冶炼厂" }
  ]
}
```

---

## 接口3：获取品类列表
- 方法：`GET`
- 路由：`/tl/get_categories`
- 传入：无
- 输出：品类id、品类名（聚合所有关联名称，用「、」分隔）
- 数据来源：`dict_categories` 表（`is_active=1`），按 `category_id` 分组，`GROUP_CONCAT(name)` 聚合
- 逻辑说明：相同品类的不同名称共用一个 `category_id`，读表时通过 id 将所有名称以"铜、紫铜、黄铜"的形式拼接输出
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": [
    { "品类id": 301, "品类名": "铜、紫铜、黄铜" },
    { "品类id": 302, "品类名": "铝、氧化铝、电解铝" },
    { "品类id": 303, "品类名": "锌" },
    { "品类id": 304, "品类名": "铅" }
  ]
}
```

---

## 接口4：获取比价表
- 方法：`POST`
- 路由：`/tl/get_comparison`
- 传入：选中仓库id列表、冶炼厂id列表、品类id列表
- 输出：(仓库、冶炼厂、品类、运费、报价) 列表
- 逻辑说明：
  - 运费取 `freight_rates` 中该(仓库,冶炼厂)最新生效日期的价格
  - 报价取 `quote_details` 中该(冶炼厂,品类)最新日期的单价，无记录则为 null
  - 结果为 (仓库×冶炼厂运费记录) × 品类 的笛卡尔积
- 模拟请求JSON：
```json
{
  "选中仓库id列表": [101, 102],
  "冶炼厂id列表": [201],
  "品类id列表": [301]
}
```
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": [
    { "仓库": "北京仓", "冶炼厂": "华北冶炼厂", "品类": "铜", "运费": 200, "报价": 9350 },
    { "仓库": "上海仓", "冶炼厂": "华北冶炼厂", "品类": "铜", "运费": 300, "报价": 9350 }
  ]
}
```

---

## 接口5：上传价格表（OCR识别）
- 方法：`POST`
- 路由：`/tl/upload_price_table`
- 传入：图片文件（FormData，支持批量）
- 输出：OCR识别的原始冶炼厂名、品类名、价格，供前端展示修改后传给接口5b
- 支持格式：jpg、png、bmp、webp
- 逻辑说明：
  1. 后端为每张图片生成UUID文件名，保存到 `uploads/price_tables/` 目录
  2. 调用 RapidOCR 对每张图片进行文字识别，提取工厂名、日期、品类+价格
  3. 直接返回识别原文，不做任何匹配，匹配逻辑在接口5b中处理
- 模拟请求（FormData）：
```
file: [图片文件1, 图片文件2]
```
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": {
    "items": [
      { "冶炼厂名": "山西亿晨环保科技有限公司", "品类名": "电动车", "价格": 9350 },
      { "冶炼厂名": "山西亿晨环保科技有限公司", "品类名": "摩托车", "价格": 8600 }
    ],
    "details": [
      {
        "image": "报价单1.jpg",
        "factory_name": "山西亿晨环保科技有限公司",
        "date": "2026-03-24",
        "items": [
          { "冶炼厂名": "山西亿晨环保科技有限公司", "品类名": "电动车", "价格": 9350 },
          { "冶炼厂名": "山西亿晨环保科技有限公司", "品类名": "摩托车", "价格": 8600 }
        ]
      }
    ]
  }
}
```

---

## 接口5b：确认价格表写入
- 方法：`POST`
- 路由：`/tl/confirm_price_table`
- 传入：报价日期、报价明细列表（冶炼厂名/id、品类名/id、价格）
- 输出：写入状态
- 逻辑说明：
  1. 前端根据接口5返回结果确认/修正后调用此接口
  2. `冶炼厂id` 为 null 时：按名称查 `dict_factories`，存在则复用，不存在则自动新建
  3. `品类id` 为 null 时：按名称查 `dict_categories.row_id`，存在则复用，不存在则自动新建
  4. 以 `(报价日期, 冶炼厂id, 品类row_id)` 为唯一键写入 `quote_details`，已存在则更新价格
- 模拟请求JSON：
```json
{
  "报价日期": "2026-03-24",
  "数据": [
    { "冶炼厂名": "山西亿晨环保科技有限公司", "冶炼厂id": 1, "品类名": "电动车", "品类id": 3, "价格": 9350 },
    { "冶炼厂名": "山西亿晨环保科技有限公司", "冶炼厂id": 1, "品类名": "摩托车", "品类id": null, "价格": 8600 },
    { "冶炼厂名": "未知工厂", "冶炼厂id": null, "品类名": "电动车", "品类id": 3, "价格": 9100 }
  ]
}
```
- 模拟返回JSON：
```json
{
  "code": 200,
  "msg": "写入成功：新增 8 条，更新 2 条"
}
```

---

## 接口6：上传运费
- 方法：`POST`
- 路由：`/tl/upload_freight`
- 传入：`{仓库, 冶炼厂, 运费}` **列表**（支持批量）
- 输出：状态提示
- 逻辑说明：根据仓库名和冶炼厂名查找对应ID，以当日日期为生效日期，写入 `freight_rates` 表；同一 (仓库, 冶炼厂, 日期) 已存在则更新运费
- 模拟请求JSON：
```json
[
  { "仓库": "北京仓", "冶炼厂": "华北冶炼厂", "运费": 200 },
  { "仓库": "上海仓", "冶炼厂": "华东冶炼厂", "运费": 300 }
]
```
- 模拟返回JSON：
```json
{
  "code": 200,
  "msg": "运费数据已存入数据库"
}
```

---

## 接口7a：获取品类映射表
- 方法：`GET`
- 路由：`/tl/get_category_mapping`
- 传入：无
- 输出：所有品类id及其对应的全部名称列表（第一个为主名称）
- 数据来源：`dict_categories` 表（`is_active=1`），按 `category_id` 分组，`is_main=1` 的排在首位
- 用途：前端展示当前映射关系，供用户查看和修改后调用接口7更新
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": [
    { "品类id": 301, "品类名称": ["铜", "紫铜", "黄铜"] },
    { "品类id": 302, "品类名称": ["铝", "氧化铝", "电解铝"] },
    { "品类id": 303, "品类名称": ["锌"] },
    { "品类id": 304, "品类名称": ["铅", "软铅"] }
  ]
}
```

---

## 接口7：更新品类映射表
- 方法：`POST`
- 路由：`/tl/update_category_mapping`
- 传入：品类映射列表（支持批量，每条含品类id + 品类名称列表）
- 输出：状态提示
- 逻辑说明：
  1. 名称列表中 **第一个为主名称**（`is_main=1`），用于比价表展示，其余为 `is_main=0`
  2. 该 `category_id` 下旧记录的 `is_main` 先全部置0
  3. 名称已存在则更新其 `category_id` 和 `is_main`；不存在则插入新行
- 模拟请求JSON：
```json
[
  { "品类id": 301, "品类名称": ["铜", "紫铜", "黄铜"] },
  { "品类id": 302, "品类名称": ["铝", "氧化铝", "电解铝"] }
]
```
- 模拟返回JSON：
```json
{
  "code": 200,
  "msg": "品类映射表更新成功，数据已存入数据库"
}
```

---

## 接口A7：采购建议
- 方法：`POST`
- 路由：`/tl/get_purchase_suggestion`
- 传入：仓库id列表、需求列表（冶炼厂id、品类id、需求吨数）
- 输出：LLM生成的各仓库发车意见 + 原始结构化数据
- 逻辑说明：
  1. 查询各(仓库,冶炼厂)最新运费、各(冶炼厂,品类)最新报价
  2. 计算综合成本（报价+运费），整理结构化数据
  3. 将数据交由大语言模型分析，要求：同仓库货物混装、尽量整车（20-30吨）、优先低成本方案
  4. 返回 LLM 生成的各仓库发车意见表文字 + 原始数据列表
- 模拟请求JSON：
```json
{
  "warehouse_ids": [101, 102],
  "demands": [
    { "smelter_id": 201, "category_id": 301, "demand": 5.0 },
    { "smelter_id": 202, "category_id": 301, "demand": 3.0 }
  ]
}
```
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": {
    "suggestion": "## 各仓库发车意见表\n\n**北京仓**\n| 装车方案 | 品类 | 吨数 | 目的冶炼厂 | 综合成本 | 备注 |\n...",
    "raw": [
      { "仓库": "北京仓", "冶炼厂": "华北冶炼厂", "品类": "铜", "需求吨数": 5.0, "报价(元/吨)": 9350, "运费(元/吨)": 200, "综合成本(元/吨)": 9550 }
    ]
  }
}
```

---

# 用户认证模块 — 接口文档

---

## 接口A1：登录
- 方法：`POST`
- 路由：`/auth/login`
- 传入：username、password
- 输出：JWT token、用户信息
- 逻辑说明：校验账号密码，成功返回 JWT token 及用户基本信息；失败返回 401

---

## 接口A0：注册
- 方法：`POST`
- 路由：`/auth/register`
- 传入：username、real_name、password、phone（可选）
- 输出：新建用户id
- 逻辑说明：账号唯一，重复则返回 400；新用户默认角色为 `user`；密码后端加盐存储
- 模拟请求JSON：
```json
{ "username": "user2", "real_name": "张三", "password": "123456", "phone": "13800138003" }
```
- 模拟返回JSON（成功）：
```json
{ "code": 200, "msg": "注册成功", "id": 3 }
```
- 模拟返回JSON（账号重复）：
```json
{ "code": 400, "msg": "账号已存在" }
```
- 模拟请求JSON：
```json
{ "username": "admin", "password": "123456" }
```
- 模拟返回JSON（成功）：
```json
{
  "code": 200,
  "msg": "登录成功",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id": 1, "username": "admin", "real_name": "管理员", "role": "admin", "phone": "13800138001", "email": "admin@example.com" }
}
```
- 模拟返回JSON（失败）：
```json
{ "detail": "账号或密码错误" }
```

---

## 接口A2：获取用户列表
- 方法：`GET`
- 路由：`/auth/users`
- 权限：仅 admin
- 传入（Query参数）：keyword（可选，账号/姓名/手机模糊搜索）、role（可选，`admin`/`user`）、page（默认1）、page_size（默认10）
- 输出：用户列表、总条数
- 请求头：`Authorization: Bearer <token>`
- 模拟返回JSON：
```json
{
  "code": 200,
  "data": {
    "total": 1,
    "list": [
      { "id": 1, "username": "admin", "real_name": "管理员", "role": "admin", "phone": "13800138001", "email": "admin@example.com" }
    ]
  }
}
```

---

## 接口A3：新增用户
- 方法：`POST`
- 路由：`/auth/users`
- 权限：仅 admin
- 传入：username、password、real_name（可选）、role（admin/user，默认user）、phone（可选）、email（可选）
- 输出：新建用户id
- 请求头：`Authorization: Bearer <token>`
- 模拟请求JSON：
```json
{ "username": "user2", "real_name": "张三", "password": "123456", "role": "user", "phone": "13800138003", "email": "zhangsan@example.com" }
```
- 模拟返回JSON：
```json
{ "code": 200, "msg": "用户创建成功", "id": 3 }
```

---

## 接口A4：修改用户角色
- 方法：`POST`
- 路由：`/auth/update_role`
- 权限：仅 admin
- 传入：id（用户id）、role（新角色）
- 请求头：`Authorization: Bearer <token>`
- 模拟请求JSON：
```json
{ "id": 2, "role": "admin" }
```
- 模拟返回JSON：
```json
{ "code": 200, "msg": "角色修改成功" }
```

---

## 接口A5：修改用户密码
- 方法：`POST`
- 路由：`/auth/change_password`
- 传入：id（用户id）、admin_key（服务端配置的固定密钥）、new_password
- 逻辑说明：校验 admin_key 与服务端 `JWT_SECRET_KEY` 一致后更新密码
- 模拟请求JSON：
```json
{ "id": 2, "admin_key": "your-secret-key", "new_password": "newpass123" }
```
- 模拟返回JSON：
```json
{ "code": 200, "msg": "密码修改成功" }
```

---

## 接口A6：删除用户
- 方法：`POST`
- 路由：`/auth/delete_user`
- 权限：仅 admin，且不可删除自己
- 传入：id（用户id）
- 逻辑说明：软删除，将 `is_active` 置0
- 请求头：`Authorization: Bearer <token>`
- 模拟请求JSON：
```json
{ "id": 3 }
```
- 模拟返回JSON：
```json
{ "code": 200, "msg": "用户已删除" }
```

---

## 补充说明
1. 所有JSON中的 `code: 200` 为通用成功状态码
2. 需登录的接口请求头须携带 `Authorization: Bearer <token>`
3. token 过期或无效返回 `401`，非 admin 调用管理接口返回 `403`
4. 错误状态码：`400`（参数校验失败）、`401`（未登录）、`403`（权限不足）、`404`（资源不存在）、`500`（服务器内部错误）
5. 数据库连接使用 `autocommit=True`，写操作自动提交
6. LLM 配置通过环境变量注入：`LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MODEL`
