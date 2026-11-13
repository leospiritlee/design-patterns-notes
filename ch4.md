# 第4章 开放-封闭原则

**一句话**：新需求来了只加代码，不改已经跑通的代码。

**故事**：小菜考研英语差两分，又因为只顾复习没找工作，两头落空。大鸟拿"一国两制"和弹性工作制打比方：社会主义制度不能动（对修改关闭），但可以加一种制度（对扩展开放）；八小时坐班不能动，但可以加弹性时段、加月度迟到额度。作者自己也承认这个类比有点牵强。

**不用它会怎样**：`calculate` 里一个 `switch`，每加一种运算就改一次。第一次改，你记得住影响面；第十次改，`switch` 已经八十行，改的人不敢确定有没有碰到别的分支，于是每次都全量回归。更隐蔽的是"临时补丁"：为了赶活动，在 `if` 里加一个 `&& type != 3`，半年后没人敢删这个条件。

真正让人难受的不是"改了"，而是"为了加一个功能，必须打开一个装着十个功能的文件"。

**违反它会怎样**：

| 违反的表现 | 后果 |
|---|---|
| 新运算 / 新促销都改同一个 `switch` | 加减乘除全部重新编译，误改风险与分支数成正比 |
| 用类型码 + `if-else` 表达业务分类 | 分支越多越不敢动，新需求只能继续加分支 |
| 抽象类里塞进具体实现的细节 | 接口一改，所有实现类跟着改，封闭性名存实亡 |
| 变化还没发生就为"以后可能"提前抽象 | 抽象方向猜错，比不抽象更难改，代码里多出一层空壳 |
| 只在客户端加判断，不动领域模型 | 客户端成了新的变化集中点，等于把 `switch` 搬了个家 |

**Java 实现**：

```java
interface Exporter {                       // 稳定抽象：新增格式不动这里
    String export(String[][] rows);
}

class CsvExporter implements Exporter {
    public String export(String[][] rows) {
        StringBuilder sb = new StringBuilder();
        for (String[] r : rows) sb.append(String.join(",", r)).append("\n");
        return sb.toString();
    }
}

class JsonExporter implements Exporter {
    public String export(String[][] rows) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < rows.length; i++) {
            sb.append("{\"region\":\"").append(rows[i][0])
              .append("\",\"amount\":").append(rows[i][1]).append("}");
            if (i < rows.length - 1) sb.append(",");
        }
        return sb.append("]").toString();
    }
}

// 新增 XmlExporter 时只加这一个类，下面的 ReportService 一行不改
class XmlExporter implements Exporter {
    public String export(String[][] rows) {
        StringBuilder sb = new StringBuilder("<report>");
        for (String[] r : rows)
            sb.append("<row><region>").append(r[0])
              .append("</region><amount>").append(r[1]).append("</amount></row>");
        return sb.append("</report>").toString();
    }
}

class ReportService {                      // 高层：只依赖 Exporter
    private final Exporter exporter;
    public ReportService(Exporter exporter) { this.exporter = exporter; }
    public String build(String[][] data) { return exporter.export(data); }
}

public class ReportDemo {
    public static void main(String[] args) {
        String[][] data = { {"华东", "1200"}, {"华南", "980"} };
        System.out.println(new ReportService(new CsvExporter()).build(data));
        System.out.println(new ReportService(new JsonExporter()).build(data));
        System.out.println(new ReportService(new XmlExporter()).build(data));
    }
}
```

**什么时候用**：
- 同一处已经被改过两次以上，第三次还会来——这时的抽象是有依据的，不是猜的。
- 变化维度清晰可枚举（导出格式、支付渠道、通知方式），且能用一句话描述扩展点。
- 已有代码被多个调用方共用，改它的代价远高于加一个新类。

**什么时候别用**：
- 变化还没发生就抽象。作者说得很直白：拒绝不成熟的抽象和抽象本身一样重要。
- 只有一处 `if`、且业务上确定不会增长。为它造接口、工厂、注册表，是把简单问题复杂化。
- 抽象成本高于改动成本时（比如这个类全项目只有一处调用，且半年才动一次）。

**易错点**：
- 把"对修改封闭"理解成"一个字都不能改"。抽象本身是要改的，封闭的是已稳定的抽象之上的扩展点。
- 抽象泄漏：接口方法签名里带上具体实现才有的类型（如 `export(ResultSet rs)`），换实现时接口就崩了。
- 只做到"客户端不加 `if`"，却让工厂类承担了全部变化，问题只是被搬到了工厂。

**和谁像、怎么区分**：依赖倒转原则——开放-封闭是目标（加代码不改代码），依赖倒转是达成它的手段（双方都依赖抽象）。
