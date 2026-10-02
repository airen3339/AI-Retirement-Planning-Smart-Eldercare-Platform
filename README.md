# AI退休规划与智慧养老平台 · 功能说明文档

**文档版本：** V1.0.0  
**适用版本：** 平台 V1.0（2026年10月正式发布）  
**文档性质：** 产品功能说明（供产品经理、开发团队、测试团队参考）

---

## 目录

1. [产品概述](#1-产品概述)
2. [用户中心模块](#2-用户中心模块)
3. [退休规划计算模块](#3-退休规划计算模块)
4. [方案管理模块](#4-方案管理模块)
5. [AI养老顾问模块](#5-ai养老顾问模块)
6. [报告生成模块](#6-报告生成模块)
7. [会员与支付模块](#7-会员与支付模块)
8. [政策数据模块](#8-政策数据模块)
9. [智慧养老服务平台（Phase3）](#9-智慧养老服务平台phase3)
10. [权限与安全机制](#10-权限与安全机制)
11. [接口总览](#11-接口总览)
12. [数据字典](#12-数据字典)

---

## 1. 产品概述

### 1.1 产品定位

**AI退休规划与智慧养老平台**是面向中国在职人员及退休预备人群的一站式智能养老规划服务平台。平台以国家最新延迟退休政策为基础，结合用户个人社保缴费数据，通过AI大模型提供个性化的退休规划建议，帮助用户提前规划养老资产、了解养老缺口、做好财务准备。

### 1.2 核心价值

| 价值点 | 描述 |
|--------|------|
| 精准计算 | 基于2025年延迟退休新政，三段式养老金精准测算，误差<5% |
| AI赋能 | 接入通义千问大模型，提供个性化养老规划对话建议 |
| 全国覆盖 | 内置全国 401 条地区养老政策数据（31 个省级行政区全覆盖 + 370 个城市级行政区），按城市精准计算 |
| 缺口可视化 | 直观展示退休后收入缺口，给出补充储蓄建议 |
| 专业报告 | 生成PDF版退休规划报告，可存档、可分享 |

### 1.3 目标用户

- **主要用户**：30~55岁在职人员，关注退休后生活质量
- **次要用户**：HR从业者，需要为员工批量规划退休方案
- **潜在用户**：金融理财规划师，需要工具辅助客户退休规划

### 1.4 技术架构简述

```
前端：Vue 3 + Vite + Element Plus + Echarts
后端：Spring Boot 2.7.18 + MyBatis Plus + Spring Security
缓存：Redis 6.x（政策缓存 / 会话 / AI对话上下文）
数据库：MySQL 8.0（8张核心业务表）
AI：通义千问API（Phase2接入，Phase1为规则引擎占位）
```

---

## 2. 用户中心模块

### 2.1 功能列表

| 功能 | 接口 | 权限 | 说明 |
|------|------|------|------|
| 手机号注册 | `POST /api/user/register` | 公开 | 短信验证码+密码注册 |
| 密码登录 | `POST /api/user/login` | 公开 | 手机号+密码 |
| 短信验证码登录 | `POST /api/user/login` | 公开 | type=sms |
| 微信小程序登录 | `POST /api/user/wx-login` | 公开 | code换取OpenID |
| 发送短信验证码 | `POST /api/user/sms/send` | 公开 | 60秒防重复，300秒有效期 |
| 获取个人信息 | `GET /api/user/profile` | 登录 | 返回脱敏数据 |
| 更新个人信息 | `PUT /api/user/profile` | 登录 | 不允许修改手机号/密码 |

### 2.2 注册流程

```
1. 用户输入手机号 → 2. 发送短信验证码（60秒锁定）
→ 3. 输入验证码+设置密码 → 4. 服务端验证码比对（Redis TTL 300s）
→ 5. BCrypt加密密码 → 6. 写入user_info表 → 7. 返回JWT Token
```

### 2.3 登录态管理

- **JWT有效期**：24小时（`jwt.expiration=86400000ms`）
- **刷新Token有效期**：7天
- **Token传递**：请求头 `Authorization: Bearer <token>`
- **踢下线机制**：管理员可置用户status=0，AuthInterceptor校验时拒绝

### 2.4 个人信息字段说明

| 字段 | 必填 | 影响 |
|------|------|------|
| 出生日期（birthDate） | 是 | 计算退休年龄、距退休月数 |
| 性别（sex） | 是 | 1=男，2=女干部/管理，3=女工人（影响法定退休年龄） |
| 月工资（monthlySalary） | 是 | 计算缴费指数 |
| 参工年份（workYear） | 否 | 辅助估算缴费年限 |
| 常住地区（areaCode） | 否 | 匹配地区养老政策 |

### 2.5 AI对话次数限制

| 会员类型 | 每日AI咨询次数 |
|----------|----------------|
| 免费用户（memberType=0） | 5次 |
| 普通会员（memberType=1） | 20次 |
| 高级会员（memberType=2） | 无限制 |

> 次数在每日凌晨由 `ai_chat_reset_date` 字段判断是否重置。

---

## 3. 退休规划计算模块

### 3.1 核心算法说明

平台实现国家法定三段式养老金计算公式（适用于2005年后参保人员）：

#### 第一段：基础养老金

$$
\text{基础养老金} = \frac{\text{当地上年度在岗职工月平均工资} \times (1 + \text{本人缴费工资指数})}{2} \times \text{缴费年限} \times 1\%
$$

**限制条件**：缴费年限 < 15年时，基础养老金为 0，仅可个人账户一次性结清。

#### 第二段：个人账户养老金

$$
\text{个人账户养老金} = \frac{\text{个人账户累计储存额}}{\text{计发月数}}
$$

**计发月数对照表（精选）：**

| 退休年龄 | 计发月数 |
|----------|----------|
| 50岁 | 195 |
| 55岁 | 170 |
| 60岁 | 139 |
| 63岁 | 约120（延迟退休最高） |

#### 第三段：过渡性养老金（1997年前工龄）

$$
\text{过渡性养老金} = \text{当地平均工资} \times \text{缴费工资指数} \times \text{视同缴费年限} \times \text{过渡系数}
$$

> 过渡系数默认1.2%，各省有差异（已内置各城市政策数据）。

#### 综合替代率

$$
\text{替代率} = \frac{\text{月养老金总额}}{\text{退休前月工资}} \times 100\%
$$

替代率评级：
- 优秀：≥ 80%
- 良好：60%~79%
- 一般：40%~59%
- 偏低：< 40%

### 3.2 延迟退休政策算法（2025年新政）

从2025年1月1日起，渐进式延迟退休：每4个月延迟1个月，直至达到目标年龄。

| 人群 | 原法定退休年龄 | 最终目标年龄 |
|------|---------------|-------------|
| 男性 | 60岁 | 63岁 |
| 女性干部/管理 | 55岁 | 58岁 |
| 女性工人 | 50岁 | 55岁 |

算法实现（`PensionCalculateUtil.calculateRetireAge`）：

```
已退休人员（原退休日≤2025-01-01）→ 不受影响，按原年龄
未退休人员 → 距离改革实施月数 / 4 = 延迟月数（上限为目标年龄-原年龄）
实际退休年龄 = 原年龄 + 延迟月数 / 12.0（精确到月）
```

### 3.3 养老缺口分析

```
目标月收入 = 退休前月工资 × 80%（维持当前80%生活水平）
月收入缺口 = max(0, 目标月收入 - 月养老金总额)
建议额外准备 = 月缺口 × 12 × 预计退休年数（按活到85岁估算）
```

### 3.4 个人账户余额估算

当用户未提供个人账户余额时，系统自动估算：

```
缴费基数 = min(max(月工资, 当地平均工资×0.6), 当地平均工资×3.0)
月缴费额 = 缴费基数 × 8%（个人账户缴费率）
账户余额（复利）= 月缴费额 × 12 × [(1 + 3.5%)^缴费年限 - 1] / 3.5%
```

### 3.5 计算接口说明

**请求：** `POST /api/retirement/calculate`（支持匿名访问）

**请求体字段：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| birthDate | String | 是 | 出生日期，格式 YYYY-MM-DD |
| sex | Integer | 是 | 1=男 2=女干部 3=女工人 |
| areaCode | String | 是 | 地区编码（如 110000=北京） |
| payYears | Double | 是 | 已缴费年限（年） |
| monthlySalary | Double | 是 | 当前月工资（元） |
| localAvgSalary | Double | 否 | 当地平均工资，为空则从政策库取 |
| accountBalance | Double | 否 | 个人账户余额，为空则系统估算 |
| transitionYears | Double | 否 | 视同缴费年限（1997年前工龄） |
| savePlan | Boolean | 否 | 是否保存为规划方案（需登录） |
| planName | String | 否 | 方案名称 |

**返回字段：**

| 字段 | 说明 |
|------|------|
| retireAge | 实际退休年龄（含延迟，精确到月） |
| retireDate | 预计退休日期 |
| monthsToRetire | 距退休剩余月数 |
| yearsToRetire | 距退休剩余年数（计算属性） |
| basePension | 基础养老金（元/月） |
| accountPension | 个人账户养老金（元/月） |
| transitionPension | 过渡性养老金（元/月） |
| totalPension | 月养老金总额（元/月） |
| replaceRate | 替代率（小数，如0.65=65%） |
| replaceRateLevel | 替代率评级（优秀/良好/一般/偏低） |
| monthlyGap | 月收入缺口（元）|
| suggestedSaving | 建议额外准备资金（元） |
| estimatedAccountBalance | 估算个人账户余额（元） |

---

## 4. 方案管理模块

### 4.1 功能列表

| 功能 | 接口 | 权限 |
|------|------|------|
| 保存规划方案 | `POST /api/retirement/plan/save` | 登录 |
| 获取方案列表 | `GET /api/retirement/plans` | 登录 |
| 获取方案详情 | `GET /api/retirement/plan/{id}` | 登录（仅本人） |
| 删除方案 | `DELETE /api/retirement/plan/{id}` | 登录（仅本人） |
| 多方案对比 | `POST /api/retirement/compare` | 登录 |

### 4.2 方案限额规则

| 会员类型 | 最大方案数 |
|----------|-----------|
| 免费用户 | 3个 |
| 普通会员 | 20个 |
| 高级会员 | 无限制 |

超出限额时，接口返回 `ResultCode.MEMBER_QUOTA_EXCEEDED`。

### 4.3 方案对比说明

- 最多同时对比 **5个** 方案
- 对比维度：退休年龄、月养老金、替代率、月收入缺口、建议储蓄
- 前端以 Echarts 多系列柱状图可视化展示

---

## 5. AI养老顾问模块

### 5.1 功能概述

AI养老顾问基于用户退休规划数据，提供多轮自然语言问答服务。**V1.0 阶段为规则引擎驱动（关键词匹配+模板回复），Phase2接入通义千问大模型。**

### 5.2 功能列表

| 功能 | 接口 | 说明 |
|------|------|------|
| 创建对话 | `POST /api/ai/conversation` | 初始化会话，获取conversationId |
| 发送消息 | `POST /api/ai/chat` | 问答，消耗每日次数配额 |
| 获取对话历史 | `GET /api/ai/history/{conversationId}` | 最近50条 |
| 获取对话列表 | `GET /api/ai/conversations` | 最近20条会话 |
| 生成规划报告（AI版） | `POST /api/ai/report/{planId}` | 基于方案数据生成文字报告 |

### 5.3 V1.0 规则引擎说明

V1.0 内置以下关键词应答场景：

| 触发关键词 | 回复主题 |
|-----------|---------|
| 延迟退休 / 推迟退休 | 解释2025年渐进式延迟退休政策 |
| 养老金 / 退休金 | 介绍三段式计算方法 |
| 缺口 / 不够用 | 养老缺口分析与建议 |
| 个人账户 | 个人账户养老金说明 |
| 缴费年限 / 社保 | 最低15年缴费要求说明 |
| 灵活就业 | 灵活就业参保政策说明 |
| 商业保险 / 年金 | 商业养老保险推荐建议 |

### 5.4 对话上下文管理

- 对话历史存储于 Redis，Key：`ai:conversation:{conversationId}`，TTL 24小时
- 每次请求携带最近 10 轮对话作为上下文（防止token超限）
- 数据库同步写入 `ai_conversation`、`ai_message` 表（用于持久化和统计）

---

## 6. 报告生成模块

### 6.1 功能概述

用户支付费用后，系统基于已保存的退休规划方案，异步生成 PDF 格式的专业退休规划报告。

### 6.2 功能列表

| 功能 | 接口 | 权限 |
|------|------|------|
| 生成报告 | `POST /api/report/generate` | 登录（需已支付订单） |
| 查询报告状态 | `GET /api/report/{reportId}` | 登录（仅本人） |
| 下载PDF报告 | `GET /api/report/download/{reportId}` | 登录（仅本人） |

### 6.3 报告生成流程

```
① 验证订单已支付（payStatus=1）
② 查询对应规划方案数据
③ 创建报告记录（status=0，生成中）
④ 委托 ReportPdfGenerator 异步生成PDF（@Async，独立线程池）
⑤ PDF写入本地存储（开发环境）/ OSS（生产环境）
⑥ 更新报告记录（status=1，成功 / status=2，失败）
⑦ 更新方案记录（reportGenerated=1）
⑧ 用户轮询 /api/report/{reportId} 查看状态，成功后下载
```

### 6.4 报告内容结构

| 章节 | 内容 |
|------|------|
| 封面 | 报告标题、生成日期 |
| 一、核心结果 | 退休日期/年龄、月养老金各段明细、替代率、缺口 |
| 二、建议 | 5条个性化养老规划建议 |
| 免责声明 | 仅供参考，以社保局核定为准 |

### 6.5 报告定价

| 报告类型 | 价格 | 说明 |
|----------|------|------|
| 基础报告（reportType=1） | 49元 | 核心数据PDF，10页内 |
| 详细报告（reportType=2） | 99元 | 含深度分析和多维建议，20页内 |

---

## 7. 会员与支付模块

### 7.1 会员体系

| 级别 | memberType | 年费 | 权益 |
|------|-----------|------|------|
| 免费用户 | 0 | 免费 | 每日5次AI、最多3个方案、基础计算 |
| 普通会员 | 1 | 199元/年 | 每日20次AI、最多20个方案、基础报告优惠 |
| 高级会员 | 2 | 399元/年 | 无限AI、无限方案、专属顾问、报告免费 |

### 7.2 订单类型

| orderType | 说明 | 创建接口 |
|-----------|------|---------|
| 1 | PDF报告订单 | `POST /api/order/report` |
| 2 | 普通会员订单 | `POST /api/order/member` |
| 3 | 高级会员订单 | `POST /api/order/member` |

### 7.3 支付流程

```
前端创建订单 → 获取orderNo
→ 调起微信/支付宝支付
→ 支付完成后，支付平台回调 /api/order/callback/wechat 或 /alipay
→ 服务端验证回调签名（防伪造）
→ 幂等更新订单状态（payStatus=1）
→ 激活会员或解锁报告权限
```

### 7.4 超时订单自动取消

`OrderServiceImpl` 内有 `@Scheduled(fixedDelay=300000)` 定时任务，每5分钟扫描30分钟内未支付的订单并取消（`payStatus=3`）。

---

## 8. 政策数据模块

### 8.1 覆盖范围

数据库内置 **401 条地区养老政策数据**（涵盖 31 个省级行政区 + 370 个城市级行政区），涵盖：

| 地区 | 覆盖情况 |
|------|---------|
| 直辖市 | 北京、上海、天津、重庆（4个，省级 + 市级码各 1 条） |
| 省级行政区 | 31 个省级行政区（省/自治区/直辖市）全部覆盖 |
| 计划单列市 | 深圳、大连、青岛、宁波、厦门 |
| 地级行政区 | 全部地级市 / 地区 / 自治州 / 盟，共 **342 个**（GB/T 2260 编码），**无一遗漏** |

> **数据来源与执行：** 基础 33 条来自 `database/init.sql`；补充 59 条来自 `database/phase3_city_policy.sql`（幂等写入）；
> 再补 285 条地级市来自 `database/phase3_city_policy_ext.sql`（幂等写入）。**全部执行后共 401 条。**
> 未收录的区县，系统自动按省份编码回退取省级数据。
>
> **修正记录（2026-09-24）：** 原 `phase3_city_policy.sql` 将「泉州市」误标为 `350300`（莆田市代码），
> 造成泉州市(350500)漏收、莆田市数据错配；同时漏收中山市(442000)。已全部修正补齐，
> 全国 **337 个地级行政区（333 个地级市/地区/自治州/盟 + 4 个直辖市）实现零缺口**；
> 城市级 346 条 = 333 地级行政区 + 4 直辖市城区 + 9 省直辖县级市，在任意城市均可直查。

**配套生态数据（同口径覆盖）：**

| 数据表 | 规模 | 来源脚本 |
|--------|------|---------|
| `service_provider` 服务商 | **66 家**（海淀区已备案真实机构，含地址/星级/床位/电话） | `phase3_extend.sql`（骨架）+ **`phase7_provider_real.sql`** |
| `subsidy_policy` 补贴政策 | **19 条**（4 直辖市 + 广东 7 市，全部带政府文号） | `phase3_extend.sql`（骨架）+ **`phase6_subsidy_real.sql`** |
| `knowledge_doc` RAG 语料 | **127 篇**（102 篇真实法规 + 25 篇已标注平台原创） | `phase2_knowledge.sql` / `_ext` / `_city` / `_advanced` + **`phase8_rag_label.sql`** |
| `point_goods` 积分商品 | **24 个**（门槛 100~5000 分） | `phase3_point.sql` + `phase3_point_goods_ext.sql` |

> ⚠️ **数据真实性整改（2026-09-25）—— 覆盖度让位于真实性。**
>
> 本项目早期为「覆盖度对齐」批量生成过大量**看似合理但实为编造**的数据：
> 服务商名称为「行政区名 + 服务类型」套模板、电话为随机数；补贴金额为「省级基准 × 系数」推算。
> 这些数据能过计数校验、能撑演示，但**任何一个都经不起查证**。
>
> 现已全部整改：**凡无 `source_url` 的记录一律删除，不以估算值填充**。整改结果：
>
> | 数据集 | 整改前 | 整改后 |
> |--------|--------|--------|
> | `subsidy_policy` | 1604 条（推算值） | **19 条**（带政府文号，可逐条核查） |
> | `service_provider` | 1604 家（虚构机构） | **66 家**（政府公示备案机构） |
> | `retirement_policy.avg_salary` | 401 条（按省估算，偏差可达 15%） | **401 条**（统计局口径真实值） |
>
> 三张表均新增 `source_name` / `source_url` / `source_date` 溯源字段，
> **无 `source_url` 不得入库**。规范详见 `database/sources/README.md`，批次计划与结果见 `database/sources/整改计划.md`。
>
> **真实数据必然稀疏**：我国高龄津贴、养老服务补贴、护理补贴**由区县制定标准**，
> 省级通常不设统一标准（如广东省内广州/深圳/东莞/珠海/肇庆各区标准均不同），
> 因此覆盖度从 3208 条收缩到 85 条是**如实反映现实**，而非功能缺失。
> 查不到的地区不填充 —— **宁可少而真，不要多而假**。

### 8.2 政策字段说明

| 字段 | 说明 |
|------|------|
| avgSalary | 当地上年度在岗职工月平均工资 |
| maleAge | 男性法定退休年龄（2025年前基准） |
| femaleCadreAge | 女性干部退休年龄 |
| femaleWorkerAge | 女性工人退休年龄 |
| transitionRatio | 过渡性养老金系数（各地1.2%~1.4%） |
| minPayYears | 最低缴费年限（一般为15年） |
| policyYear | 政策年份（用于版本管理） |

### 8.3 省级回退逻辑

如用户所在城市无独立政策数据，系统自动按省份编码取省级平均数据（`RetirementPolicyMapper.findProvincePolicy`）。

### 8.4 政策接口

```
GET /api/policy/list?keyword=北京          # 搜索城市政策
GET /api/policy/city/{areaCode}           # 获取指定城市政策
```

---

## 9. 智慧养老服务平台（Phase3）

> **当前状态：骨架实现，计划2027年Q4上线完整功能。**

### 9.1 规划功能

| 功能模块 | 说明 | 上线计划 |
|----------|------|---------|
| 养老服务搜索 | 按城市/类型搜索养老机构/居家服务 | 2027.Q4 |
| 服务预约下单 | 在线预约上门服务/机构入住 | 2027.Q4 |
| 机构入住申请 | 在线提交入住申请资料 | 2028.Q1 |
| 养老补贴政策查询 | 各地政府养老补贴政策查询 | 2028.Q1 |

### 9.2 当前占位接口

```
GET  /api/service/list              # 返回 "功能建设中" 提示
GET  /api/service/provider/{id}     # 返回占位数据
POST /api/service/book              # 返回 "即将上线" 提示
POST /api/service/apply             # 返回 "即将上线" 提示
GET  /api/service/subsidy           # 返回 "即将上线" 提示
```

---

## 10. 权限与安全机制

### 10.1 认证机制

- **无状态 JWT 认证**：每次请求通过 `AuthInterceptor` 解析 Bearer Token
- **ThreadLocal 用户上下文**：认证成功后将 `userId` 存入 `CURRENT_USER_ID` ThreadLocal，请求结束后清除
- **Spring Security** 双重防护：SecurityConfig + AuthInterceptor 协同工作

### 10.2 公开接口（无需登录）

```
POST /api/user/register
POST /api/user/login
POST /api/user/sms/send
POST /api/user/wx-login
POST /api/retirement/calculate    # 允许匿名计算（不保存）
GET  /api/policy/list
GET  /api/policy/city/**
GET  /doc.html                    # Knife4j文档
```

### 10.3 安全防护措施

| 安全点 | 实现方式 |
|--------|---------|
| 密码存储 | BCrypt 加密（强度10） |
| SQL注入 | MyBatis Plus 参数化查询，无原始SQL拼接 |
| XSS防护 | Spring Security 全局过滤 |
| CSRF | 无状态JWT，禁用CSRF（`csrf().disable()`） |
| 短信频率限制 | Redis锁，60秒内同一手机号只允许发送1次 |
| 支付回调验签 | 验证微信/支付宝签名后才处理订单状态 |
| 数据隔离 | 所有涉及用户数据的接口均校验 userId 归属 |
| 日志脱敏 | 手机号、密码、Token 不写入日志 |

### 10.4 接口限流（建议配置）

生产环境建议在 Nginx 层配置：
- 公开接口：50 req/min/IP
- 短信发送：5 req/min/IP
- AI对话：20 req/min/用户

---

## 11. 接口总览

| 模块 | 接口路径 | 方法 | 权限 |
|------|---------|------|------|
| 用户 | /api/user/register | POST | 公开 |
| 用户 | /api/user/login | POST | 公开 |
| 用户 | /api/user/sms/send | POST | 公开 |
| 用户 | /api/user/wx-login | POST | 公开 |
| 用户 | /api/user/profile | GET/PUT | 登录 |
| 退休计算 | /api/retirement/calculate | POST | 公开 |
| 退休计算 | /api/retirement/plan/save | POST | 登录 |
| 退休计算 | /api/retirement/plans | GET | 登录 |
| 退休计算 | /api/retirement/plan/{id} | GET/DELETE | 登录 |
| 退休计算 | /api/retirement/compare | POST | 登录 |
| 政策 | /api/policy/list | GET | 公开 |
| 政策 | /api/policy/city/{code} | GET | 公开 |
| 订单 | /api/order/report | POST | 登录 |
| 订单 | /api/order/member | POST | 登录 |
| 订单 | /api/order/status/{orderNo} | GET | 登录 |
| 订单 | /api/order/callback/wechat | POST | 公开（验签） |
| 订单 | /api/order/callback/alipay | POST | 公开（验签） |
| 报告 | /api/report/generate | POST | 登录 |
| 报告 | /api/report/{reportId} | GET | 登录 |
| 报告 | /api/report/download/{reportId} | GET | 登录 |
| AI顾问 | /api/ai/conversation | POST | 登录 |
| AI顾问 | /api/ai/chat | POST | 登录 |
| AI顾问 | /api/ai/history/{id} | GET | 登录 |
| AI顾问 | /api/ai/conversations | GET | 登录 |
| AI顾问 | /api/ai/report/{planId} | POST | 登录 |
| 养老服务 | /api/service/list | GET | 公开 |
| 养老服务 | /api/service/provider/{id} | GET | 公开 |
| 养老服务 | /api/service/book | POST | 登录 |
| 养老服务 | /api/service/apply | POST | 登录 |
| 养老服务 | /api/service/subsidy | GET | 公开 |

> 完整 Swagger 文档访问：`http://localhost:8080/doc.html`

---

## 12. 数据字典

### 12.1 用户状态枚举

| 字段 | 值 | 含义 |
|------|-----|------|
| sex | 1 | 男性 |
| sex | 2 | 女性（干部/管理岗） |
| sex | 3 | 女性（工人） |
| memberType | 0 | 免费用户 |
| memberType | 1 | 普通会员 |
| memberType | 2 | 高级会员 |
| status | 1 | 正常 |
| status | 0 | 禁用 |
| source | 1 | H5 Web |
| source | 2 | 微信小程序 |
| source | 3 | APP |

### 12.2 订单状态枚举

| payStatus | 含义 |
|-----------|------|
| 0 | 待支付 |
| 1 | 已支付 |
| 2 | 已退款 |
| 3 | 已取消 |

### 12.3 报告状态枚举

| status | 含义 |
|--------|------|
| 0 | 生成中 |
| 1 | 已完成 |
| 2 | 生成失败 |

### 12.4 业务状态码（ResultCode）

| 状态码 | 含义 |
|--------|------|
| 200 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未授权（需登录） |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |
| 1001 | 用户已存在 |
| 1002 | 用户不存在 |
| 1003 | 密码错误 |
| 1004 | 验证码错误或已过期 |
| 1005 | 验证码发送过于频繁 |
| 2001 | 方案不存在 |
| 2002 | 超出方案数量限额 |
| 3001 | 未找到该地区退休政策 |
| 4001 | 订单不存在 |
| 4002 | 订单状态异常 |
| 5001 | 会员权益不足 |
| 5002 | AI咨询次数已用完 |

---

*文档最后更新：2026年09月24日*

---

## 🏢 关于我们

<p align="center">
  <a href="http://www.net188.net">
    <img src="http://www.net188.net/images/logo1.png" alt="Net188 Logo" width="200" />
  </a>
</p>

<p align="center">
  <strong>Net188 · 互联网技术服务</strong>
</p>

<p align="center">
  专注于跨平台应用开发、AI Agent 集成与大模型应用落地。<br/>
  提供从产品设计、开发实施到部署运维的全栈技术解决方案。
</p>

<p align="center">
  🌐 <a href="http://www.net188.net"><strong>www.net188.net</strong></a>
</p>

---
