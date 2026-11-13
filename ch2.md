# 第2章 策略模式

**一句话**：把一族可互换的算法各自关进类里，换算法像换电池。

**故事**：大鸟让小菜做商场收银软件。第一版算完总价，大鸟说"所有商品打八折"，小菜想直接乘 0.8；再问"活动结束怎么办""周年庆打五折怎么办"；接着又加"满 300 返 100"。小菜想给每种折扣写一个子类，被大鸟拦住：打一折和打九折只是参数不同，本质是同一个算法。

**不用它会怎样**：v1.0 是 `total = price * num`。要打八折，改成 `total = price * num * 0.8`；活动结束，改回去；再来个五折，再改一次。每改一次，所有收银机都要重装一遍。

v1.1 加了下拉框，`switch` 三个分支，每个分支干的事几乎一样，只有折扣系数不同——重复代码三份。这时候来一个"满 300 返 100"，`switch` 装不下了，得写函数；再来"满 100 积 10 分"，函数里套 `if`；再来"第二件半价"，`if` 里再套 `if`。到最后，改一个返利门槛（300 改成 200），要在几十行分支里翻找，改完不敢确认有没有碰到别的活动。

**结构**：

| 角色 | 职责 |
|---|---|
| CashSuper（抽象策略） | 声明 `acceptCash(money)`，统一收费入口 |
| CashNormal / CashRebate / CashReturn（具体策略） | 各封装一种收费算法，用构造参数区分力度，而不是靠子类数量 |
| CashContext（上下文） | 持有一个策略，对外只暴露 `getResult`，屏蔽策略类型 |
| Client | 只与 Context 交互，不知道算法存在 |

**Java 实现**：

```java
abstract class CashSuper {
    public abstract double acceptCash(double money);
}

class CashNormal extends CashSuper {
    public double acceptCash(double money) { return money; }
}

class CashRebate extends CashSuper {          // 折扣：一个参数搞定所有折扣率
    private final double rate;
    public CashRebate(double rate) { this.rate = rate; }
    public double acceptCash(double money) { return money * rate; }
}

class CashReturn extends CashSuper {          // 返利：满 threshold 返 back
    private final double threshold, back;
    public CashReturn(double threshold, double back) {
        this.threshold = threshold;
        this.back = back;
    }
    public double acceptCash(double money) {
        if (money < threshold) return money;
        return money - Math.floor(money / threshold) * back;
    }
}

class CashContext {                            // 策略 + 简单工厂
    private final CashSuper strategy;

    public CashContext(String type) {
        switch (type) {
            case "正常收费":    strategy = new CashNormal();          break;
            case "打八折":      strategy = new CashRebate(0.8);       break;
            case "满300返100":  strategy = new CashReturn(300, 100);  break;
            default: throw new IllegalArgumentException("未知促销: " + type);
        }
    }

    public double getResult(double money) { return strategy.acceptCash(money); }
}

public class Cashier {
    public static void main(String[] args) {
        System.out.println(new CashContext("正常收费").getResult(700));    // 700.0
        System.out.println(new CashContext("打八折").getResult(700));      // 560.0
        System.out.println(new CashContext("满300返100").getResult(700)); // 500.0
    }
}
```

**什么时候用**：
- 同一件事有多种算法，且使用时会互相替换（促销、计费、路由、限流、压缩）。
- 代码里出现按类型分支的 `switch` / `if-else` 链，且分支还在增长。
- 每个算法都想单独写单元测试，不想为了测一个折扣去构造整个收银流程。
- 业务规则会随时间、客户、地区变化，而"变化的那部分"能被清晰地圈出来。

**什么时候别用**：
- 只有一两个分支、且确定不会增长。一个 `if` 比三个类加一个 Context 更好读。
- 算法需要调用方传入大量特有参数（比如 A 要三个参数、B 要一个配置对象），硬塞进统一接口会让接口变形，反而更难用。

**易错点**：
- 把 `switch` 从客户端搬进 `CashContext` 就以为消除了条件语句——只是挪了位置，加算法照样要改 Context。真要彻底解决得上反射或配置化注册。
- 具体策略里藏了可变状态。策略对象通常应当无状态、可共享；一旦有状态，多个 Context 共用一个实例就会串数据。
- 客户端仍然 `new CashRebate(0.8)`，那就等于没封装，调用方还是认识具体类。

**和谁像、怎么区分**：状态模式——类图几乎一样，差别在意图：策略是外部选好再塞进去，策略之间互不知道；状态是对象内部自己流转，状态往往知道下一个状态是谁。简单工厂——一个解决"造对象"，一个解决"换算法"，实践中常常合并成一个 Context。
