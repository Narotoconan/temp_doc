基于您的需求，我设计一个标准的库存管理数据库方案：## 设计要点说明：

**1. 表结构设计**

- **products**: 货物基础信息，一个产品对应多个SKU
- **skus**: 引入标准SKU概念，管理货物规格，使用JSON存储灵活规格属性
- **inventory_batches**: 核心库存表，按批次管理，包含生产日期和自动生成的生产月份字段

**2. 核心特性**

- **批次管理**: 每个SKU按生产批次分别管理库存，便于质量追溯
- **生产月份**: 使用计算列自动生成，优化按月统计查询性能
- **数量管理**: 区分当前库存、预留数量、可用数量
- **状态控制**: 支持正常、冻结、过期等状态管理

**3. 查询优化**

- **月度汇总视图**: 预聚合按月统计数据，提高查询效率
- **复合索引**: 针对SKU+生产日期的组合查询优化
- **计算列**: production_month存储索引，避免重复计算

**4. 扩展性**

- SKU规格使用JSON存储，支持灵活的属性扩展
- 预留仓库ID、库位等字段，支持多仓库管理
- 成本价字段支持财务核算需求

这个设计既满足了按生产日期统计库存的需求，又保持了数据结构的标准化和可扩展性。

```mysql
-- 货物库存统计数据库设计
-- 采用标准SKU体系，支持按生产日期统计库存

-- 1. 货物基础信息表 (Product)
CREATE TABLE products (
    product_id VARCHAR(32) PRIMARY KEY COMMENT '货物ID',
    product_name VARCHAR(200) NOT NULL COMMENT '货物名称',
    product_code VARCHAR(100) UNIQUE NOT NULL COMMENT '货物编码',
    category_id VARCHAR(32) COMMENT '分类ID',
    brand VARCHAR(100) COMMENT '品牌',
    unit VARCHAR(20) NOT NULL COMMENT '计量单位',
    description TEXT COMMENT '货物描述',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，0-停用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_product_code (product_code),
    INDEX idx_category (category_id)
) COMMENT '货物基础信息表';

-- 2. SKU规格表 (Stock Keeping Unit)
CREATE TABLE skus (
    sku_id VARCHAR(32) PRIMARY KEY COMMENT 'SKU ID',
    product_id VARCHAR(32) NOT NULL COMMENT '关联货物ID',
    sku_code VARCHAR(100) UNIQUE NOT NULL COMMENT 'SKU编码',
    sku_name VARCHAR(200) NOT NULL COMMENT 'SKU名称',
    specifications JSON COMMENT '规格属性JSON：{"颜色":"红色","尺寸":"L","重量":"1kg"}',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，0-停用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    INDEX idx_product (product_id),
    INDEX idx_sku_code (sku_code)
) COMMENT 'SKU规格表';

-- 3. 库存批次表 (Inventory Batch) - 核心表
CREATE TABLE inventory_batches (
    batch_id VARCHAR(32) PRIMARY KEY COMMENT '批次ID',
    sku_id VARCHAR(32) NOT NULL COMMENT 'SKU ID',
    batch_no VARCHAR(100) NOT NULL COMMENT '批次号',
    production_date DATE NOT NULL COMMENT '生产日期',
    production_month VARCHAR(7) AS (DATE_FORMAT(production_date, '%Y-%m')) STORED COMMENT '生产年月',
    expiry_date DATE COMMENT '过期日期',
    initial_quantity DECIMAL(15,3) NOT NULL DEFAULT 0 COMMENT '初始数量',
    current_quantity DECIMAL(15,3) NOT NULL DEFAULT 0 COMMENT '当前库存数量',
    reserved_quantity DECIMAL(15,3) NOT NULL DEFAULT 0 COMMENT '预留数量',
    available_quantity DECIMAL(15,3) AS (current_quantity - reserved_quantity) STORED COMMENT '可用数量',
    warehouse_id VARCHAR(32) COMMENT '仓库ID',
    location VARCHAR(100) COMMENT '库位',
    cost_price DECIMAL(12,4) COMMENT '成本价',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，2-冻结，3-过期',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (sku_id) REFERENCES skus(sku_id),
    INDEX idx_sku_production (sku_id, production_date),
    INDEX idx_production_month (production_month),
    INDEX idx_batch_no (batch_no),
    INDEX idx_warehouse (warehouse_id),
    UNIQUE KEY uk_batch_sku (batch_no, sku_id)
) COMMENT '库存批次表-按生产日期管理库存';

-- 4. 库存汇总表 (按月统计视图)
CREATE VIEW inventory_monthly_summary AS
SELECT 
    p.product_id,
    p.product_name,
    s.sku_id,
    s.sku_code,
    s.sku_name,
    ib.production_month,
    ib.warehouse_id,
    COUNT(ib.batch_id) as batch_count,
    SUM(ib.current_quantity) as total_quantity,
    SUM(ib.available_quantity) as available_quantity,
    SUM(ib.reserved_quantity) as reserved_quantity,
    AVG(ib.cost_price) as avg_cost_price,
    MIN(ib.production_date) as earliest_production_date,
    MAX(ib.production_date) as latest_production_date
FROM inventory_batches ib
JOIN skus s ON ib.sku_id = s.sku_id
JOIN products p ON s.product_id = p.product_id
WHERE ib.status = 1
GROUP BY p.product_id, s.sku_id, ib.production_month, ib.warehouse_id;

-- 5. 常用查询示例

-- 查询某SKU各月份生产的库存情况
-- SELECT * FROM inventory_monthly_summary 
-- WHERE sku_code = 'SKU001' 
-- ORDER BY production_month DESC;

-- 查询某货物在指定月份生产的所有批次库存
-- SELECT ib.*, s.sku_code, p.product_name
-- FROM inventory_batches ib
-- JOIN skus s ON ib.sku_id = s.sku_id  
-- JOIN products p ON s.product_id = p.product_id
-- WHERE p.product_code = 'PROD001' 
-- AND ib.production_month = '2024-06'
-- ORDER BY ib.production_date;

-- 索引优化建议：
-- 1. 对于频繁按生产月份查询的场景，production_month字段已建立索引
-- 2. 对于SKU+生产日期的组合查询，已建立复合索引
-- 3. 考虑根据实际查询模式添加其他组合索引
```





