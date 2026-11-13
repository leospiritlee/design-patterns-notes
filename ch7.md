# 第7章 为别人做嫁衣——代理模式

**一句话**：客户端不必认识真正的对象，也能用它的功能。

**故事**：娇娇讲高中往事：同班的戴励一次次替隔壁班的卓贾易送娃娃、送花、送巧克力。小菜听得唏嘘，大鸟一句话点破——卓贾易才是追求者，戴励是代理。后来看《武林外传》，吕秀才让燕小六代其向郭芙蓉求婚，两人异口同声："代理模式。"

**不用它会怎样**：第一版很直白，客户端 `new SchoolGirl("娇娇")`，再 `new Pursuit(mm)`，然后 `gg.giveDolls()`。问题立刻出现：**被追求者必须认识追求者**。可娇娇根本不认识卓贾易，这段代码在现实里不成立——它把两个本该无关的类硬绑在了一起。

第二版"修"了一下：把 `Pursuit` 直接改名成 `Proxy`，把卓贾易换成戴励。这下更错——礼物变成戴励送的了，卓贾易这个人凭空消失，业务逻辑整个歪掉。

真正的痛点在这里：送礼物这件事有**三个**参与者，追求者买东西、代理递东西、被追求者收东西。少一层抽象，就只能二选一：要么被追求者认识追求者（不现实），要么追求者不存在（不成立）。

**结构**：

| 角色 | 职责 |
|---|---|
| Subject | 真实对象和代理的共用接口，客户端只依赖它 |
| RealSubject | 真正干活的实体，代理所代表的对象 |
| Proxy | 持有真实对象的引用，实现同一接口，把请求转给它，并可自行加前后处理 |
| Client | 面向 Subject 编程，不知道背后是代理还是实体 |

```java
interface Subject {
    void giveDolls();
    void giveFlowers();
    void giveChocolate();
}

class SchoolGirl {
    private final String name;
    public SchoolGirl(String name) { this.name = name; }
    public String getName() { return name; }
}

class Pursuit implements Subject {                // RealSubject
    private final SchoolGirl mm;
    public Pursuit(SchoolGirl mm) { this.mm = mm; }
    public void giveDolls()     { System.out.println(mm.getName() + "，送你洋娃娃"); }
    public void giveFlowers()   { System.out.println(mm.getName() + "，送你鲜花"); }
    public void giveChocolate() { System.out.println(mm.getName() + "，送你巧克力"); }
}

class Proxy implements Subject {                  // 唯一同时认识双方的人
    private final Pursuit gg;
    public Proxy(SchoolGirl mm) { this.gg = new Pursuit(mm); }
    public void giveDolls()     { gg.giveDolls(); }
    public void giveFlowers()   { gg.giveFlowers(); }
    public void giveChocolate() { gg.giveChocolate(); }
}

public class Client {
    public static void main(String[] args) {
        Subject proxy = new Proxy(new SchoolGirl("娇娇"));
        proxy.giveDolls();       // 客户端以为在跟"追求者"打交道
        proxy.giveFlowers();
    }
}
```

代理能插话的地方就在那几个转发方法里：加日志、加权限校验、加缓存、延迟初始化——都可以，客户端一行不用改。

**什么时候用**：

- **远程代理**：对象在另一台机器/另一个进程，本地放个替身，把网络调用伪装成本地调用（RPC stub、WebService 的 wsdl 生成物）。
- **虚拟代理**：对象创建代价极大，先用轻量替身占位，真要用时再加载（大图占位框、懒加载的实体）。
- **安全代理**：调用前检查权限，不同角色给不同的代理。
- **智能指引**：调用前后做内务——引用计数、加锁检查、访问日志、首次访问时装载。
- 一句话判据：你想在**不改变客户端代码**的前提下，控制"能不能访问、什么时候访问、访问时顺带做什么"。

**什么时候别用**：

- 只是想加日志/计时这类横切逻辑，切面（AOP）或装饰器更合适，不必为每个方法手写一遍转发。
- 代理里除了 `return real.xxx()` 什么都没有，且没有加任何控制的打算——这层就是纯开销。

**易错点**：

- **代理和真实对象接口不同步**。给 `Subject` 加了新方法，只改了 `Pursuit` 忘了 `Proxy`，编译期报错还是好的，运行时漏功能才要命。
- **每次调用都 new 一个真实对象**。代理的意义之一就是复用/控制那个实体，在方法里反复 `new Pursuit(mm)` 等于白代理。
- **客户端同时持有代理和实体**。`new Proxy(mm)` 之后又自己 `new Pursuit(mm)` 直接用，控制就绕过去了，代理形同虚设。

**和谁像、怎么区分**：

- **vs 装饰模式**：见第 6 章。一句话——装饰是"我给你加东西，你自己组装"，代理是"我替它出面，你别管它"。
- **vs 适配器**：适配器**换接口**（把 A 的接口转成 B 期望的样子），代理**保持同一接口**。
- **vs 外观（Facade）**：外观是给一群对象包一个简化入口，代理是给**一个**对象做替身。

---
