# 数字快印平台 - 架构设计文档
**Digital Fastprint Platform Architecture**

版本：v1.0  
日期：2025-10-26  
状态：MVP 设计阶段

---

## 1. 执行摘要（非技术人也能看懂）

### 系统目标
为个人与小微企业提供"一站式在线打印服务"，用户上传文件→选择打印选项（纸张/颜色/装订）→下单支付→获取履约通知→自提或配送，同时提供商家/运营后台管理订单与产品。

### MVP 范围（必做能力）
1. **用户注册/登录**：手机号或邮箱 + 密码，JWT 鉴权
2. **产品浏览与定制**：展示打印服务（文档/名片/海报等），动态配置选项（纸张、尺寸、颜色、数量）
3. **文件上传与预览**：支持 PDF/Word/图片，限额 50MB/文件
4. **下单与支付**：实时计价、创建订单、对接支付网关（支付宝/微信）
5. **订单管理**：用户查看订单状态；后台接单、修改状态、标记完成
6. **基础后台**：产品/选项/订单/用户管理，简易报表（订单量/金额）

### 关键架构决策
1. **单体优先**：Next.js App Router 统一前后台，API Routes 承载业务逻辑，3 个月内无须微服务化
2. **云中立存储**：Postgres 作为主库（可本地可托管），对象存储使用 S3 兼容接口（避免绑定 Supabase Storage 专属特性）
3. **渐进式权限**：JWT + RBAC（用户/商家/管理员三角色），MVP 阶段 RLS 思路优先做在应用层（为后续迁移留后路）
4. **支付解耦**：支付网关抽象适配层，今天用支付宝/微信，明天可换 Stripe/PayPal
5. **演进友好**：Clean Architecture 分层（领域层不依赖框架），为未来拆分微服务或换框架留接口

### 上线路径与里程碑
- **T0（0–2周）**：项目骨架、Auth、数据模型、本地开发环境就绪
- **T1（3–6周）**：产品模块、文件上传、下单流程、支付对接（Happy Path）
- **T2（7–10周）**：后台 CRUD、订单状态机、审计日志、基础监控与上线
- **T3（10–12周）**：灰度测试、修复紧急 Bug、上线正式环境

---

## 2. 业务域与用户旅程（Domain-first）

### 2.1 角色画像

| 角色         | 权限级别 | 典型场景                                           |
|------------|------|------------------------------------------------|
| **访客**     | 0    | 浏览产品、查看价格、不可下单                                 |
| **注册用户**   | 1    | 上传文件、下单、支付、查看自己的订单历史                           |
| **商家操作员**  | 2    | 查看所有订单、更新订单状态（接单/打印中/完成）、管理产品/选项              |
| **平台管理员**  | 9    | 全局权限：用户管理、角色分配、系统配置、审计日志查询、财务报表              |

### 2.2 关键用例

| 用例           | 触发条件                 | 动作                                    | 期望结果                      |
|--------------|----------------------|-----------------------------------------|---------------------------|
| 用户注册         | 访客点击"注册"             | 填写手机号/邮箱、密码、验证码，提交表单                   | 创建用户记录、返回 JWT，跳转到首页       |
| 浏览产品与报价      | 用户登录后进入产品页           | 展示打印服务列表（名片/宣传单/文档），选择选项后实时计算价格         | 显示预估价格、引导上传文件             |
| 上传打印文件       | 用户在产品详情页点击"上传文件"     | 客户端上传到对象存储（预签名URL），服务端记录 file 元数据       | 文件上传成功，显示缩略图/预览            |
| 创建订单并支付      | 用户点击"提交订单"           | 服务端生成订单（冻结价格+文件引用），调支付网关生成支付链接/二维码      | 跳转支付页，支付成功后订单状态→ `paid`   |
| 商家接单与履约      | 商家后台看到新订单（状态=paid）   | 下载打印文件、更新订单状态为 `processing` → `completed` | 用户收到通知、订单状态同步             |
| 用户查询订单       | 用户进入"我的订单"           | 查询数据库（WHERE user_id=当前用户）               | 展示订单列表、可点击查看详情            |
| 管理员审计查询      | 管理员进入审计日志模块          | 输入时间范围/用户ID/操作类型，查询 audit_logs         | 显示操作记录（谁、何时、做了什么、改了什么字段）  |

### 2.3 用户旅程

```
[访客] 搜索引擎/广告 → 落地页（展示服务与价格）
          ↓
      注册/登录（手机号/邮箱+密码）
          ↓
[注册用户] 浏览产品列表 → 选择服务（如"彩色名片"）
          ↓
      选择选项：纸张（300g铜版纸）、尺寸（90×54mm）、数量（100张）
          ↓
      实时显示价格：¥35.00
          ↓
      上传设计文件（PDF/JPG，< 50MB）
          ↓
      提交订单 → 跳转支付（支付宝/微信）
          ↓
      支付成功 → 订单状态 = `paid`，收到确认短信/邮件
          ↓
[商家] 后台收到新订单通知 → 下载文件 → 打印制作
          ↓
      更新订单状态：`processing` → `completed`
          ↓
[用户] 收到完成通知 → 到店自提或等待配送
          ↓
      （可选）售后/退款/复购
```

### 2.4 MVP 业务边界

#### 本期 **做**
- ✅ 手机号/邮箱注册登录（无第三方 OAuth）
- ✅ 产品展示与动态选项配置（前端计算实时价格，后端校验）
- ✅ 文件上传（PDF/图片，单文件 50MB 限制）
- ✅ 下单与支付（支付宝/微信扫码）
- ✅ 订单状态流转（待支付→已支付→制作中→已完成→已取消）
- ✅ 商家后台订单管理（接单、修改状态、查看文件）
- ✅ 审计日志（记录关键操作：订单创建/支付/状态变更）

#### 本期 **不做**（后续迭代）
- ❌ 第三方登录（微信/支付宝 OAuth）
- ❌ 会员等级与积分系统
- ❌ 优惠券/促销活动
- ❌ 物流对接与实时配送跟踪
- ❌ 复杂报表（BI 数据看板）
- ❌ 多语言/多币种
- ❌ 消息队列/异步任务（MVP 同步处理）