## 我想查2024-05月分货物A的红色库存，怎么查？

基于前面的数据库设计，查询2024年5月生产的货物A红色库存的SQL如下：查询说明：

```mysql
-- 货物库存统计数据库设计
-- 采用标准SKU体系，支持按生产日期统计库存

-- 1. 货物基础信息表 (Product)
CREATE TABLE products (
    product_id VARCHAR(32) PRIMARY KEY COMMENT '货物ID',
    product_name VARCHAR(200) NOT NULL COMMENT '货物名称',
    product_code VARCHAR(100) UNIQUE NOT NULL COMMENT '货物编码',
    category_id VARCHAR(32) COMMENT '分类ID',
    brand VARCHAR(100) COMMENT '品牌',
    unit VARCHAR(20) NOT NULL COMMENT '计量单位',
    description TEXT COMMENT '货物描述',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，0-停用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_product_code (product_code),
    INDEX idx_category (category_id)
) COMMENT '货物基础信息表';

-- 2. SKU规格表 (Stock Keeping Unit)
CREATE TABLE skus (
    sku_id VARCHAR(32) PRIMARY KEY COMMENT 'SKU ID',
    product_id VARCHAR(32) NOT NULL COMMENT '关联货物ID',
    sku_code VARCHAR(100) UNIQUE NOT NULL COMMENT 'SKU编码',
    sku_name VARCHAR(200) NOT NULL COMMENT 'SKU名称',
    specifications JSON COMMENT '规格属性JSON：{"颜色":"红色","尺寸":"L","重量":"1kg"}',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，0-停用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    INDEX idx_product (product_id),
    INDEX idx_sku_code (sku_code)
) COMMENT 'SKU规格表';

-- 3. 库存批次表 (Inventory Batch) - 核心表
CREATE TABLE inventory_batches (
    batch_id VARCHAR(32) PRIMARY KEY COMMENT '批次ID',
    sku_id VARCHAR(32) NOT NULL COMMENT 'SKU ID',
    batch_no VARCHAR(100) NOT NULL COMMENT '批次号',
    production_date DATE NOT NULL COMMENT '生产日期',
    production_month VARCHAR(7) AS (DATE_FORMAT(production_date, '%Y-%m')) STORED COMMENT '生产年月',
    expiry_date DATE COMMENT '过期日期',
    initial_quantity DECIMAL(15,3) NOT NULL DEFAULT 0 COMMENT '初始数量',
    current_quantity DECIMAL(15,3) NOT NULL DEFAULT 0 COMMENT '当前库存数量',
    reserved_quantity DECIMAL(15,3) NOT NULL DEFAULT 0 COMMENT '预留数量',
    available_quantity DECIMAL(15,3) AS (current_quantity - reserved_quantity) STORED COMMENT '可用数量',
    warehouse_id VARCHAR(32) COMMENT '仓库ID',
    location VARCHAR(100) COMMENT '库位',
    cost_price DECIMAL(12,4) COMMENT '成本价',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，2-冻结，3-过期',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (sku_id) REFERENCES skus(sku_id),
    INDEX idx_sku_production (sku_id, production_date),
    INDEX idx_production_month (production_month),
    INDEX idx_batch_no (batch_no),
    INDEX idx_warehouse (warehouse_id),
    UNIQUE KEY uk_batch_sku (batch_no, sku_id)
) COMMENT '库存批次表-按生产日期管理库存';

-- 4. 库存汇总表 (按月统计视图)
CREATE VIEW inventory_monthly_summary AS
SELECT 
    p.product_id,
    p.product_name,
    s.sku_id,
    s.sku_code,
    s.sku_name,
    ib.production_month,
    ib.warehouse_id,
    COUNT(ib.batch_id) as batch_count,
    SUM(ib.current_quantity) as total_quantity,
    SUM(ib.available_quantity) as available_quantity,
    SUM(ib.reserved_quantity) as reserved_quantity,
    AVG(ib.cost_price) as avg_cost_price,
    MIN(ib.production_date) as earliest_production_date,
    MAX(ib.production_date) as latest_production_date
FROM inventory_batches ib
JOIN skus s ON ib.sku_id = s.sku_id
JOIN products p ON s.product_id = p.product_id
WHERE ib.status = 1
GROUP BY p.product_id, s.sku_id, ib.production_month, ib.warehouse_id;

-- 5. 常用查询示例

-- 查询2024年5月生产的货物A红色库存
SELECT 
    p.product_name,
    s.sku_code,
    s.sku_name,
    ib.batch_no,
    ib.production_date,
    ib.current_quantity,
    ib.available_quantity,
    ib.warehouse_id,
    ib.location
FROM inventory_batches ib
JOIN skus s ON ib.sku_id = s.sku_id
JOIN products p ON s.product_id = p.product_id
WHERE p.product_name = '货物A'  -- 或使用 p.product_code = 'A001'
  AND ib.production_month = '2024-05'
  AND JSON_EXTRACT(s.specifications, '$.颜色') = '红色'
  AND ib.status = 1
ORDER BY ib.production_date;

-- 如果需要汇总数据（推荐）
SELECT 
    p.product_name,
    '红色' as color,
    ib.production_month,
    COUNT(ib.batch_id) as batch_count,
    SUM(ib.current_quantity) as total_quantity,
    SUM(ib.available_quantity) as available_quantity
FROM inventory_batches ib
JOIN skus s ON ib.sku_id = s.sku_id
JOIN products p ON s.product_id = p.product_id
WHERE p.product_name = '货物A'
  AND ib.production_month = '2024-05'
  AND JSON_EXTRACT(s.specifications, '$.颜色') = '红色'
  AND ib.status = 1
GROUP BY p.product_id, ib.production_month;

-- 查询某SKU各月份生产的库存情况
-- SELECT * FROM inventory_monthly_summary 
-- WHERE sku_code = 'SKU001' 
-- ORDER BY production_month DESC;

-- 索引优化建议：
-- 1. 对于频繁按生产月份查询的场景，production_month字段已建立索引
-- 2. 对于SKU+生产日期的组合查询，已建立复合索引
-- 3. 考虑根据实际查询模式添加其他组合索引
```



**方法1：详细批次查询**

- 显示每个批次的详细信息，包括批次号、生产日期、具体库存数
- 适合需要了解具体批次分布情况的场景

**方法2：汇总查询（推荐）**

- 将所有红色SKU的2024年5月库存汇总
- 显示总批次数、总库存量、可用库存量
- 查询效率更高，适合统计报表

**关键查询条件：**

- `p.product_name = '货物A'` - 指定货物
- `ib.production_month = '2024-05'` - 指定生产月份
- `JSON_EXTRACT(s.specifications, '$.颜色') = '红色'` - 从JSON规格中提取颜色属性
- `ib.status = 1` - 只查询正常状态的库存

**性能优化建议：** 如果经常按颜色查询，可以考虑添加虚拟列和索引：

```sql
ALTER TABLE skus ADD COLUMN color VARCHAR(50) AS (JSON_EXTRACT(specifications, '$.颜色'));
CREATE INDEX idx_color ON skus(color);
```



