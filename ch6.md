# 第6章 穿什么有这么重要？——装饰模式

**一句话**：给对象叠加功能时，别为每一种组合都写一个子类。

**故事**：小菜要去见娇娇，问大鸟穿什么。大鸟把嘻哈、白领、颓废、超人（内裤外穿）挨个调侃了一遍，最后说关键不在穿什么。话锋一转，让他写一个能自由换装的 Avatar 系统。

**不用它会怎样**：第一版把"穿大T恤""穿垮裤""穿破球鞋"写成 Person 类里的方法，客户端按顺序一个个调。功能是通的，可大鸟一说"再加个超人装扮"，就得动 Person 类——刚学的开放-封闭原则当场破功。

第二版改成继承：`Finery` 抽象类，`TShirt`、`Jeans`、`Sneakers` 各一个子类。看着挺面向对象，问题却更深：

1. **穿戴过程漏到客户端**。客户端得写 `ts.show(); jeans.show(); sneakers.show();`——相当于光着身子在众人面前一件件穿，而不是内部组装好再出场。
2. **组合是子类爆炸**。西装+T恤+披风、T恤+垮裤+皮鞋……每加一种搭配就要加一个类。想给同一个人上半身西装、下半身垮裤、左脚皮鞋右脚球鞋，继承体系根本表达不了。
3. **顺序改不了**。先穿内裤后穿外裤是凡人，反过来是超人。继承把顺序焊死在类里，客户端没有话语权。

需求真正变的时刻：周年庆要"打8折再满300返100"，下周又要"满200送50再打7折"。按继承的路子，得写 `CashReturnRebate`，把 `CashReturn` 和 `CashRebate` 的代码合并着重抄一遍；再来"返现后再打折"，再抄一遍；再送积分、抽奖、小礼品……组合一多，类就数不清了。

**结构**：

| 角色 | 职责 |
|---|---|
| Component | 抽象接口，声明被装饰对象和装饰者共有的方法，客户端只认它 |
| ConcreteComponent | 被装饰的核心对象，只做最基本的事，不知道自己会被装饰 |
| Decorator | 抽象装饰类，持有 Component 引用，并把请求转发给它 |
| ConcreteDecorator | 具体装饰，在转发前后加上自己的那一点功能 |

关键点：装饰者**继承** Component（为了类型一致，可以替换），同时**持有** Component（为了包装）。继承拿类型，组合拿行为。

```java
interface Person { void show(); }

class Student implements Person {                 // ConcreteComponent
    public void show() { System.out.print("小菜"); }
}

abstract class Finery implements Person {         // Decorator
    protected Person component;
    public Finery(Person component) { this.component = component; }
    public void show() { component.show(); }      // 转发
}

class TShirt extends Finery {                     // ConcreteDecorator
    public TShirt(Person c) { super(c); }
    public void show() { System.out.print("大T恤 "); super.show(); }
}

class Jeans extends Finery {
    public Jeans(Person c) { super(c); }
    public void show() { System.out.print("垮裤 "); super.show(); }
}

class Sneakers extends Finery {
    public Sneakers(Person c) { super(c); }
    public void show() { System.out.print("破球鞋 "); super.show(); }
}

public class Client {
    public static void main(String[] args) {
        Person xc = new Student();
        xc = new TShirt(xc);       // 一层层包起来
        xc = new Jeans(xc);
        xc = new Sneakers(xc);
        xc.show();                 // 破球鞋 垮裤 大T恤 小菜
    }
}
```

加草帽只需新增一个 `StrawHat extends Finery`，再在客户端挑个位置包上去，其余代码一行不动。

**什么时候用**：

- 需要给一批对象**在运行时**加功能，而且加不加、加几层要由运行时决定。
- 功能可以自由组合，组合数远大于功能数（5 种功能就有几十种搭配）。
- 原类不能改：第三方库、已上线且稳定的核心类。
- 想给某个对象单独加功能，而不是给整个类加（继承只能整类生效）。
- 装饰可以撤销——把外层剥掉就退回原样，继承做不到。

**什么时候别用**：

- 组合方式固定且只有两三种，直接写两个子类比搭一条装饰链清楚得多。
- 装饰顺序敏感、又没人能保证调用方按对的顺序包，链条会变成定时炸弹。

**易错点**：

- **忘了转发**。ConcreteDecorator 里只写自己的逻辑、不调 `super.show()`，链条从这里断掉，后面的装饰全部失效。这是最常见的一种。
- **顺序被当成无关紧要**。加密和敏感词过滤都是持久化前的装饰，先加密再过滤，过滤就再也匹配不上了。作者的原话：理想情况是装饰类彼此独立，才能任意顺序组合。
- **身份判断失效**。装饰后对象的运行时类型是 `TShirt` 而不是 `Student`，`instanceof Student` 一律为 false；`equals`/`hashCode` 不转发也会出问题。

**和谁像、怎么区分**：

- **vs 代理**：结构几乎一样，都是"同接口 + 持有一个同类型对象 + 转发"。区别在**谁决定、为什么**：装饰由客户端一层层组装，目的是**加功能**，可以套任意多层；代理自己创建并持有真实对象，客户端不知道有真实对象存在，目的是**控制访问**，通常只有一层。
- **vs 建造者**：建造者要求建造过程稳定（先加壳再加轮子），装饰的组装过程完全不固定，顺序由客户端说了算。
- **vs 策略**：策略是换掉整个算法，装饰是在原算法外面加东西。

---