---

## 3. 技术选型与原则（含取舍说明）

### 架构原则
1. **演进式架构**：从单体出发，按需拆分（不提前预设微服务边界）
2. **Clean Architecture**：领域层（Domain）独立于框架，UI 层与基础设施层可替换
3. **领域优先（Domain-first）**：先定义业务实体与用例，再选技术实现
4. **先 CRUD 后复杂**：MVP 优先完成增删改查，避免早期引入 CQRS/Event Sourcing
5. **避免厂商锁定**：数据库/存储/支付/邮件 均通过适配层封装，今天用 A 服务明天可换 B

### 核心技术栈

| 层级       | 技术选型                     | 理由与取舍                                                                      |
|----------|--------------------------|----------------------------------------------------------------------------|
| **前端**   | Next.js 14 App Router    | SSR + SSG + API Routes 一体，减少前后端对接成本；App Router 新特性（Server Components）提升首屏性能  |
|          | TypeScript + Tailwind CSS| 类型安全 + 快速样式开发，减少 CSS 命名冲突                                                  |
| **后端**   | Next.js API Routes       | 与前端同仓库，减少部署复杂度；MVP 阶段流量不大无须独立后端                                             |
| **数据库**  | PostgreSQL 15+           | 关系型数据（订单/用户/产品），事务支持强；可选托管（Supabase/RDS/自建），迁移成本低                            |
| **ORM**  | Prisma 或 Drizzle ORM    | 类型安全、迁移管理方便；Drizzle 更轻量（取舍：Prisma 生态好，Drizzle 性能更优）                          |
| **认证**   | JWT（自实现）或 NextAuth.js | MVP 用 JWT 存储 userId + role，无需额外 session 表；后续可换 OAuth（NextAuth.js 适配层预留）     |
| **对象存储** | S3 兼容接口（MinIO/R2/OSS）  | 避免绑定 Supabase Storage 专属 API，保持迁移能力；Cloudflare R2 无出口流量费                     |
| **支付**   | 适配层 + 支付宝/微信             | 封装 PaymentGateway 接口，provider 字段可切换；MVP 先对接国内主流，后续加 Stripe                    |
| **CDN**  | Cloudflare（或 Vercel）    | 静态资源加速 + DDoS 防护；Vercel 部署方便但成本较高（T1 后评估迁移）                                  |
| **监控**   | Sentry（错误）+ Uptime 监控   | 开源免费额度够用；生产日志用结构化 JSON（便于后续接 ELK/Loki）                                      |

### 运行形态
- **单体应用（Monolith-first）**：Next.js 统一服务前台 + 后台 + API
- **数据库**：单一 Postgres 实例（开发环境可用 Docker，生产可选托管）
- **无状态化**：JWT 认证（不依赖 session 表），可水平扩展（负载均衡）
- **静态资源**：上传到对象存储 → CDN 分发（减轻服务器压力）

---

## 4. 高阶架构图（文字化说明）

```
                         ┌─────────────────────────┐
                         │   用户浏览器 / 手机浏览器     │
                         └───────────┬─────────────┘
                                     │ HTTPS
                                     ↓
                         ┌─────────────────────────┐
                         │    CDN / Cloudflare     │ ← 静态资源（JS/CSS/Image）
                         └───────────┬─────────────┘
                                     │
                ┌────────────────────┴────────────────────┐
                │                                         │
                ↓                                         ↓
    ┌────────────────────────┐              ┌────────────────────────┐
    │   B2C 前台（Next.js）    │              │  管理后台（Next.js）      │
    │  /products, /orders    │              │  /admin/*, RBAC 拦截   │
    └───────────┬────────────┘              └───────────┬────────────┘
                │                                       │
                └───────────────┬───────────────────────┘
                                │
                                ↓
                    ┌────────────────────────┐
                    │   Next.js API Routes   │ ← 业务逻辑（Controller 层）
                    │  /api/auth, /api/orders│
                    │  /api/products, etc.   │
                    └───────────┬────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
                ↓               ↓               ↓
        ┌──────────┐    ┌─────────────┐   ┌──────────────┐
        │ Postgres │    │ 对象存储(S3)  │   │ 外部服务适配层  │
        │  (主库)   │    │  打印文件/头像 │   │ 支付/短信/邮件 │
        └──────────┘    └─────────────┘   └──────────────┘
```

**数据流说明**：
1. 用户通过 CDN 访问静态资源（首屏 HTML 由 Next.js SSR 生成）
2. 动态请求（API）直达 Next.js API Routes
3. API Routes 调用 Service 层（Clean Architecture 领域服务）
4. Service 层操作 Postgres（订单/用户数据）、对象存储（文件）、外部服务（支付）
5. 敏感操作写入 audit_logs 审计表

---

## 5. 模块与职责清单

### 5.1 B2C 前台模块（面向终端用户）

| 模块       | 路由              | 功能                               | 权限      |
|----------|-----------------|----------------------------------|---------|
| 首页       | `/`             | 轮播图、热门产品、活动入口                    | 公开      |
| 产品列表     | `/products`     | 展示所有打印服务（卡片式），筛选/搜索              | 公开      |
| 产品详情     | `/products/:id` | 显示选项（纸张/尺寸/数量），实时计价，上传文件入口       | 公开可查，下单需登录 |
| 购物车/下单   | `/checkout`     | 确认订单明细、填写收货信息、提交订单               | 需登录     |
| 支付页      | `/payment/:id`  | 展示订单金额、支付方式选择（支付宝/微信扫码）          | 需登录     |
| 我的订单     | `/orders`       | 查询当前用户所有订单（状态筛选、分页）              | 需登录     |
| 订单详情     | `/orders/:id`   | 显示订单明细、支付状态、打印文件下载、售后入口          | 需登录且本人  |
| 个人中心     | `/profile`      | 修改头像/昵称/密码、地址管理                  | 需登录     |

