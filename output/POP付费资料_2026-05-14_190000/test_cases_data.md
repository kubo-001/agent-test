# POP付费资料 - 数据层测试用例

> 本文档专注于数据层面的测试用例，覆盖幂等性、数据一致性、权益状态、软删除过滤、权限校验、边界场景。

---

## 1. 幂等性测试

### TC-D001: 三方订单重复回调不重复创建订单
- **前置条件**: material_order表无记录，存在商品ProductA
- **测试步骤**:
  1. 调用 `/service/product/open`，入参：orderSn=ORDER001, productIds=[ProductA], ucId=UC001, categoryId=CAT001, orderSource=third_party, amount=99, paidTime=2026-05-14 10:00:00
  2. 再次调用相同入参的 `/service/product/open`（相同orderSn）
  3. 查询 material_order 表记录数
  4. 查询 material_member 表记录数
- **预期结果**:
  - 第1次调用：返回 errors=[]，material_order 插入1条记录，material_member 插入1条记录
  - 第2次调用：返回 errors=["已开通"] 或 errors 包含已存在订单提示，material_order 仍是1条，material_member 仍是1条
  - uk_order_source_product_user 唯一约束未被破坏
- **优先级**: P0
- **测试类型**: 功能

### TC-D002: 同一用户同一商品重复开通返回已存在
- **前置条件**: UC001 已拥有 ProductA 的 material_member 权益
- **测试步骤**:
  1. 调用 `/service/product/open`，入参：ucId=UC001, categoryId=CAT001, productIds=[ProductA], orderSn=ORDER002, orderSource=third_party, amount=99
  2. 检查返回的 errors 字段
- **预期结果**: errors 包含 "已开通" 或类似提示，不重复创建 material_member 记录
- **优先级**: P0
- **测试类型**: 功能

### TC-D003: 不同订单来源的相同商品均可正常开通
- **前置条件**: UC001 曾在其他平台购买过 ProductA
- **测试步骤**:
  1. 调用 `/service/product/open`，orderSource=platform_a, orderSn=ORD-A-001
  2. 调用 `/service/product/open`，orderSource=platform_b, orderSn=ORD-B-001
  3. 查询 material_order 表
- **预期结果**: 两平台订单均记录，uk_order_source_product_user 约束校验通过（不同orderSource）
- **优先级**: P1
- **测试类型**: 功能

---

## 2. 数据一致性测试

### TC-D004: 多商品原子性写入-全部成功
- **前置条件**: ProductA、ProductB、ProductC 均有效
- **测试步骤**:
  1. 调用 `/service/product/open`，productIds=[ProductA, ProductB, ProductC], orderSn=ORDER003
  2. 查询 material_order 表记录
  3. 查询 material_member 表记录
- **预期结果**:
  - material_order 插入3条记录（每个商品1条）
  - material_member 插入3条记录（每个商品1条）
  - 数据一致，无遗漏
- **优先级**: P0
- **测试类型**: 功能

### TC-D005: 多商品原子性写入-部分商品无效
- **前置条件**: ProductA 有效，ProductB 不存在，ProductC 有效
- **测试步骤**:
  1. 调用 `/service/product/open`，productIds=[ProductA, ProductB, ProductC], orderSn=ORDER004
  2. 检查返回的 errors 字段
  3. 查询 material_order 表
  4. 查询 material_member 表
- **预期结果**:
  - errors 包含 ProductB 的错误信息
  - ProductA 和 ProductC 的记录正常写入
  - material_order 有2条记录（ProductA、ProductC）
  - material_member 有2条记录（ProductA、ProductC）
- **优先级**: P0
- **测试类型**: 异常

### TC-D006: 多商品原子性写入-全部失败
- **前置条件**: ProductA、ProductB 均无效（status≠1 或 is_del=1）
- **测试步骤**:
  1. 调用 `/service/product/open`，productIds=[ProductA, ProductB], orderSn=ORDER005
  2. 检查返回的 errors
  3. 查询 material_order 表
  4. 查询 material_member 表
- **预期结果**:
  - errors 包含所有失败商品的信息
  - material_order 无新增记录
  - material_member 无新增记录
  - 事务回滚，无数据污染
- **优先级**: P0
- **测试类型**: 异常

### TC-D007: 订单和权益的原子性保证
- **前置条件**: 数据库正常连接
- **测试步骤**:
  1. 开启事务，插入 material_order
  2. 插入 material_member
  3. 模拟插入 material_member 时主键冲突
  4. 检查事务回滚状态
- **预期结果**: 事务回滚，material_order 记录不保留（数据库层面保证原子性）
- **优先级**: P1
- **测试类型**: 功能

---

## 3. 权益状态测试

