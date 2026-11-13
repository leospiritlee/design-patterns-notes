# 第8章 工厂制造细节无须知——工厂方法模式

**一句话**：加一种新产品时只加新类，不动已经稳定的老代码。

**故事**：小菜炫耀简单工厂到处好用。大鸟用"猪怎么变成红烧肉"打比方——养殖、运输、屠宰、烹饪的细节你都不必知道，点菜就行。小菜写完计算器却更困惑了：工厂方法明明比简单工厂多了好几个类，凭什么说它更好？

**不用它会怎样**：简单工厂的全部逻辑压在一个 `OperationFactory` 的 switch 上：

```java
switch (operate) {
    case "+": return new OperationAdd();
    case "-": return new OperationSub();
    case "*": return new OperationMul();
    case "/": return new OperationDiv();
}
```

加减乘除这四个类其实早就稳定了，半年没人动过。可只要加一个"求 x 的 n 次方"，就必须打开这个文件、在 switch 里插一个 case、重新编译、重新测试整个工厂——**为了一个新增功能，去改一段已经验证过的成熟代码**。这就是把鸡蛋全放在一个篮子里。

更糟的是这个文件会一直长：指数、对数、正余弦、阶乘……每个新运算都往同一个 switch 里挤，冲突、回归、误改全集中在这一处。而且它是全项目最容易被多人同时修改的文件，合并冲突是家常便饭。

工厂方法做的事很朴素：把这个 switch **拆成一个个工厂类**。以后加指数运算，只是新增 `Pow` 和 `PowFactory` 两个文件，`AddFactory`、`SubFactory` 一个字节都不用碰。

**结构**：

| 角色 | 职责 |
|---|---|
| Product | 抽象产品，声明产品共有的接口 |
| ConcreteProduct | 具体产品，各自实现 |
| Factory | 抽象工厂，只声明一个创建产品的接口方法 |
| ConcreteFactory | 具体工厂，决定实例化哪个具体产品 |
| Client | 只依赖 Product 和 Factory，不认识任何 ConcreteProduct |

```java
interface Operation { double apply(double a, double b); }

class Add implements Operation { public double apply(double a, double b) { return a + b; } }
class Sub implements Operation { public double apply(double a, double b) { return a - b; } }
class Pow implements Operation { public double apply(double a, double b) { return Math.pow(a, b); } }

interface OperationFactory { Operation create(); }          // 关键：实例化延迟到子类

class AddFactory implements OperationFactory { public Operation create() { return new Add(); } }
class SubFactory implements OperationFactory { public Operation create() { return new Sub(); } }
class PowFactory implements OperationFactory { public Operation create() { return new Pow(); } }

public class Client {
    private static double calc(OperationFactory f, double a, double b) {
        return f.create().apply(a, b);                      // 客户端全程不见具体类
    }
    public static void main(String[] args) {
        System.out.println(calc(new AddFactory(), 3, 4));   // 7.0
        System.out.println(calc(new PowFactory(), 2, 10));  // 1024.0
    }
}
```

**什么时候用**：

- 一个类无法预知自己该创建哪个类的实例，这个决定应该由子类做。
- 产品种类会持续增加，而框架/上层代码不应该跟着改。
- 想屏蔽创建过程的复杂度——构造参数多、需要组装、需要读配置（书中收银程序里，"先打折再满减"的工厂就把整条装饰链藏起来了）。
- 需要按环境切换产品族：测试环境一套工厂，生产环境一套。
- 判据很直接：**你发现自己在反复改同一个 switch/if-else 来 new 不同的类**。

**什么时候别用**：

- 产品固定就三五个、且明确不会扩展。此时工厂方法只会把 5 个类变成 10 个类，纯属自找麻烦，简单工厂足够。
- 工厂类里只有一句 `return new Xxx()`，且产品没有任何构造复杂度——这是把 `new` 换了个地方写，耦合只是转移，没有消除。

**易错点**：

- **一个产品一个工厂，类数量爆炸**。书里的做法值得记：不必拆到"加法工厂/减法工厂"，把稳定的产品归到一个"基础运算工厂"，新增的高级运算归到"高级运算工厂"。工厂的粒度按**产品族**分，不按单个产品分。
- **只搬了 new，没搬判断**。工厂接口是有了，可客户端还在 `if (type == 1) f = new AddFactory();`——switch 从工厂挪到了客户端，问题原封不动。
- **误以为工厂方法彻底消灭了修改**。新增一个产品族，还是得加一个工厂类、并且在上层某处选择它。工厂方法把"改"缩小到了"加"，但没做到零改动；真正要零改动得上反射或配置。

**和谁像、怎么区分**：

- **vs 简单工厂（第 1 章）**：简单工厂是**一个**工厂类 + switch，加产品必须改这个类，违反开放-封闭；工厂方法是**一个工厂接口 + N 个工厂类**，加产品只需加类。书中比喻：一个工厂是一维，多个工厂是二维。简单工厂的工厂类"知道的细节太多"，工厂方法把实例化**延迟到子类**。
- **vs 抽象工厂**：工厂方法一个工厂造**一种**产品；抽象工厂一个工厂造**一族**相关产品（比如同时造 CPU 和主板）。
- **vs 模板方法**：工厂方法其实就是模板方法的一个特例——父类定骨架，子类决定具体类型。

---