### 5.2 管理后台模块（面向商家/运营）

| 模块       | 路由                     | 功能                             | 权限       |
|----------|------------------------|--------------------------------|----------|
| 登录      | `/admin/login`         | 管理员账号登录（邮箱+密码，或 2FA 待后续）       | 公开       |
| 仪表盘     | `/admin/dashboard`     | 今日订单量/金额、待处理订单数、热门产品 Top 5     | 商家/管理员   |
| 订单管理    | `/admin/orders`        | 查看所有订单、筛选（状态/时间/用户）、修改状态、下载文件 | 商家/管理员   |
| 产品管理    | `/admin/products`      | 增删改查产品、配置选项（纸张/尺寸/价格规则）       | 管理员      |
| 用户管理    | `/admin/users`         | 查询用户列表、封禁/解封、角色分配              | 管理员      |
| 审计日志    | `/admin/audit-logs`    | 查询关键操作记录（用户/操作类型/时间范围）         | 管理员      |
| 系统配置    | `/admin/settings`      | 修改全局配置（如支付开关、上传限额、联系方式）       | 超级管理员    |

### 5.3 通用组件与中间件

| 组件          | 职责                                 | 说明                               |
|-------------|------------------------------------|------------------------------------|
| AuthMiddleware | 校验 JWT、提取 userId 与 role         | 拦截 API 请求，未登录返回 401            |
| RBACGuard     | 基于角色的访问控制（如 role >= 2 才可访问后台） | 在 API Routes 或 Server Component 中调用 |
| RateLimiter   | 限流（登录 5 次/分钟，下单 10 次/小时）        | 防暴力破解与 DDoS，基于 IP 或 userId      |
| AuditLogger   | 记录关键操作到 audit_logs             | 订单创建/支付/状态变更/用户角色修改            |
| ErrorHandler  | 统一错误格式、记录日志、返回 trace_id        | 所有 API 异常捕获后返回标准 JSON           |
| PaymentAdapter| 支付网关抽象接口（createOrder/verifyCallback） | 今天用支付宝，明天可换 Stripe 不改业务代码      |
| StorageAdapter| 对象存储抽象（presignedUrl/upload/delete）  | 避免绑定 Supabase Storage 专属 API     |

---

## 6. 数据库设计（MVP 表结构）

### 表清单与主要字段

#### 6.1 `users`（用户表）
```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(255) UNIQUE,          -- 可为空（如手机号注册）
  phone         VARCHAR(20) UNIQUE,           -- 可为空（如邮箱注册）
  hashed_password VARCHAR(255) NOT NULL,      -- bcrypt 加密
  role          INT DEFAULT 1,                -- 0=banned, 1=user, 2=merchant, 9=admin
  status        VARCHAR(20) DEFAULT 'active', -- active | suspended | deleted
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
CREATE INDEX idx_users_phone ON users(phone) WHERE phone IS NOT NULL;
```
**说明**：
- `email` 或 `phone` 至少一个非空（应用层校验）
- `role`：1=普通用户，2=商家操作员，9=平台管理员
- 密码用 bcrypt 存储（salt round 12）

#### 6.2 `profiles`（用户扩展信息）
```sql
CREATE TABLE profiles (
  user_id       UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  name          VARCHAR(100),
  avatar_url    TEXT,
  locale        VARCHAR(10) DEFAULT 'zh-CN',
  timezone      VARCHAR(50) DEFAULT 'Asia/Shanghai',
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);
```
**说明**：与 `users` 1:1 关系，分离核心认证与业务属性。

#### 6.3 `products`（产品表）
```sql
CREATE TABLE products (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          VARCHAR(200) NOT NULL,
  slug          VARCHAR(200) UNIQUE NOT NULL, -- URL 友好（如 "color-business-card"）
  description   TEXT,
  category      VARCHAR(50),                  -- 如 "名片" | "文档" | "海报"
  base_price    DECIMAL(10,2) NOT NULL,       -- 基础价格（单位：元）
  pricing_mode  VARCHAR(20) DEFAULT 'fixed',  -- fixed | formula（公式计价）
  status        VARCHAR(20) DEFAULT 'active', -- active | draft | archived
  sort_order    INT DEFAULT 0,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_status ON products(status);
```

#### 6.4 `product_options`（产品选项表）
```sql
CREATE TABLE product_options (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id    UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  name          VARCHAR(100) NOT NULL,        -- 如 "纸张类型"
  type          VARCHAR(20) NOT NULL,         -- select | radio | number
  values        JSONB NOT NULL,               -- 如 {"options": ["铜版纸", "哑粉纸"], "default": "铜版纸"}
  price_impact  JSONB,                        -- 如 {"铜版纸": 0, "哑粉纸": 5} 加价规则
  required      BOOLEAN DEFAULT TRUE,
  sort_order    INT DEFAULT 0,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_product_options_product ON product_options(product_id);
```
**说明**：
- `values` 存储选项配置（前端渲染用）
- `price_impact` 存储价格影响（如某选项 +5 元）

#### 6.5 `orders`（订单表）
```sql
CREATE TABLE orders (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_no      VARCHAR(32) UNIQUE NOT NULL,  -- 业务订单号（yyyyMMddHHmmss + 随机6位）
  user_id       UUID NOT NULL REFERENCES users(id),
  status        VARCHAR(20) DEFAULT 'pending',-- pending | paid | processing | completed | cancelled | refunded
  total_amount  DECIMAL(10,2) NOT NULL,
  currency      VARCHAR(3) DEFAULT 'CNY',
  notes         TEXT,                         -- 用户备注
  delivery_mode VARCHAR(20) DEFAULT 'pickup', -- pickup | delivery
  delivery_info JSONB,                        -- 地址/联系方式
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);
```