### TC-D008: 永久权益 deadline=0 验证
- **前置条件**: UC001 已开通 ProductA 权益
- **测试步骤**:
  1. 调用权益查询接口或直接从 material_member 表查询
  2. 检查 deadline 字段值
- **预期结果**: deadline = 0（或空/NULL，代码层固定返回0）
- **优先级**: P0
- **测试类型**: 功能

### TC-D009: 权益生效状态验证
- **前置条件**: UC001 已开通 ProductA，ProductA 状态正常（status=1, is_del=0）
- **测试步骤**:
  1. 查询 UC001 的 material_member
  2. 调用首页 paidMaterial 接口
- **预期结果**:
  - material_member 记录存在且 status=1
  - 首页 paidMaterial 返回 ProductA 相关数据
- **优先级**: P0
- **测试类型**: 功能

### TC-D010: 权益关闭状态流转
- **前置条件**: UC001 拥有 ProductA 的有效权益
- **测试步骤**:
  1. 将 material_member 的 status 更新为 0（模拟关闭）
  2. 调用首页 paidMaterial 接口
  3. 调用商品列表接口
- **预期结果**: 接口返回数据中不包含 ProductA（权益已关闭）
- **优先级**: P1
- **测试类型**: 功能

### TC-D011: 权益查询返回 deadline=0 的代码逻辑验证
- **前置条件**: material_member 表有数据，deadline 字段为 NULL 或 0
- **测试步骤**:
  1. 代码审查或日志检查权益返回逻辑
  2. 调用权益相关接口
- **预期结果**: 接口返回的 deadline 字段值固定为 0，不受数据库值影响
- **优先级**: P1
- **测试类型**: 功能

---

## 4. 软删除与过滤测试

### TC-D012: 商品下架后首页不展示
- **前置条件**: UC001 已购买 ProductA，ProductA 原状态正常
- **测试步骤**:
  1. 将 material_product 的 status 更新为 0 或 is_del 更新为 1
  2. 调用首页 paidMaterial 接口
- **预期结果**: 首页返回数据中不包含 ProductA（已被过滤）
- **优先级**: P0
- **测试类型**: 功能

### TC-D013: 商品下架后商品列表不展示
- **前置条件**: UC001 已购买 ProductA
- **测试步骤**:
  1. 将 material_product 设置为下架状态（status=0 或 is_del=1）
  2. 调用 `/ahuyikao/popMaterial/productList` 接口
- **预期结果**: 商品列表中不包含 ProductA
- **优先级**: P0
- **测试类型**: 功能

### TC-D014: 商品下架后用户权益不物理删除
- **前置条件**: UC001 已购买 ProductA
- **测试步骤**:
  1. 将 material_product 设置为下架状态（status=0 或 is_del=1）
  2. 查询 material_member 表
- **预期结果**: material_member 记录仍然存在，仅展示层过滤
- **优先级**: P0
- **测试类型**: 功能

### TC-D015: 资料下架后资料列表不展示
- **前置条件**: ProductA 下有 Material1、Material2，其中 Material1 已下架
- **测试步骤**:
  1. 将 Material1 的 status 设置为 0 或 is_del 设置为 1
  2. 调用 `/ahuyikao/popMaterial/materialList`，productId=ProductA
- **预期结果**: 返回的列表中不包含 Material1，仅返回 Material2
- **优先级**: P0
- **测试类型**: 功能

### TC-D016: 商品无上架资料时列表过滤
- **前置条件**: ProductA 下所有资料均已下架（status≠1 或 is_del=1）
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/productList`
  2. 调用首页 paidMaterial
- **预期结果**: ProductA 不出现在列表中（展示条件：商品下有上架PDF资料）
- **优先级**: P1
- **测试类型**: 边界

### TC-D017: 上下架过滤条件组合验证
- **前置条件**: ProductA 状态为 status=1, is_del=0，但下无任何上架资料
- **测试步骤**:
  1. 查询 material_product 状态
  2. 查询 material_material 表关联的上架状态
  3. 调用各展示接口
- **预期结果**: 上下架过滤条件为 status=1 AND is_del=0 AND 存在上架PDF资料，三者同时满足才展示
- **优先级**: P1
- **测试类型**: 边界

---

## 5. 权限校验测试

### TC-D018: 无权益用户不能访问资料列表
- **前置条件**: UC002 未购买任何资料
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/materialList`，categoryId=CAT001, productId=ProductA（UC002无权益）
  2. 检查返回结果
- **预期结果**: 返回权限错误或空数据，拒绝访问
- **优先级**: P0
- **测试类型**: 功能

