### 一、基本概念

**状态 state**

* 定义（s）：agent所处的环境位置（eg：棋盘格每一格对应一个s）
* 状态空间：s的集合，数组

**动作 action**

* 定义（a）：每个状态时所有可能的行动（eg：棋盘格上下左右走）
* 动作空间：a的集合

**状态转移 state-transition**

* 定义：s —（通过某个a）—> s‘，结果由人为定义 或 物理规则决定
* 表示：条件概率，p (s‘ | s, a) = 1

**策略 policy**

* 定义：决定agent在一个state下该如何采取action
* 表示：
  * 数学：条件概率，pi (a1 | s1) = 0，pi (a1 | s2) = 1，...
  * 实践：二维数组存放所有情况的概率表达一个 policy

**reward**

* 定义（r）：设计采取每个action后得到的分数，结合action执行前后的状态
* 表示：条件概率，p (r | s, a) = p，reward为r的概率

**trajectory**

* 定义：state - action - reward链，sn —(ax, r=x)—> sn+1

**return**

* 定义：每个 trajectory 所有 reward 之和
* 作用：借用return评估策略好坏

**discounted return**

* 定义：给定折扣率gamma，discounted return = sigma gamma^n-1 * reward
* 作用：
  * 对于不停止的策略，有无穷长trajectory，将return结果收敛
  * 控制agent远近偏好，gamma趋近于0偏好更近reward，趋近于1偏好更远reward （eg：无穷个1的reward+1个无穷远处的♾️reward）

**episode**

* 定义：一个有限长任务到达终点的trajectory
* episodic tasks 与 continuing tasks转换：target state限制状态空间 / 改reward

**马尔可夫决策过程 MDP**

* 马尔可夫性质：状态转移、reward与历史无关；策略（最优）也与历史无关（上述定义就是与历史无关，数学推论证明这是最优策略）
* 决策：policy
* 过程：条件概率，状态转移、reward

___

### 二、贝尔曼公式

从某个s起点的总discounted return可以借用其他state起点的discounted return来求解

**state value**

* 定义：Gt为一个trajectory的discounted return，state value vpi(s) 为Gt的期望
* 意义 ：评估一个特定策略下，每个状态的价值。结合初始状态分布，可以评估策略好坏。
* return 与 state value：return针对单一条trajectory，state value针对单一起点可能的多条trajectory的期望

**贝尔曼公式**

* 干什么：数学上求state value vπ(s) 
* 三层随机性质
  * 策略结果：给出的策略后采取a有概率
  * 环境模型：给出a本身s的转移也有概率、不同a/s->s给出的reward也有概率
* 推导
  * Gt = Rt+1 + \gamma * Gt+1，直观理解vpi(s) = 所有可能下的求和 Rt+1 + \gamma * vpi(s')
  * 1带入2变成vpi(s)的递推求解式vπ(s)=a∑π(a∣s)s′,r∑p(s′,r∣s,a)[r+γvπ(s′)]，数学上联立求解每一个vpi(s)
* 矩阵向量形式：n个联立求解

**action value**

* 定义：在s下采取特定的a后得到的分数期望，qpi(s, a) = E[Gt | s, a]
* 计算：通过state value计算；不计算state value计算（通过数据，不依赖模型的方法）
* 意义：对比当前策略和其他选择的分数，改进策略。哪怕后续策略都不是最优，反复迭代完仍是最优。
* state value 与 action value：某个s下所有a value求平均即为s value

___

### 三、贝尔曼最优公式

引入：通过action value贪心更新策略，如果后续策略都不是最优，为什么迭代后仍然能得到整体最优？

**最优策略**

* 定义：当前策略所有state的state value都大于其他策略，则称最优
* 性质：特定reward下，一定存在一个确定性的最优策略）；贪心（每个s处贪心，一定是最优）；reward变为f(reawrd)后，最优策略不变；gamma决定最优策略一定是相对最短路

**贝尔曼最优公式**

* 定义：最优策略max pi时的贝尔曼公式，此时所有state value均为最大，需要在求解中同时求最优策略。
  * 这里同步求解的策略本质就是对s下每个a的概率的系数
* 最优策略求解：
  * 若已知最优v（state value），贪心，当前state下选最大state value方向的概率为1，如果不唯一则在最大中随意分配。dp证明。
* 最优状态值求解：
  * 对于贝尔曼最优，有V=f(V)：一组正确的最优状态价值，经过一次贝尔曼更新后，应当保持完全不变。
  * contraction mapping theorem：
    * 称|x2 - x1| >= |f(x2) - f(x1)|为contraction mapping，一定唯一存在不动点f(x) = x
    * 对于每个contraction mapping，可以通过迭代求解不动点：随便初始化x，将f(x)作为下一轮x迭代
  * 贝尔曼最优中的V=f(V)是contraction mapping，若已知最优策略，可以通过上述迭代求所有state value
* 如何二者耦合迭代，详见后续第四章

___

### 四、值迭代与策略迭代