#### 6.6 `order_items`（订单明细表）
```sql
CREATE TABLE order_items (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id    UUID NOT NULL REFERENCES products(id),
  product_name  VARCHAR(200) NOT NULL,        -- 冗余（防产品删除后订单丢失信息）
  quantity      INT NOT NULL CHECK (quantity > 0),
  unit_price    DECIMAL(10,2) NOT NULL,
  subtotal      DECIMAL(10,2) NOT NULL,       -- quantity * unit_price
  options_snapshot JSONB,                     -- 保存选项快照（如 {"纸张": "铜版纸", "数量": 100}）
  file_ids      JSONB,                        -- 关联打印文件 ID 数组
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
```

#### 6.7 `payments`（支付记录表）
```sql
CREATE TABLE payments (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      UUID NOT NULL REFERENCES orders(id),
  provider      VARCHAR(20) NOT NULL,         -- alipay | wechat | stripe
  status        VARCHAR(20) DEFAULT 'pending',-- pending | success | failed | refunded
  amount        DECIMAL(10,2) NOT NULL,
  currency      VARCHAR(3) DEFAULT 'CNY',
  txn_ref       VARCHAR(100),                 -- 外部支付流水号
  paid_at       TIMESTAMPTZ,
  refunded_at   TIMESTAMPTZ,
  meta          JSONB,                        -- 存回调原始数据
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_txn_ref ON payments(txn_ref);
```

#### 6.8 `files`（文件表）
```sql
CREATE TABLE files (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id      UUID NOT NULL REFERENCES users(id),
  bucket        VARCHAR(100) NOT NULL,        -- 如 "print-files"
  key           TEXT NOT NULL,                -- 对象存储路径（如 "uploads/2025/10/abc123.pdf"）
  filename      VARCHAR(255) NOT NULL,        -- 原始文件名
  mime_type     VARCHAR(100),
  size_bytes    BIGINT,
  status        VARCHAR(20) DEFAULT 'active', -- active | deleted
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_files_owner ON files(owner_id);
CREATE INDEX idx_files_status ON files(status);
```

#### 6.9 `audit_logs`（审计日志表）
```sql
CREATE TABLE audit_logs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_id      UUID REFERENCES users(id),    -- 谁操作（可为空=系统触发）
  action        VARCHAR(50) NOT NULL,         -- CREATE | UPDATE | DELETE | LOGIN | PAYMENT
  entity        VARCHAR(50) NOT NULL,         -- orders | users | products
  entity_id     UUID,
  changes       JSONB,                        -- 变更前后对比（如 {"status": {"from": "pending", "to": "paid"}}）
  ip_address    INET,
  user_agent    TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_id);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity, entity_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at DESC);
```

### 索引策略
- **高频查询字段**：user_id, order_id, status, created_at 均建索引
- **唯一约束**：email, phone, order_no 唯一索引（避免重复）
- **时间降序**：created_at DESC 索引（订单列表/审计日志分页）
- **部分索引**：`WHERE status = 'active'`（减少索引大小）

---

## 7. 权限与行级安全（RLS 思路）

MVP 阶段在 **应用层（API Routes）** 实现权限控制，为后续迁移到数据库 RLS（如 Supabase）或微服务网关留后路。

### 7.1 核心规则

| 资源              | 读取（Read）                  | 写入（Write）             |
|-----------------|---------------------------|------------------------|
| `users`         | 本人可读自己记录；管理员可读所有        | 本人可改自己 profile；管理员可改角色 |
| `profiles`      | 同上                        | 同上                     |
| `products`      | 所有登录用户可读（前台展示）          | 仅管理员可增删改               |
| `product_options` | 同上                        | 同上                     |
| `orders`        | 本人可读自己订单；商家/管理员可读所有     | 本人创建自己订单；商家可改状态        |
| `order_items`   | 同上（跟随 order）             | 创建时与 order 一起写入        |
| `payments`      | 本人可读自己支付记录；管理员可读所有      | 系统自动创建/更新（回调）          |
| `files`         | 本人可读自己上传文件；商家可读订单关联文件   | 本人可上传/删除自己文件           |
| `audit_logs`    | 仅管理员可读                    | 系统自动写入（应用层触发）          |

### 7.2 实现方式（伪代码）

```typescript
// middleware/rbac.ts
export function requireAuth(req: Request) {
  const token = req.headers.get('Authorization')?.replace('Bearer ', '');
  if (!token) throw new UnauthorizedError();
  const payload = verifyJWT(token); // 返回 { userId, role }
  return payload;
}

export function requireRole(minRole: number) {
  return (req: Request) => {
    const { role } = requireAuth(req);
    if (role < minRole) throw new ForbiddenError();
  };
}

// API 示例：GET /api/orders/:id
export async function GET(req: Request, { params }) {
  const { userId, role } = requireAuth(req);
  const order = await db.orders.findUnique({ where: { id: params.id } });
  
  // 行级安全：非本人且非商家/管理员，不可查看
  if (order.user_id !== userId && role < 2) {
    throw new ForbiddenError('无权查看此订单');
  }
  
  return Response.json(order);
}
```

### 7.3 审计埋点
在关键操作后调用 `AuditLogger.log()`：
- 用户登录成功 → `action='LOGIN'`
- 创建订单 → `action='CREATE', entity='orders'`
- 订单状态变更 → `action='UPDATE', changes={status: {from, to}}`
- 支付成功 → `action='PAYMENT', entity='orders'`

---

## 8. API 规范（REST/JSON，简洁示例）

### 8.1 通用约定
- **Base URL**：`https://api.yoursite.com` 或 `/api`（Next.js 同域）
- **Content-Type**：`application/json`
- **认证**：`Authorization: Bearer <JWT>`
- **分页**：`?page=1&limit=20`（默认 limit=20，最大 100）
- **排序**：`?sort=created_at&order=desc`

### 8.2 错误格式
```json
{
  "error_code": "UNAUTHORIZED",
  "message": "Invalid or expired token",
  "trace_id": "abc123xyz",
  "timestamp": "2025-10-26T12:34:56Z"
}
```

