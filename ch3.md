# 第3章 单一职责原则

**一句话**：一个类只该有一个"老板"能命令它改。

**故事**：大鸟买了个电子阅读器，小菜问"手机不能看吗"。大鸟说手机不纯粹：能打电话、能听歌、能刷消息，读着读着被推送打断，永远进不了沉浸状态；电子阅读器除了看书什么都干不了，反而把这件事做到了极致。

**不用它会怎样**：所有代码写进 `Form1.java`——按钮事件、业务算法、SQL 语句、日志、格式校验全在一个类里。这时候任何一个需求来了，改的都是这个文件：改界面要动它，改算法要动它，换数据库还要动它。三个人同时改，天天冲突。

书里的俄罗斯方块最典型：下落、旋转、碰撞判断、移动、堆积、消层，加上绘制、擦除、键盘响应，全塞在窗体类里。功能没问题，可一旦要做 3D 版、Web 版，会发现一行都复用不了——游戏逻辑和 GDI+ 绘图缠在一起，只能整个复制过去再改。而其中真正不变的（下落、碰撞、消层）本来是最该被复用的部分。

**违反它会怎样**：

| 违反的表现 | 后果 |
|---|---|
| 算法、SQL、界面事件全写在窗体类里 | 换界面就得重写业务；业务无法被 Web 端、App 端复用 |
| 游戏逻辑和绘制代码混在一起 | 做 3D 版 / Web 版时零复用，只能整份复制 |
| 一个类同时管连接管理和业务计算 | 换数据库要改业务类，改业务又要重测数据库连接 |
| 类名带 `Manager`、`Util`、`Helper`，几十个 public 方法各管一摊 | 改 A 功能误伤 B 功能，回归测试范围无法收敛 |
| 一个方法里既校验参数又查库又拼报文 | 单测必须连数据库，mock 成本高于收益 |

**Java 实现**：

```java
// 职责一：游戏逻辑。只关心二维数组怎么变，完全不知道界面存在
class BlockBoard {
    private final int[][] grid = new int[10][20];   // 0 空，1 有方块

    public boolean canMoveLeft(int x, int y) {
        return x - 1 >= 0 && grid[x - 1][y] == 0;
    }

    public boolean canMoveDown(int x, int y) {
        return y + 1 < 20 && grid[x][y + 1] == 0;
    }

    public void land(int x, int y) { grid[x][y] = 1; }

    public int clearFullRows() {
        int cleared = 0;
        for (int y = 19; y >= 0; y--) {
            boolean full = true;
            for (int x = 0; x < 10; x++) {
                if (grid[x][y] == 0) { full = false; break; }
            }
            if (full) { shiftDown(y); cleared++; y++; }   // 补位后重查同一行
        }
        return cleared;
    }

    private void shiftDown(int row) {
        for (int y = row; y > 0; y--)
            for (int x = 0; x < 10; x++) grid[x][y] = grid[x][y - 1];
        for (int x = 0; x < 10; x++) grid[x][0] = 0;
    }

    public int cell(int x, int y) { return grid[x][y]; }
}

// 职责二：界面。只负责把数组画出来、把按键翻译成逻辑调用
class TetrisView {
    private final BlockBoard board = new BlockBoard();

    public void onLeftKey(int x, int y) {
        System.out.println(board.canMoveLeft(x, y) ? "左移一格" : "撞墙");
    }

    public void render() {
        for (int y = 0; y < 20; y++) {
            StringBuilder line = new StringBuilder();
            for (int x = 0; x < 10; x++) line.append(board.cell(x, y) == 1 ? "[]" : " .");
            System.out.println(line);
        }
    }
}

public class Tetris {
    public static void main(String[] args) {
        TetrisView view = new TetrisView();
        view.onLeftKey(0, 5);
        view.render();
    }
}
```

**什么时候用**：
- 你能想出多于一个动机去改这个类——比如"界面要改"和"算法要改"，就该拆。
- 类名里出现 `And`、`Manager`、`Util`、`Helper`，通常已经在兼职。
- 测试一个方法要 mock 一堆无关依赖。
- 同一个人在两个不相干的需求里都要改同一个文件。

**什么时候别用**：
- 一个几十行的脚本、一个一次性的数据清洗，拆成五个类只会让人找不着入口。
- 职责拆得比变化还细：类爆炸之后，读一次业务流程要跳十个文件，理解成本超过维护收益。

**易错点**：
- 把"职责"当成"方法"来拆——一个方法一个类，不叫单一职责，叫灾难。
- 类拆开了，但共享一个可变全局状态，等于没拆，耦合从编译期挪到了运行期。
- 界面和逻辑分开了，却让界面直接操作数组下标，逻辑类被绕过，规则散落两处。

**和谁像、怎么区分**：接口隔离原则——单一职责管"类为什么变"，接口隔离管"调用方被迫依赖了什么"，一个面向变化原因，一个面向使用方。
