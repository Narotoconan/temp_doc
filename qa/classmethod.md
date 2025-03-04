# classmethod

---

在 Python 实际开发中，类方法（`@classmethod`）的应用远不止基础的工厂方法，以下是 7 个典型的实际应用场景和代码示例，涵盖从设计模式到框架开发的高级用法：

---

### 1. **多态构造器（替代 `__init__`）**
```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    @classmethod
    def from_polar(cls, radius, theta):
        """极坐标构造器"""
        x = radius * math.cos(theta)
        y = radius * math.sin(theta)
        return cls(x, y)

# 使用极坐标创建向量
v = Vector.from_polar(5, math.pi/4)
print(v.x, v.y)  # 输出：3.5355, 3.5355
```
**优势**：提供多种对象创建方式，保持构造函数简洁。

---

### 2. **单例模式实现**
```python
class DatabaseConnection:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            # 初始化数据库连接
        return cls._instance

    @classmethod
    def get_instance(cls):
        """全局唯一实例获取入口"""
        if not cls._instance:
            cls()  # 触发__new__
        return cls._instance

# 使用
db1 = DatabaseConnection.get_instance()
db2 = DatabaseConnection.get_instance()
print(db1 is db2)  # True
```
**场景**：全局资源配置（数据库连接池、配置中心）。

---

### 3. **策略模式注册器**
```python
class PaymentStrategy:
    _strategies = {}

    @classmethod
    def register(cls, name):
        """支付策略注册装饰器"""
        def decorator(subclass):
            cls._strategies[name] = subclass
            return subclass
        return decorator

    @classmethod
    def get_strategy(cls, name):
        """动态获取策略实现"""
        return cls._strategies.get(name)

@PaymentStrategy.register("alipay")
class AlipayStrategy:
    def pay(self, amount):
        print(f"支付宝支付 {amount} 元")

@PaymentStrategy.register("wechat")
class WeChatPayStrategy:
    def pay(self, amount):
        print(f"微信支付 {amount} 元")

# 使用
strategy = PaymentStrategy.get_strategy("alipay")()
strategy.pay(100)  # 输出：支付宝支付 100 元
```
**应用**：插件化架构、支付网关等需要动态扩展的场景。

---

### 4. **ORM 级联操作**
```python
class User(Model):
    @classmethod
    def get_active_users(cls):
        """获取所有活跃用户（类级别查询）"""
        return cls.query.filter_by(is_active=True).all()

    @classmethod
    def bulk_create(cls, user_list):
        """批量创建用户"""
        with db.session.begin():
            for user_data in user_list:
                user = cls(**user_data)
                db.session.add(user)
        return True

# 使用
active_users = User.get_active_users()
User.bulk_create([{"name": "Alice"}, {"name": "Bob"}])
```
**优势**：将数据库操作封装在模型类内部，保持代码组织性。

---

### 5. **模板方法模式**
```python
class ReportGenerator:
    @classmethod
    def generate(cls):
        """报表生成模板方法"""
        data = cls._fetch_data()
        processed = cls._process_data(data)
        cls._export(processed)

    @classmethod
    def _fetch_data(cls):
        raise NotImplementedError

    @classmethod
    def _process_data(cls, data):
        raise NotImplementedError

    @classmethod
    def _export(cls, data):
        raise NotImplementedError

class SalesReport(ReportGenerator):
    @classmethod
    def _fetch_data(cls):
        return db.query(Sales).all()

    @classmethod
    def _process_data(cls, data):
        return [s.amount for s in data]

    @classmethod
    def _export(cls, data):
        print(f"销售总额: {sum(data)}")

# 使用
SalesReport.generate()
```
**特点**：通过类方法实现算法骨架，子类填充具体实现。

---

### 6. **类级别缓存管理**
```python
class GeoService:
    _cache = {}
    _cache_expiry = 3600  # 1小时

    @classmethod
    def get_city_coordinates(cls, city_name):
        """带缓存的坐标查询"""
        if city_name in cls._cache:
            cached_time, data = cls._cache[city_name]
            if time.time() - cached_time < cls._cache_expiry:
                return data

        # 调用外部API
        data = cls._call_geo_api(city_name)
        cls._cache[city_name] = (time.time(), data)
        return data

    @classmethod
    def clear_cache(cls):
        cls._cache.clear()
```
**用途**：管理类级别的共享资源缓存。

---

### 7. **跨类协作的入口方法**
```python
class Order:
    @classmethod
    def create_from_cart(cls, cart):
        """从购物车创建订单"""
        order = cls()
        order.items = [cls._convert_item(item) for item in cart.items]
        order.total = sum(item.price for item in order.items)
        InventoryService.reserve_items(order.items)  # 调用库存服务
        return order

    @classmethod
    def _convert_item(cls, cart_item):
        return OrderItem(
            product_id=cart_item.product_id,
            quantity=cart_item.quantity
        )

# 使用
order = Order.create_from_cart(user_cart)
```
**优势**：封装复杂对象构建过程，协调多个子系统。

---

### 类方法 vs 静态方法的选择标准

| **场景**               | **类方法** | **静态方法** |
| ---------------------- | ---------- | ------------ |
| 需要访问类属性/方法    | ✔️          | ❌            |
| 需要被子类覆盖实现     | ✔️          | ❌            |
| 需要创建类实例         | ✔️          | ❌            |
| 纯粹的工具函数         | ❌          | ✔️            |
| 与类密切相关的逻辑组织 | ✔️          | ✔️            |

---

### 实际开发建议

1. **优先使用类方法的场景**：
   - 需要维护类级别状态（计数器、缓存）
   - 实现替代构造函数（工厂方法）
   - 处理继承体系中的多态行为

2. **避免滥用的情况**：
   - 纯工具函数（应使用静态方法或模块函数）
   - 需要大量实例属性访问的操作（应使用实例方法）

3. **性能优化技巧**：
   ```python
   @classmethod
   def heavy_computation(cls):
       # 缓存计算结果
       if not hasattr(cls, "_cache"):
           cls._cache = cls._do_computation()
       return cls._cache
   ```

4. **框架开发模式**：
   ```python
   class PluginBase:
       _plugins = []
       
       @classmethod
       def register(cls, plugin):
           """插件注册入口"""
           if not issubclass(plugin, cls):
               raise TypeError
           cls._plugins.append(plugin)
   ```

---

类方法的核心价值在于其 **对类本身的操作能力** 和 **在继承体系中的多态表现**，合理使用可以大幅提升代码的灵活性和可维护性。在实际项目中，结合具体业务场景灵活选择方法类型，才是面向对象设计的精髓所在。