**HTTP 状态码**：
- 200 OK：成功
- 201 Created：资源创建成功
- 400 Bad Request：参数错误
- 401 Unauthorized：未登录
- 403 Forbidden：无权限
- 404 Not Found：资源不存在
- 429 Too Many Requests：限流
- 500 Internal Server Error：服务器错误

### 8.3 核心接口

#### 认证模块
```
POST /api/auth/register
Body: { email?, phone?, password, code }
Response: { user: {...}, token: "..." }

POST /api/auth/login
Body: { email/phone, password }
Response: { token: "...", expires_in: 86400 }

POST /api/auth/logout
Headers: Authorization
Response: 204 No Content

POST /api/auth/refresh
Body: { refresh_token }
Response: { token: "...", expires_in: 86400 }
```

#### 产品模块
```
GET /api/products?category=名片&status=active
Response: { data: [...], pagination: {...} }

GET /api/products/:id
Response: { id, name, description, options: [...], base_price }
```

#### 订单模块
```
POST /api/orders
Body: {
  items: [{ product_id, quantity, options: {...}, file_ids: [...] }],
  delivery_mode: "pickup",
  delivery_info: {...}
}
Response: { order_id, order_no, total_amount, pay_url }

GET /api/orders?status=paid
Response: { data: [...], pagination: {...} }

GET /api/orders/:id
Response: { order_no, status, total_amount, items: [...], payments: [...] }

PATCH /api/orders/:id/status
Body: { status: "processing" }
Response: { order_id, status, updated_at }
```

#### 支付模块
```
POST /api/payments/intent
Body: { order_id, provider: "alipay" }
Response: { payment_id, qr_code_url, expires_at }

POST /api/payments/webhook (支付回调，无需鉴权但需验签)
Body: { ... } (由支付网关发送)
Response: 200 OK (告知支付网关收到)
```

#### 文件模块
```
POST /api/files/presigned-url
Body: { filename, mime_type, size_bytes }
Response: { upload_url, file_id }  (客户端直传到对象存储)

POST /api/files (元数据创建)
Body: { file_id, bucket, key, ... }
Response: { file_id, url }

DELETE /api/files/:id
Response: 204 No Content
```

### 8.4 速率限制

| 接口                  | 限制                  | 说明          |
|---------------------|---------------------|-------------|
| `/api/auth/login`   | 5 次/分钟/IP           | 防暴力破解       |
| `/api/auth/register`| 3 次/小时/IP           | 防批量注册       |
| `/api/orders`（POST）| 10 次/小时/user        | 防刷单         |
| `/api/files/*`      | 100 次/小时/user       | 防滥用上传       |
| 其他 GET 接口          | 300 次/分钟/user       | 正常使用不触发     |

---

## 9. 部署与环境（云中立）

### 9.1 环境划分

| 环境      | 用途            | 数据库            | 对象存储         | 域名示例                   |
|---------|---------------|----------------|--------------|------------------------|
| **dev** | 本地开发          | Docker Postgres| MinIO（本地）    | localhost:3000         |
| **staging** | 测试与演示     | Supabase 免费版  | Cloudflare R2| staging.yoursite.com   |
| **prod**| 生产            | Supabase Pro/RDS| Cloudflare R2| www.yoursite.com       |

### 9.2 部署方案（云中立）

#### 方案 A：Vercel + Supabase + Cloudflare R2（推荐 MVP）
- **优点**：零运维、部署快（git push 自动构建）、免费额度够用
- **缺点**：Vercel 免费版有 Serverless 函数时长限制（10s），大文件处理需异步
- **成本**：约 $0–25/月（取决于流量）

#### 方案 B：自建 VPS（如 Hetzner/DigitalOcean）+ Docker
- **优点**：成本可控（固定 $10–20/月）、无厂商限制
- **缺点**：需自己维护 Nginx/SSL/监控
- **适用**：T2 后流量增长，从 Vercel 迁移以节省成本

#### 容器化（可选）
```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### 9.3 环境变量管理
```bash
# .env.local (不提交到 git)
DATABASE_URL=postgresql://user:pass@host:5432/dbname
JWT_SECRET=your-secret-key-here
NEXT_PUBLIC_API_URL=https://api.yoursite.com

# 对象存储（S3 兼容）
S3_ENDPOINT=https://xxx.r2.cloudflarestorage.com
S3_ACCESS_KEY_ID=...
S3_SECRET_ACCESS_KEY=...
S3_BUCKET=print-files

# 支付
ALIPAY_APP_ID=...
ALIPAY_PRIVATE_KEY=...
WECHAT_MCHID=...
WECHAT_API_KEY=...

# 监控
SENTRY_DSN=https://...
```

### 9.4 基础监控
- **健康检查**：`/api/health`（返回数据库连接状态、存储可用性）
- **错误告警**：Sentry 捕获 5xx 错误，邮件/企微通知
- **性能监控**：Vercel Analytics 或自建（记录 API 耗时 > 3s 告警）
- **日志**：结构化 JSON 输出到 stdout（生产可接 Loki/Datadog）

---

## 10. 性能与可用性

### 10.1 性能目标

| 指标         | MVP 目标   | 优化方向（T2+）                  |
|------------|----------|---------------------------|
| **TTFB**   | < 800ms  | SSR 优化、数据库查询缓存            |
| **FCP**    | < 1.5s   | 代码分割、关键 CSS 内联            |
| **LCP**    | < 2.5s   | 图片懒加载、CDN 加速               |
| **首页加载**   | < 3s     | 预渲染（SSG）、Service Worker    |
| **API 响应** | < 500ms  | 慢查询优化、Redis 缓存（T2 引入）     |

### 10.2 数据库优化

#### 索引策略（已在表结构体现）
- **高频查询**：user_id, order_id, status, created_at
- **联合索引**：`(user_id, status, created_at DESC)`（用户订单列表）
- **避免全表扫描**：WHERE 条件必有索引

#### 常见查询示例
```sql
-- 用户订单列表（带分页）
SELECT * FROM orders 
WHERE user_id = $1 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 0;