### TC-D019: 有权益用户可以正常访问资料列表
- **前置条件**: UC001 已有 ProductA 的有效权益
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/materialList`，categoryId=CAT001, productId=ProductA
  2. 检查返回的 material 列表
- **预期结果**: 返回有效的资料列表（已上架且未删除的）
- **优先级**: P0
- **测试类型**: 功能

### TC-D020: 跨科目权益校验-A用户访问B科目商品
- **前置条件**: UC001 拥有 CAT001 科目的 ProductA 权益，但尝试访问 CAT002 的 ProductB
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/materialList`，categoryId=CAT002, productId=ProductB
- **预期结果**: 返回权限错误或空数据，跨科目校验不通过
- **优先级**: P0
- **测试类型**: 功能

### TC-D021: 权益失效后权限立即失效
- **前置条件**: UC001 曾拥有 ProductA 权益（material_member 存在）
- **测试步骤**:
  1. 将 material_member 的 status 更新为 0
  2. 调用 `/ahuyikao/popMaterial/materialList`，categoryId=CAT001, productId=ProductA
- **预期结果**: 返回权限错误，权益状态变更后权限校验立即生效
- **优先级**: P1
- **测试类型**: 功能

### TC-D022: 同一用户同一科目同一商品只保留一条有效权益
- **前置条件**: UC001 在 CAT001 科目下对 ProductA 有两条 material_member 记录（异常数据）
- **测试步骤**:
  1. 插入第二条记录（uk_user_product_category 约束被绕过）
  2. 调用接口查询权益
- **预期结果**: 唯一约束 uk_user_product_category 保证同一用户同一科目同一商品只有一条有效权益
- **优先级**: P1
- **测试类型**: 异常

---

## 6. 边界测试

### TC-D023: 无效商品ID处理-商品不存在
- **前置条件**: ProductNonExist 不存在于 material_product 表
- **测试步骤**:
  1. 调用 `/service/product/open`，productIds=[ProductNonExist], orderSn=ORDER100
  2. 检查 errors 返回值
- **预期结果**: errors 包含商品不存在的错误信息
- **优先级**: P0
- **测试类型**: 边界

### TC-D024: 空商品ID列表处理
- **前置条件**: 无
- **测试步骤**:
  1. 调用 `/service/product/open`，productIds=[], orderSn=ORDER101
  2. 检查返回结果
- **预期结果**: errors 包含参数校验错误，或正常返回但无数据写入
- **优先级**: P1
- **测试类型**: 边界

### TC-D025: 空资料列表-商品有效但无资料
- **前置条件**: ProductA 状态正常，但无任何 material 记录
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/materialList`，productId=ProductA
- **预期结果**: 返回空列表，不报错
- **优先级**: P1
- **测试类型**: 边界

### TC-D026: 商品下无上架资料时首页不展示
- **前置条件**: ProductA 有资料但全部下架
- **测试步骤**:
  1. 调用首页 paidMaterial 接口
- **预期结果**: ProductA 不展示（展示条件包含：商品下有上架PDF资料）
- **优先级**: P0
- **测试类型**: 边界

### TC-D027: 分页边界-页码超出
- **前置条件**: 某商品有大量资料（>100条）
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/materialList`，productId=ProductA, page=999, pageSize=10
- **预期结果**: 返回空列表或正确提示无更多数据
- **优先级**: P2
- **测试类型**: 边界

### TC-D028: 分页边界-页大小为0或负数
- **前置条件**: 有资料数据
- **测试步骤**:
  1. 调用 `/ahuyikao/popMaterial/materialList`，pageSize=0
  2. 调用 `/ahuyikao/popMaterial/materialList`，pageSize=-1
- **预期结果**: 返回错误或使用默认值（10）
- **优先级**: P2
- **测试类型**: 边界

### TC-D029: 订单金额边界-0元订单
- **前置条件**: 无
- **测试步骤**:
  1. 调用 `/service/product/open`，amount=0, productIds=[ProductA]
  2. 检查返回和数据库记录
- **预期结果**: 允许0元订单开通（或按业务规则校验），material_order 记录正常
- **优先级**: P1
- **测试类型**: 边界

### TC-D030: 订单金额边界-负数金额
- **前置条件**: 无
- **测试步骤**:
  1. 调用 `/service/product/open`，amount=-1, productIds=[ProductA]
  2. 检查参数校验
- **预期结果**: 返回参数校验错误，订单不创建
- **优先级**: P1
- **测试类型**: 异常

---

## 7. 缓存相关测试

### TC-D031: Redis缓存-POP_ALERT_CACHE_KEY写入
- **前置条件**: Redis 服务正常
- **测试步骤**:
  1. 调用开通接口 `/service/product/open`
  2. 查询 Redis，key=POP_ALERT_CACHE_KEY
- **预期结果**: 缓存写入成功，value 包含 mainCategoryId, targetType=30, targetId, targetName, productName, isAllocation
- **优先级**: P1
- **测试类型**: 功能

