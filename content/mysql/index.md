---
title: 索引原理
weight: 1
---

## B+ 树索引

### Q: 为什么 MySQL 使用 B+ 树而不是 B 树？

| 对比 | B 树 | B+ 树 |
|------|------|-------|
| 数据存储 | 所有节点都存数据 | 只有叶子节点存数据 |
| 范围查询 | 需要中序遍历 | 叶子节点链表，直接扫描 |
| 磁盘IO | 每次查询 IO 不稳定 | 查询路径固定，IO 次数稳定 |
| 单页数据量 | 较少 | 更多（非叶节点只存 key）|

**结论**：B+ 树更适合磁盘存储，范围查询性能更优。

---

## 聚簇索引 vs 非聚簇索引

### Q: 什么是聚簇索引？

**聚簇索引**：数据行和索引存储在一起，叶子节点存放完整的行数据。

- InnoDB 默认以**主键**作为聚簇索引
- 如果没有主键，选第一个非空唯一索引
- 如果还没有，InnoDB 自动生成隐藏的 rowid

**非聚簇索引（二级索引）**：叶子节点存储的是**主键值**，查询需要**回表**。

```
二级索引查询流程：
name索引树 → 找到主键id → 主键索引树 → 找到完整行数据（回表）
```

### Q: 什么是覆盖索引？

当查询的所有列都在索引中时，不需要回表，称为**覆盖索引**。

```sql
-- 假设有联合索引 (name, age)
SELECT name, age FROM users WHERE name = 'Alice';
-- ✅ 覆盖索引，无需回表

SELECT name, age, email FROM users WHERE name = 'Alice';
-- ❌ email 不在索引中，需要回表
```

---

## 索引失效场景

### Q: 哪些情况会导致索引失效？

```sql
-- 1. 对索引列使用函数
WHERE YEAR(create_time) = 2024  -- ❌

-- 2. 隐式类型转换
WHERE phone = 13888888888  -- phone是varchar，❌

-- 3. 前导模糊查询
WHERE name LIKE '%Alice%'  -- ❌（前缀%）
WHERE name LIKE 'Alice%'   -- ✅

-- 4. 不符合最左前缀原则（联合索引）
-- 索引: (a, b, c)
WHERE b = 1 AND c = 2  -- ❌ 跳过了 a

-- 5. OR 条件一侧无索引
WHERE id = 1 OR name = 'Alice'  -- name无索引则全表扫描
```

{{% callout type="info" %}}
使用 `EXPLAIN` 查看执行计划，关注 `type` 字段：`const` > `ref` > `range` > `index` > `ALL`
{{% /callout %}}
