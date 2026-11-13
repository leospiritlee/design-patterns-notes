# 第9章 简历复印——原型模式

**一句话**：复制一个现成对象，比重新 new 再逐字段赋值更快。

**故事**：小菜要去招聘会，印了厚厚一叠简历，连小学在哪读都写上，还标榜"Java 精通、C++ 精通、Python 精通"。大鸟嫌他浮夸，说起当年同学手抄简历、一份简历舍不得给人的年代，然后出题：写个简历类，要能设姓名、性别、年龄、工作经历，最后输出三份。

**不用它会怎样**：第一版老老实实 `new Resume("小菜")` 三次，每次再 `setSex`、`setAge`、`setWorkExperience`。三份还行，二十份就是二十遍初始化；更要命的是**改一个字要改二十处**——1998 年写成 1999 年，二十个地方挨个找。

于是想偷懒，写成 `Resume r2 = r1; Resume r3 = r1;`。这是传引用，三份"简历"其实是同一张纸的三个名字，改一份全变。纸面上写着"简历在 r1 那里"，没有实际内容。

真实场景里的代价更明显：简历对象往往是从数据库查出来、算了一堆统计、拼了排版信息才构造好的。每 new 一次就跑一遍构造函数，二十份就是二十次重复的昂贵初始化，而它们 95% 的字段完全一样。

**结构**：

| 角色 | 职责 |
|---|---|
| Prototype | 声明克隆自身的接口（Java 里就是 `Cloneable` + `clone()`） |
| ConcretePrototype | 实现克隆，返回自己的副本 |
| Client | 拿到一个原型实例后，靠 `clone()` 造新对象，不再 `new` |

```java
class WorkExperience implements Cloneable {          // 引用型成员，必须能自己克隆
    private String timeArea, company;
    public void set(String timeArea, String company) {
        this.timeArea = timeArea; this.company = company;
    }
    @Override public WorkExperience clone() {
        try { return (WorkExperience) super.clone(); }
        catch (CloneNotSupportedException e) { throw new AssertionError(e); }
    }
    @Override public String toString() { return timeArea + " " + company; }
}

class Resume implements Cloneable {
    private String name, sex, age;
    private WorkExperience work;

    public Resume(String name) { this.name = name; this.work = new WorkExperience(); }
    private Resume(WorkExperience work) { this.work = work.clone(); }   // 供深复制用

    public void setPersonal(String sex, String age) { this.sex = sex; this.age = age; }
    public void setWork(String timeArea, String company) { work.set(timeArea, company); }

    @Override public Resume clone() {                 // 深复制：引用对象也复制一份
        Resume obj = new Resume(this.work);
        obj.name = this.name; obj.sex = this.sex; obj.age = this.age;
        return obj;
    }

    @Override public String toString() { return name + " " + sex + " " + age + " | " + work; }
}

public class Client {
    public static void main(String[] args) {
        Resume a = new Resume("小菜");
        a.setPersonal("男", "23");
        a.setWork("2019-2023", "XX 公司");

        Resume b = a.clone();
        b.setWork("2023-2026", "YY 公司");      // 只改不一样的那一处

        System.out.println(a);   // 小菜 男 23 | 2019-2023 XX 公司
        System.out.println(b);   // 小菜 男 23 | 2023-2026 YY 公司
    }
}
```

把 `clone()` 里的 `new Resume(this.work)` 换成 `super.clone()`，就退回浅复制：三份简历的工作经历指向同一个 `WorkExperience`，最后设置的会覆盖前两次。

**什么时候用**：

- 对象的创建成本高：查库、算指标、解析文件、网络往返，而初始化结果基本不变。
- 新对象只是老对象改一两个字段，其余全同。
- 想避免调用方写一长串 setter，也避免它知道对象的构造细节。
- 需要保存某一时刻的快照（撤销、回滚、对比），复制比重建可靠。
- 大批量创建同类对象（循环里造一千个相似配置）。

**什么时候别用**：

- 对象就几个字段、构造是纯赋值，`new` 比 `clone` 更快也更清楚。
- 对象持有不可/不该复制的资源：数据库连接、文件句柄、线程、socket。复制出来的是同一份资源，迟早出事。
- 引用层级很深或存在循环引用，深复制写到第几层说不清——这种对象更适合重新构造。

**易错点**：

- **浅复制当深复制用**。这是本章的核心坑：`super.clone()` 对值类型逐位复制，对引用类型只复制引用。三个简历改同一个工作经历，后写的覆盖先写的，输出全是最后一份。
- **以为 String 是值类型所以安全**。String 确实是引用，只是不可变，所以"看起来"没出问题；换成 `Date`、`List`、自定义对象立刻暴露。
- **clone 不执行构造函数**。构造器里的初始化逻辑、`final` 字段的赋值都不会跑，依赖构造器做校验或注册的对象克隆出来是半成品。

**和谁像、怎么区分**：

- **vs 工厂方法**：都在造对象。工厂是**从无到有**（知道类型，按参数造），原型是**从有到有**（拿现成的复制），且不需要知道任何创建细节。
- **vs 拷贝构造函数**：目的一样，但原型靠多态——客户端拿着 `Prototype` 接口就能克隆，不必知道具体类型。

---