-- 商家待处理订单（状态=paid）
SELECT o.*, u.name FROM orders o
JOIN profiles u ON o.user_id = u.user_id
WHERE o.status = 'paid'
ORDER BY o.created_at ASC;

-- 审计日志查询（按时间范围）
SELECT * FROM audit_logs
WHERE created_at BETWEEN $1 AND $2
  AND entity = $3
ORDER BY created_at DESC;
```

### 10.3 缓存策略（T2 引入 Redis）
- **产品列表**：缓存 5 分钟（产品不常改）
- **用户 Profile**：缓存 10 分钟（修改后 invalidate）
- **订单详情**：不缓存（实时性要求高）

### 10.4 异步处理（T3 按需引入）
MVP 阶段同步处理，若出现瓶颈再引入消息队列：
- **邮件/短信**：发送后台任务（防阻塞 API 响应）
- **大文件处理**：上传后异步生成缩略图/预览

---

## 11. 安全清单（MVP 必做）

### 11.1 认证与授权
- [x] 密码 bcrypt 加密（salt round ≥ 12）
- [x] JWT 过期时间 24 小时，Refresh Token 7 天
- [x] 每个 API 路由都有 Auth 中间件拦截
- [x] RBAC 校验（role >= minRole）
- [x] 敏感接口（修改角色/删除用户）需二次验证或审计日志

### 11.2 输入校验与防注入
- [x] 所有用户输入用 Zod/Joi 校验（类型+长度+格式）
- [x] Prisma/Drizzle 参数化查询（防 SQL 注入）
- [x] 富文本（如产品描述）用 DOMPurify 过滤 XSS
- [x] 文件上传白名单（仅 PDF/JPG/PNG，< 50MB）
- [x] 文件类型二次校验（magic number，防伪造扩展名）

### 11.3 速率限制与防攻击
- [x] 登录接口 5 次/分钟（基于 IP）
- [x] 注册接口需邮箱/手机验证码（防批量注册）
- [x] 下单接口 10 次/小时/用户（防刷单）
- [x] 全局 API 限流（300 次/分钟/用户）

### 11.4 敏感信息保护
- [x] 数据库密码、JWT Secret、支付密钥 均用环境变量（不提交 git）
- [x] 返回 API 时过滤 `hashed_password` 等敏感字段
- [x] 日志脱敏（不记录密码/支付卡号）
- [x] HTTPS 强制（生产环境 Cloudflare 自动开启）

### 11.5 审计与备份
- [x] 关键操作写入 `audit_logs`（登录/支付/状态变更）
- [x] 数据库每日自动备份（Supabase 自带/自建用 pg_dump cron）
- [x] 备份恢复演练（每月一次，确保可还原）

### 11.6 测试用例（部分示例）
| 用例                 | 预期结果                |
|--------------------|---------------------|
| 未登录访问 `/api/orders` | 返回 401            |
| 用户 A 访问用户 B 的订单     | 返回 403            |
| SQL 注入尝试（`' OR 1=1`）| 参数化查询防御，返回 400    |
| 上传 100MB 文件        | 前端/后端双重拦截，返回 413 |
| 密码错误 6 次连续登录       | 触发限流，返回 429       |

---

## 12. MVP 范围与里程碑

### T0（第 1–2 周）：项目骨架与基础设施
**目标**：可运行的"Hello World"、数据库连通、认证能跑

**交付物**：
- [x] Next.js 项目初始化（App Router + TypeScript + Tailwind）
- [x] Prisma/Drizzle ORM 配置 + 数据库迁移脚本（users/profiles/products 表）
- [x] JWT 认证实现（注册/登录/登出）
- [x] 中间件：`requireAuth` + `requireRole`
- [x] Sentry 错误监控集成
- [x] GitHub Actions CI（代码格式检查 + 测试）

**验收标准（DoD）**：
- ✅ `npm run dev` 可启动本地环境
- ✅ 可用 Postman 调用 `/api/auth/register` 创建用户
- ✅ 登录后拿到 JWT，访问 `/api/profile` 返回用户信息
- ✅ 数据库迁移可重复执行（up/down）

---

### T1（第 3–6 周）：核心业务流程（Happy Path）
**目标**：用户能完成"浏览产品→上传文件→下单→支付"全流程

**交付物**：
- [x] 产品模块（CRUD + 选项配置）
- [x] 前台产品列表/详情页（SSR）
- [x] 文件上传（对象存储预签名 URL + 元数据入库）
- [x] 下单流程（创建订单 + 订单明细 + 实时计价）
- [x] 支付对接（支付宝/微信扫码，生成 qr_code_url）
- [x] 支付回调处理（验签 + 更新订单状态 + 审计日志）
- [x] 用户查看订单列表/详情

**验收标准（DoD）**：
- ✅ 产品页可显示"彩色名片"，选择选项后价格动态更新
- ✅ 上传 PDF 文件（< 50MB），返回 file_id
- ✅ 点击"提交订单"，跳转支付二维码
- ✅ 扫码支付后（用沙箱环境测试），订单状态变为 `paid`
- ✅ 审计日志记录订单创建/支付事件

---

### T2（第 7–10 周）：管理后台与运营支持
**目标**：商家可管理订单、产品，管理员可查审计日志

**交付物**：
- [x] 管理后台登录页（独立路由 `/admin/login`，RBAC 拦截）
- [x] 仪表盘（今日订单量/金额、待处理订单数）
- [x] 订单管理（列表/详情/修改状态/下载文件）
- [x] 产品管理（增删改查产品/选项）
- [x] 用户管理（列表/封禁/角色分配）
- [x] 审计日志查询（筛选 + 分页）
- [x] 基础报表（订单趋势图/热门产品 Top 5）
- [x] 健康检查 `/api/health`

**验收标准（DoD）**：
- ✅ 商家账号登录后，可查看所有订单
- ✅ 点击订单，可修改状态为"制作中"→"已完成"
- ✅ 管理员可新增产品、配置选项（纸张/尺寸/价格）
- ✅ 审计日志可查询"今天谁修改了订单 XYZ 的状态"
- ✅ `/api/health` 返回数据库/存储连通状态