### TC-D032: Redis缓存过期时间验证
- **前置条件**: 缓存已写入
- **测试步骤**:
  1. 查询缓存 TTL
  2. 等待 30 天后再次查询
- **预期结果**: 过期时间约 30 天（2592000 秒）
- **优先级**: P1
- **测试类型**: 功能

### TC-D033: 缓存穿透-商品不存在时无缓存写入
- **前置条件**: 无
- **测试步骤**:
  1. 调用开通接口，productIds=[不存在的商品]
  2. 查询 Redis 缓存
- **预期结果**: 无缓存写入（或写入错误标记）
- **优先级**: P2
- **测试类型**: 边界

---

## 8. 数据完整性测试

### TC-D034: 唯一约束uk_order_source_product_user验证
- **前置条件**: 已有订单记录（orderSn=ORDER200, orderSource=third_party, productId=ProductA, userId=UC001）
- **测试步骤**:
  1. 尝试插入相同组合的记录
- **预期结果**: 唯一约束报错，重复数据拒绝写入
- **优先级**: P0
- **测试类型**: 功能

### TC-D035: 唯一约束uk_user_product_category验证
- **前置条件**: UC001 在 CAT001 下已有 ProductA 的 material_member 记录
- **测试步骤**:
  1. 尝试插入相同的 material_member 记录
- **预期结果**: 唯一约束报错，同一用户同一科目同一商品只保留一条
- **优先级**: P0
- **测试类型**: 功能

### TC-D036: 索引idx_user_category_status有效性
- **前置条件**: material_member 表有大量数据
- **测试步骤**:
  1. 执行查询：SELECT * FROM material_member WHERE user_id=? AND category_id=? AND status=1
  2. 检查 EXPLAIN 结果
- **预期结果**: 使用 idx_user_category_status 索引，查询高效
- **优先级**: P1
- **测试类型**: 功能

### TC-D037: 索引idx_order_sn有效性
- **前置条件**: material_order 表有大量数据
- **测试步骤**:
  1. 执行查询：SELECT * FROM material_order WHERE order_sn=?
  2. 检查 EXPLAIN 结果
- **预期结果**: 使用 idx_order_sn 索引，查询高效
- **优先级**: P1
- **测试类型**: 功能

---

## 9. 异常场景测试

### TC-D038: 数据库连接失败时的幂等性处理
- **前置条件**: 数据库连接异常
- **测试步骤**:
  1. 模拟数据库连接超时
  2. 调用开通接口
- **预期结果**: 返回错误，errors 包含数据库错误信息，不产生脏数据
- **优先级**: P0
- **测试类型**: 异常

### TC-D039: Redis服务不可用时的降级处理
- **前置条件**: Redis 服务停止
- **测试步骤**:
  1. 调用开通接口
  2. 检查返回结果
- **预期结果**: 接口正常返回，缓存写入失败不影响主流程
- **优先级**: P1
- **测试类型**: 异常

### TC-D040: 并发开通同一商品-先到先得
- **前置条件**: UC001 未购买 ProductA
- **测试步骤**:
  1. 并发调用两次 `/service/product/open`，相同 ucId, categoryId, productId，不同 orderSn
  2. 检查最终数据状态
- **预期结果**: 只有一个成功创建 material_member，或均成功但 orderSn 不同（最终一致性）
- **优先级**: P0
- **测试类型**: 异常

### TC-D041: 订单号为空时的处理
- **前置条件**: 无
- **测试步骤**:
  1. 调用 `/service/product/open`，orderSn=null 或空字符串
  2. 检查返回和数据库状态
- **预期结果**: 返回参数校验错误，或使用系统生成的订单号
- **优先级**: P1
- **测试类型**: 异常

### TC-D042: 订单来源为空时的处理
- **前置条件**: 无
- **测试步骤**:
  1. 调用 `/service/product/open`，orderSource=null 或空字符串
  2. 检查数据库约束和返回
- **预期结果**: 返回错误或使用默认值，uk_order_source_product_user 约束仍有效
- **优先级**: P1
- **测试类型**: 异常

---

## 测试覆盖率总结

| 测试类型 | 用例数 | 覆盖场景 |
|---------|--------|---------|
| 功能测试 | 18 | 幂等性、权益状态、权限校验、数据一致性 |
| 边界测试 | 9 | 空数据、无效ID、分页边界、金额边界 |
| 异常测试 | 8 | 数据库异常、Redis异常、并发、参数校验 |
| 数据完整性 | 4 | 唯一约束、索引有效性 |
| 缓存测试 | 3 | 写入、过期、穿透 |
| **总计** | **42** | - |

| 优先级 | 用例数 |
|--------|--------|
| P0 | 16 |
| P1 | 20 |
| P2 | 6 |

---

*文档生成时间: 2026-05-15*
*测试类型: 数据层测试*