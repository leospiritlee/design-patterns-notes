# 第1章 代码无错就是优？——简单工厂模式

**一句话**：把"该 new 哪个类"从调用方手里拿走，调用方只递一个符号。

**故事**：小菜去面试，题目是写一个计算器，他十分钟写完、自认为没 bug，却再没收到通知。大鸟点破：题目要的不是"能算对"，是"好维护、好复用、好扩展"。借曹操改诗、工匠重刻雕版的故事讲清四种特性，再从控制台计算器一步步逼出运算类和工厂。

**不用它会怎样**：一个 `calculate(a, b, op)` 方法里躺着 `switch`。要加平方根，改 `switch`；要加指数，再改 `switch`；要修加法的一个 bug，整个文件重新编译，加减乘除全部参与，手一滑把 `a + b` 写成 `a - b`，测试还未必发现。

书里举的薪资系统更狠：原本只有技术（月薪）、销售（底薪+提成）、经理（年薪+股份）三种算法，现在只加一个"兼职时薪"，却必须把装着三种算法的整个类交出去改。改的人顺手在月薪算法里加了一行"每月涨 10%"——本来只让你加功能，结果动到了跑得好好的老代码。这类事故的根源不是人不小心，是代码逼着人必须在同一处动刀。

**结构**：

| 角色 | 职责 |
|---|---|
| Operation（抽象产品） | 声明 `getResult(a, b)`，是所有运算的统一类型 |
| Add / Sub / Mul / Div（具体产品） | 各自实现一种运算，彼此不引用 |
| OperationFactory（工厂） | 按符号决定 new 哪个具体产品，返回抽象类型 |
| Client | 只依赖 Operation 和工厂，代码里不出现具体运算类名 |

**Java 实现**：

```java
abstract class Operation {
    public abstract double getResult(double a, double b);
}

class Add extends Operation {
    public double getResult(double a, double b) { return a + b; }
}

class Sub extends Operation {
    public double getResult(double a, double b) { return a - b; }
}

class Mul extends Operation {
    public double getResult(double a, double b) { return a * b; }
}

class Div extends Operation {
    public double getResult(double a, double b) {
        if (b == 0) throw new IllegalArgumentException("除数不能为 0");
        return a / b;
    }
}

class OperationFactory {
    public static Operation create(String op) {
        switch (op) {
            case "+": return new Add();
            case "-": return new Sub();
            case "*": return new Mul();
            case "/": return new Div();
            default:  throw new IllegalArgumentException("不支持的运算符: " + op);
        }
    }
}

public class Calculator {
    public static void main(String[] args) {
        Operation op = OperationFactory.create("/");
        System.out.println(op.getResult(10, 4));   // 2.5
    }
}
```

**什么时候用**：
- 产品种类少、且短期内基本不会增加（四则运算、几种固定格式的解析器）。
- 创建过程有共性动作，比如都要读配置、都要校验参数、都要设默认值。
- 调用方不该知道具体类名——换实现时不想改调用方代码。
- 产品对象需要集中管理生命周期或做缓存。

**什么时候别用**：
- 产品会持续新增。每加一种就要改工厂的 `switch`，等于把变化点集中到了一个必改的文件里，违反开放-封闭原则。
- 产品有多个变化维度（比如既要按运算类型、又要按精度策略），简单工厂会退化成嵌套 `switch`，该上工厂方法或抽象工厂。

**易错点**：
- 工厂里查不到就 `return null`，把 NPE 甩给调用方。应该抛异常或返回一个"空对象"。
- 把业务逻辑写进工厂。工厂只负责"造"，一掺计算就变成了上帝类。
- `switch` 漏 `default`，新增符号时静默返回 `null`，问题延后到运行时才爆。

**和谁像、怎么区分**：工厂方法模式——简单工厂是一个类管所有产品，加产品必改它；工厂方法是每个产品配一个工厂类，加产品只加类。抽象工厂则是"一族产品"，处理多个维度的组合。