---

### T3（第 10–12 周）：灰度测试与上线准备
**目标**：修复 Bug、性能优化、正式上线

**交付物**：
- [x] 单元测试覆盖率 > 60%（关键业务逻辑）
- [x] E2E 测试（Playwright）：注册→登录→下单→支付
- [x] 性能优化（图片懒加载、代码分割、CDN 配置）
- [x] SEO 优化（meta 标签、sitemap.xml）
- [x] 灰度测试（邀请 10 位真实用户试用，收集反馈）
- [x] 生产环境部署（配置 HTTPS/CDN/监控告警）
- [x] 运营文档（管理后台使用手册、常见问题 FAQ）

**验收标准（DoD）**：
- ✅ Lighthouse 评分：Performance > 80，Accessibility > 90
- ✅ E2E 测试通过率 100%
- ✅ Sentry 错误率 < 1%（7 天内无 P0 故障）
- ✅ 备份恢复演练成功（可在 15 分钟内还原数据库）
- ✅ 正式域名 SSL 证书有效、CDN 缓存命中率 > 80%

---

## 13. 风险与演进路线

### 13.1 近期风险（MVP 阶段）

| 风险项          | 影响       | 缓解措施                                   | Owner      |
|--------------|----------|----------------------------------------|------------|
| 支付合规问题       | 高        | 使用官方 SDK，沙箱环境充分测试；阅读支付宝/微信文档合规条款       | 后端工程师      |
| 对象存储费用超支     | 中        | Cloudflare R2 无出口流量费；设置用量告警（> $10/月）       | 全栈工程师      |
| 短信发送率受限      | 中        | MVP 阶段用邮箱注册优先；短信限流（1 次/分钟/手机号）           | 后端工程师      |
| 大文件上传超时      | 中        | 前端分片上传（< 10MB/片）；后端异步合并                  | 前端+后端      |
| 数据库连接数耗尽     | 低（MVP流量小）| Prisma 连接池限制 10；监控连接数告警                   | 后端工程师      |
| 未做 GDPR 合规   | 低（国内为主） | T2 后添加"数据导出/删除"功能；隐私政策页面                 | 产品经理+法务    |

### 13.2 技术债清单

| 技术债              | 影响       | 到期时间   | 还债方案                             |
|------------------|----------|--------|----------------------------------|
| 审计日志未分表          | 低（100w 行内）| T4（6个月）| 按月分表（audit_logs_202510）         |
| 支付回调无幂等校验        | 中        | T2 末    | 增加 `txn_ref` 唯一索引，防重复处理         |
| 文件上传无病毒扫描        | 中        | T3     | 集成 ClamAV 或云厂商扫描 API             |
| 前端无 i18n 多语言支持   | 低（MVP 仅中文）| T5（1年） | 引入 next-i18next，提取文案到 JSON       |
| 单体应用日志混杂         | 低        | T4     | 结构化日志（Winston）+ 分类（auth/order）  |
| 无 Redis 缓存       | 中（流量增长后） | T3 末    | 部署 Redis，缓存产品列表/用户 Profile      |

### 13.3 演进路线（12 个月规划）

#### 短期（0–3 个月）：MVP 上线与稳定
- ✅ 完成 T0→T1→T2→T3 里程碑
- ✅ 灰度测试 → 正式上线
- ✅ 监控告警机制完善（Sentry + Uptime）

#### 中期（3–6 个月）：功能扩展与性能优化
- 🔄 会员等级/积分系统（鼓励复购）
- 🔄 优惠券/促销活动模块
- 🔄 物流对接（快递 100 或顺丰 API）
- 🔄 引入 Redis 缓存（产品列表/用户 Profile）
- 🔄 订单量 > 1000/天时考虑**读写分离**（主从 Postgres）

#### 长期（6–12 个月）：架构演进与规模化
- 🚀 **垂直拆分**：订单服务独立（独立数据库 + API）
- 🚀 引入消息队列（RabbitMQ/SQS）：异步处理邮件/短信/报表
- 🚀 CQRS 模式（仅报表查询用只读副本，主库专注写入）
- 🚀 多地域部署（CDN 边缘节点 + 对象存储跨区域复制）
- 🚀 去锁定迁移演练（从 Supabase 迁移到自建 Postgres/RDS）

### 13.4 去锁定策略（Cloud-agnostic）

| 依赖服务    | 当前选择                | 抽象层设计                   | 迁移难度 |
|---------|---------------------|-------------------------|------|
| 数据库     | Supabase Postgres   | Prisma ORM（标准 SQL）      | 低    |
| 对象存储    | Cloudflare R2       | S3 兼容接口（aws-sdk）        | 低    |
| 认证      | 自实现 JWT            | 可换 NextAuth.js/Supabase Auth | 中    |
| 支付      | 支付宝/微信              | PaymentAdapter 接口封装     | 低    |
| 邮件/短信   | 待定（可用阿里云/SendGrid）| EmailService 接口         | 低    |
| 托管平台    | Vercel              | 可迁移到 Docker + Nginx     | 中    |

**迁移手册（占位）**：
- 数据库导出：`pg_dump` → 导入新 Postgres
- 对象存储迁移：`aws s3 sync` 或 `rclone`
- 环境变量替换：`DATABASE_URL` / `S3_ENDPOINT` 等

---

## 14. 导师式讲解（Why）

### 决策 1：为什么单体优先，不一开始就微服务？
**原因**：  
微服务的收益在**团队规模 > 10 人**、**服务独立演进需求**时才显现。MVP 阶段只有 3 人，强行拆服务会带来：
- **沟通成本**：服务边界不清晰时，频繁跨服务调用导致延迟
- **调试地狱**：分布式链路追踪（Tracing）成本高，本地调试需启动 N 个容器
- **过度设计**：业务未验证就引入 Kafka/K8s，3 个月后可能砍掉功能

**取舍**：  
单体 MVP 可快速迭代，等订单量 > 5000/天、团队扩到 6 人时，再垂直拆分"订单服务"（独立数据库 + API Gateway）。

