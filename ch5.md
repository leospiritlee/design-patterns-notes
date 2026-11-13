# 第5章 依赖倒转原则

**一句话**：谁也别依赖谁的具体实现，只认中间的接口。

**故事**：娇娇电脑蓝屏，小菜在电话里指挥她打开机箱、拔掉一根内存条，居然修好了。大鸟问他为什么电脑能远程修、收音机不能——电脑的 CPU、内存、硬盘都是按标准接口插拔的，坏了换一个就行；收音机的电阻三极管全焊死在板子上，一处坏牵连一片，不懂的人根本无从下手。

**不用它会怎样**：业务类里直接 `new MySqlConnection()`，SQL 语句散落在 service 方法中。客户说这次要换成 Oracle，或者要支持国产数据库，你会发现改动点不是一处，而是所有出现过连接字符串的地方。更麻烦的是单测——想测一段下单逻辑，必须先有一个能连上的数据库。

另一种常见形态是"高层调低层函数库"：所有项目都调用同一套数据库访问函数，看起来是复用。可新项目要用另一种存储方式时，高层业务逻辑和旧的访问函数绑死在一起，一行都拿不走。**能复用的是函数，搬不走的是逻辑。**

**违反它会怎样**：

| 违反的表现 | 后果 |
|---|---|
| 业务类里 `new MySqlConnection()`、写死连接串 | 换数据库要改业务代码，改动点分散且不可枚举 |
| 高层模块直接调用低层的具体函数库 | 高层逻辑无法迁移到新项目，复用只停留在函数级 |
| 接口方法签名里出现具体实现的类型 | 换实现时接口被迫修改，所有实现类一起改 |
| 为了"复用"复制低层代码到新项目 | 两份逻辑各自演化，修 bug 要修两遍 |
| 依赖注入只做了一半：注入了接口，内部又 `new` 具体类 | 表面解耦，实际仍绑死，测试照样连库 |

**Java 实现**：

```java
import java.util.*;

class Order {
    final String id;
    Order(String id) { this.id = id; }
}

interface OrderRepository {                    // 抽象：高低层都依赖它
    void save(Order order);
    Order findById(String id);
}

class MySqlOrderRepository implements OrderRepository {
    private final Map<String, Order> db = new HashMap<>();
    public void save(Order o) {
        db.put(o.id, o);
        System.out.println("写入 MySQL: " + o.id);
    }
    public Order findById(String id) { return db.get(id); }
}

class MemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new HashMap<>();
    public void save(Order o) {
        store.put(o.id, o);
        System.out.println("写入内存: " + o.id);
    }
    public Order findById(String id) { return store.get(id); }
}

class OrderService {                           // 高层模块：只认接口
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }   // 构造注入
    public void placeOrder(String id) {
        repo.save(new Order(id));
        System.out.println("下单完成: " + repo.findById(id).id);
    }
}

public class Shop {
    public static void main(String[] args) {
        new OrderService(new MySqlOrderRepository()).placeOrder("A1001");
        new OrderService(new MemoryOrderRepository()).placeOrder("A1002");  // 单测用内存版
    }
}
```

**什么时候用**：
- 跨模块、跨进程、跨存储调用——数据库、消息队列、第三方 API、文件系统。
- 同一段业务逻辑要跑在多种实现上（生产用 MySQL，测试用内存，离线用文件）。
- 需要在不启动外部依赖的情况下做单元测试。
- 依赖的东西由别人维护、版本会变（SDK、外部服务）。

**什么时候别用**：
- 稳定的 JDK 工具类、字符串处理、集合操作，不必再包一层接口。
- 每个类都配一个接口，接口数量超过实现数量，读代码要来回跳。只在真正会替换的地方倒转。
- 一次性脚本、内部小工具，注入框架的配置成本高于收益。

**易错点**：
- 有接口但仍 `new` 具体类——`private OrderRepository repo = new MySqlOrderRepository();`，倒转了个寂寞。
- 接口按实现定制：`void save(Order o, MySqlConnection conn)`，抽象里带着细节，换实现就崩。
- 把依赖倒转等同于"用了 Spring"。框架只是装配工具，类里写死具体类型，注入了也白搭。

**和谁像、怎么区分**：里氏代换原则——里氏保证"子类能顶替父类而不出错"，依赖倒转保证"只依赖能被顶替的东西"；前者是后者的前提，没有可替换性，依赖抽象就是空谈。控制反转则是依赖倒转的一种实现手段（把创建权交出去）。

# 《大话设计模式》精华版 · 第 2 组