---

### 决策 2：为什么选 Postgres 而不是 MongoDB/Firebase？
**原因**：  
打印订单业务是**强事务场景**（订单-明细-支付 三表一致性），关系型数据库的 ACID 保证更可靠。MongoDB 的灵活 schema 在"产品选项"（JSONB）上有优势，但 Postgres 的 JSONB 字段已能满足，且：
- **查询能力**：SQL 生态成熟（JOIN/聚合/索引），复杂报表方便
- **迁移成本低**：Prisma ORM 标准 SQL，今天用 Supabase，明天换 RDS 改一行配置
- **成本**：MongoDB Atlas 免费版 512MB，Postgres 可自建 Docker 或用 Supabase 免费版

**取舍**：  
若未来需要"实时协作"（如在线设计编辑器），可引入 Firebase Realtime Database 作为**辅助存储**，主数据仍在 Postgres。

---

### 决策 3：为什么用 JWT 而不是 Session？
**原因**：  
- **无状态**：JWT 自包含 userId + role，无需查 session 表（减少数据库查询）
- **水平扩展**：多台服务器共享 JWT Secret 即可，无需 Redis 存 session（MVP 阶段省去 Redis 部署）
- **跨域友好**：前后端分离（或未来 App）可直接传 Token

**取舍**：  
JWT 无法主动"踢人"（除非维护黑名单），若需强制登出（如修改密码后所有设备下线），T2 后可引入 Redis 存"有效 Token 白名单"或短期 Refresh Token 机制。

---

### 决策 4：为什么对象存储用 S3 兼容接口，而不是直接绑定 Supabase Storage？
**原因**：  
Supabase Storage 专属 API（如 `supabase.storage.from('bucket').upload()`）若深度使用，未来迁移需改大量代码。而 S3 兼容接口（aws-sdk）是**事实标准**：
- **可迁移**：Cloudflare R2、MinIO、阿里云 OSS、AWS S3 均支持
- **成本灵活**：R2 无出口流量费（适合打印文件频繁下载），可随时换厂商

**取舍**：  
封装 `StorageAdapter` 接口，今天用 R2，明天换 AWS S3 只改配置，业务代码不动。

---

### 决策 5：为什么支付网关要抽象适配层？
**原因**：  
支付宝/微信的 API 差异大（签名算法/回调格式不同），若直接写 `if (provider === 'alipay')` 散落各处，后续加 Stripe/PayPal 会牵一发动全身。

**取舍**：  
定义 `PaymentGateway` 接口：
```typescript
interface PaymentGateway {
  createOrder(amount, currency, callbackUrl): Promise<{ qr_code_url }>;
  verifyCallback(req): Promise<{ txn_ref, status }>;
}
```
每个支付方式写一个实现类（AlipayGateway/WechatGateway），业务层只调用接口。

---

### 决策 6：为什么 MVP 阶段在应用层做权限，而不是数据库 RLS？
**原因**：  
Postgres RLS（Row-Level Security）或 Supabase RLS 功能强大，但**调试成本高**（权限规则写在 SQL，前端报错难定位）。MVP 阶段在 API Routes 做 RBAC：
- **可读性强**：`if (order.user_id !== userId && role < 2) throw Forbidden`
- **灵活**：临时加权限逻辑无需改数据库迁移

**取舍**：  
T3 后若前端直连数据库（如用 Supabase Realtime），再迁移权限规则到 RLS；但保持应用层有**二次校验**（Defense in Depth）。

---

### 决策 7：为什么用 Clean Architecture 分层？
**原因**：  
MVP 虽小，但业务逻辑（如"订单计价规则"）应独立于框架。分层结构：
```
/src
  /domain        ← 核心业务逻辑（纯 TS，不依赖 Next.js/Prisma）
  /application   ← 用例（Use Cases，调用 domain + repositories）
  /infrastructure← 技术实现（Prisma/S3/支付 SDK）
  /presentation  ← UI 层（Next.js 组件/API Routes）
```

**取舍**：  
未来若要从 Next.js 迁移到 NestJS（或拆微服务），domain + application 层可直接复用，只改 infrastructure 层。

---

### 决策 8：为什么不一开始就引入 Redis/消息队列？
**原因**：  
**YAGNI 原则**（You Aren't Gonna Need It）—— MVP 阶段日订单 < 100，同步处理即可：
- 邮件发送：调三方 API（< 200ms），阻塞 API 响应可接受
- 报表统计：管理员手动查询，无需实时

**取舍**：  
等日订单 > 1000、邮件发送失败影响用户体验时，再引入消息队（如 BullMQ + Redis）异步处理。

---

## 总结

这份架构设计以**演进式思维**为核心：
1. **先简后繁**：单体 → 垂直拆分 → 微服务（按需）
2. **领域优先**：业务实体/用例先于技术选型
3. **云中立**：数据库/存储/支付 均可迁移，避免绑定单一厂商
4. **MVP 聚焦**：8–12 周上线最小可行产品，砍掉非核心功能
5. **技术债可控**：列清单、定到期时间、按优先级还债

**给团队的建议**：
- **前端工程师**：专注 B2C 前台体验（产品页/下单流程/订单列表），用 Tailwind 快速出 UI
- **后端工程师**：专注 API Routes + 数据库设计，确保支付回调幂等性
- **全栈工程师**：兼顾管理后台 CRUD、对象存储对接、监控告警配置

**上线前最后一问**：  
我们是否能在 15 分钟内从备份恢复数据库？如果不能，先做备份演练。  
我们是否有 Sentry 告警？如果没有，先接入错误监控。  
我们是否测试过支付回调重复发送？如果没有，先加幂等校验。

---

**本文档为"可落地"架构设计**，而非纸上谈兵。请在实施过程中根据实际反馈调整，保持敏捷迭代。

*撰写：AI 架构师（Mentor 模式）*  
*适用场景：B2C SaaS、中小型 Web 服务、预算有限的创业团队*

