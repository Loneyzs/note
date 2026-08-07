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

* 定义（r）：设计采取每个action后得到的分数，人设计结果规则，具体概率分布跟随状态转移的概率分布
* 表示：条件概率，p (r | s, a) = p，reward为r的概率

**trajectory**

* 定义：state - action - reward链，sn —(ax, r=x)—> sn+1

**return**

* 定义：每个 trajectory 所有 reward 之和
* 作用：借用return评估策略好坏

**discounted return**

* 定义：给定折扣率γ，discounted return = ∑ γ^n-1 * reward
* 作用：
  * 对于不停止的策略，有无穷长trajectory，将return结果收敛
  * 控制agent远近偏好，γ趋近于0偏好更近reward，趋近于1偏好更远reward （eg：无穷个1的reward+1个无穷远处的♾️reward）

**episode**

* 定义：一个有限长任务到达终点的trajectory
* episodic tasks 与 continuing tasks转换：target state限制状态空间 / 改reward

**马尔可夫决策过程 MDP**

* 马尔可夫性质：状态转移、reward与历史无关；策略（最优）也与历史无关（上述定义就是与历史无关，数学推论证明这是最优策略）
* 决策：policy
* 过程：条件概率，状态转移、reward

___

### 二、贝尔曼公式

干什么：对于特定策略和每个state，计算每一个state value (v)

从某个s起点的总discounted return可以借用其他state起点的discounted return来求解

**state value**

* 定义：Gt为一个trajectory的discounted return，state value vpi(s) 为Gt的期望
* 意义 ：评估一个特定策略下，每个状态的价值。结合初始状态分布，可以评估策略好坏。
* return 与 state value：return针对单一条trajectory，state value针对单一起点可能的多条trajectory的期望

**贝尔曼公式**

* 三层随机性质
  * 策略结果：给出的策略后采取a有概率
  * 环境模型：给出a本身s的转移也有概率、不同a/s->s给出的reward也有概率
* 推导
  * Gt = Rt+1 + \γ * Gt+1，直观理解vpi(s) = 所有可能下的求和 Rt+1 + γ * vpi(s')
  * 1带入2变成vpi(s)的递推求解式vπ(s)=a∑π(a∣s)s′,r∑p(s′,r∣s,a)[r+γvπ(s′)]，数学上联立求解每一个vpi(s)
* 矩阵向量形式：n个联立求解
* 迭代算法：借用contraction mapping theorem结论，vk开始迭代，迭代无穷词一定逼近真实vk。遍历所有s各自作为起点，迭代完得到V。
  * 策略不随内部迭代出的V而改变。

**action value**

* 定义：在s下采取特定的a后得到的分数期望，qpi(s, a) = E[Gt | s, a]
* 计算：通过state value计算；不计算state value计算（通过数据，不依赖模型的方法）
* 意义：对比当前策略和其他选择的分数，改进策略。哪怕后续策略都不是最优，反复迭代完仍是最优。
* state value 与 action value：某个s下所有a value求平均即为s value

___

### 三、贝尔曼最优公式

干什么：计算最优下策略下的state value

引入：通过action value贪心更新策略，如果后续策略都不是最优，为什么迭代后仍然能得到整体最优？

**最优策略**

* 定义：当前策略所有state的state value都大于其他策略，则称最优。最优策略与最优状态值对应。
* 性质：特定reward下，一定存在一个确定性的最优策略；贪心（每个s处贪心，一定是最优）；reward变为f(reawrd)后，最优策略不变；γ决定最优策略一定是相对最短路

**贝尔曼最优公式**

* 定义：最优策略max pi时的贝尔曼公式，此时所有state value均为最大。固定内部更新V时每轮都贪心，理解为评估同时寻找最优策略。
  * 这里同步求解的策略本质就是对s下每个a的概率的系数
* 最优策略求解：
  * 若已知最优v（state value），贪心，当前state下选最大state value方向的概率为1，如果不唯一则在最大中随意分配。dp证明。
* 最优状态值求解：
  * 对于贝尔曼最优，有V=f(V)：一组正确的最优状态价值，经过一次贝尔曼更新后，应当保持完全不变。
  * contraction mapping theorem：
    * 称|x2 - x1| >= |f(x2) - f(x1)|为contraction mapping，一定唯一存在不动点f(x) = x
    * 对于每个contraction mapping，可以通过迭代求解不动点：随便初始化x，将f(x)作为下一轮x迭代
  * 贝尔曼最优算子（max pi）恒为contraction mapping，对其迭代一定会收敛到最优状态值。
  * 策略随contraction mapping theorem内部迭代出的值变化, 理解为每一次更新只考虑下一步最优后的V
* 本质是先求最优状态值, 再通过结果一步反推最优策略

___

### 四、值迭代与策略迭代

做什么：求最优策略。model-base，dp方法。

本质上,三者中值迭代是特殊的,他通过贝尔曼最优公式做到了同步更新策略

**值迭代**

* 定义 / 证明：一层循环。给定任意状态值，通过贝尔曼最优算子迭代最优状态值，状态值收敛后，贪心即为最优策略。理解为每轮加入一层贪心真实奖励，最终初值影响归零。
  * max贪心有两含义：一是组成贝尔曼最优算子中间过程，每一次迭代时贪心选择最优实际奖励；二是最终贪心为最优策略。两部分有各自证明方式，并不完全相同。
* 迭代过程
  * 值更新：Vk带入贝尔曼最优算子->Vk+1，通过contraction mapping theorem迭代更新值，内部迭代中策略随V变化，一直贪心。
  * 策略更新：所有轮迭代完，用最优状态值贪心确定最优策略

**策略迭代**

* 定义 / 证明：两层循环。给定任意策略，内层通过贝尔曼公式迭代计算所有state value，外层对每个state贪心一步得到新策略。最终可证明唯一且正确。
* 迭代过程
  * 策略评估：给定任意策略，使用旧策略通过贝尔曼方程迭代法更新state value进行评估，内部迭代中策略保持外部循环上一次更新的不变。
  * 策略改进：外部每次循环完，通过贪心，改进每个state下当前的策略。

**截断策略迭代**

* 定义：值迭代内层循环，固定更新m次，不等到完全收敛，直接进行外层下一轮。内层变为1则为值迭代，内层变为无穷则为策略迭代

___

### 五、蒙特卡洛（控制）方法 

做什么：策略迭代的基础上替换模型部分变成无模型。解决未知状态转移的概率分布，进一步影响r结果未知。

对于环境来说，模型和数据总得有一个。model-free的一个问题是没法自由选择起点->没法自由遍历。

**蒙特卡洛估计**

* mean estimation：多次实验采样取平均，近似模型的概率分布。大数定理。

**MC-basic RL**

* 核心：在策略迭代框架上。v = sigmma pi(s, a) * qpi(s, a)（行动*行动价值），其中qpi(s, a) = E[Gt | s, a]，不展开Gt，用原始实验定义求qpi
* 具体操作：策略评估中更新每个点更新状态值。对每个a（s, a）出发，pi策略，设置episode递归叠加立即reward得到多组（状态转移是概率的，单独一组不合理）采样g（s， a），求平均即为 E[Gt | s, a]，p(a, s) * 每个qpi(s, a)得到状态值。采样通过大的物理引擎交互得到。策略更新不变（贪心）。大数定理、均值估计。
* 理解：对于无模型的策略评估，采用bellman公式没展开之前的形式，通过转换成Q来估计近似。

**MC Exploring Starts**

* visit：定义每个（s, a）为一个visit。策略评估工作变为遍历每个visit，得到其为起点的action value。
* MC-basic问题：在上述核心思想里，每个visit产生的episode链，包含了其他visit为起点的episode。
* 优化思路：广义截断策略迭代，每轮策略改进只随机选一组（s, a）。得到完整episode链后，递归回来给这条链上每个（s, a）添加一组估计值。每轮维护这一组更新的采样，直接更新策略。探索由随即起点保证，直观认为所有visit都可能被更新到所以正确。无证明无条件收敛。

**MC epsilon-greedy**

* epsilon-greedy policy：π(a∣s)={1−ε+∣A∣ε,∣A∣ε,a=argmaxaQ(s,a)其他动作，贪心和其他策略各自概率，平衡了样本q下探索和剥削。因为走过的（s,a）可能样本少，和实际Q有区别。实际Q已知则直接贪心。
* Exploring Starts的问题：每轮要求能把环境重置到任意（s,a），有些仿真或者现实世界收集数据做不到。
* 优化思路：epsilon-greedy policy，每轮更新一条episode的q后策略自然改变。每轮从环境正常起点开始，探索和epsilon、episode有关，有随机性不会停在终点。使用时epsilon应逐渐减小。无证明无条件收敛。

初次之外还有蒙特卡洛V估计方法：按照当前pi走完episode，递归叠加回每个s起点的V

___

### 六、随机近似与梯度下降（TD的前置理论基础）

#### 6.1随机近似理论

干什么：用单个有噪声的观测，滚动更新均值估计

**incremental mean estimation**

* 滚动更新的均值估计：迭代wk+1=wk - αt * (wk - xk)，当αt=1/k为平均，或其他值时逼近E

**Robbins-Monro algorithm**

* 问题：求解g(w*) = 0，且g未知（类比神经网络）
* 迭代算法：设当前最优解为wk+1，wk+1 = wk - ak * g~(wk, nk)，其中g~ = g(wk) + nk即有噪声的观测，ak为每轮迭代的参数。
  * 条件：梯度>0曲线递增且有上界、ak和=♾️且平方和<♾️（即ak趋于0但不要太快）、n有分布条件。
  * ak选择：常见1/k，实际选择赋一个非常小的常数。也有其他选择。
* 应用：证明上述滚动更新的均值估计收敛。

#### 6.2随机梯度下降理论（SGD）

干什么：用单个有噪声的观测，更新估计函数的梯度至极小值点

**SGD算法**

* 问题：J(w)=E[f(w, X)]，优化w使得J最小，且f未知。含有随机变量的函数期望的优化问题。
  * 下述三者都是采样值带入得到一个期望函数，对这个函数求梯度得最小值。
* 梯度下降（GD）：迭代wk+1 = wk − αk∇wE[ f (wk,X) ] = wk − αkE[∇w f (wk,X) ]。不适用未知 f 的情况。普通DL中学习率的情况。
* 批量梯度下降（BGD）：E[∇w f (wk,X) ] = 1/n * ∑∇w f (wk, xi)，n个采样估计期望，估计好但采样多
* 小批量梯度下降（MBGD）：E[∇w f (wk,X) ] = 1/m * ∑∇w f (wk, xi)，一部分采样估计期望，性能和采样适中
* 随机梯度下降（SGD）：wk+1 = wk − αk ∇w f(wk,xk) ，只用单个采样估计期望，采样少但性能不好
  * 收敛性及条件：依赖Robbins-Monro证明，同Robbins-Monro条件。
  * 应用：未知 f 的情况下更新参数

___

### 七、时序差分方法（TD）

做什么：model-free且即时迭代

**TD learning of states value (prediction) **

* 作用：自举迭代更新v。通过bellman方程求解给定策略pi的state value，评估固定策略能访问到的S的V。
* 算法：定义vt(s)为s在t时刻的V的估计值，vt+1(st) = vt(st) + αt(st)[rt+1 + γ*vt(st+1) - vt(st)]。全部随机初始化后。按环境的初始状态分布得到起点开始迭代，由于环境依赖接口限制，不能全局指定起点遍历S。迭代中维护全局V，episode中遇到的S也会开始更新。
* 理解：极其类似Rt+1 + γ * vpi(s')。只是证明如果模型未知，用采样的可能有误差的r和s'仍能收敛V。
  * 数学证明了无模型下bellman用采样值求解仍然成立，只是实际迭代方式有区别（s转移的随机性）。
  * 同底层Rt+1 + γ * vpi(s')，每次新的采样信息通过 r 和当前预测的 V(s') 进入迭代。
* 应用求最优策略：普通策略迭代框架，代替普通bellman进行V更新。

**Sarsa **

* 作用：Qπ的Bellman期望方程解给定策略qpi的action value，与TD平行。如果用ε-greedy则同步更新策略。
* 算法：把TD所有V换成Q。qt+1(st, at) = qt(st, at) - αt(st, at)[qt(st. at) - [rt+1 + γ*qt(st+1, at+1)]]。
* 理解：和TD完全对应，有Qπ 的 Bellman 期望方程作为随机变量侧的支撑。
* 应用求最优策略：如果用ε-greedy则在Q更新时同步更新策略（ε-greedy的性质），此时迭代后也是直接的贝尔曼最优（对应同时更新出来最优策略）。不用则纯做prediction。
* Expecded Sarsa && n-step Sarsa：通过估计求解形式再变化的bellman公式，Gt分解方式不同变MC/TD
  * Online 和 offline：能否训练和收集数据同步
* 应用：本质是prediction，用 ϵ-greedy 后也可以作为control

**Q-learning**

* 作用：直接估计最优action value
* 算法：sarsa的TD target变max，下一步主动选择一个max的a走，而不是根据原策略。qt+1(st, at) = qt(st, at) - αt(st, at)[qt(st. at) - [rt+1 + γ* max qt(st+1, ax)]]。在数学上证明直接用估计值解贝尔曼最优。
* 性质on-policy和off-policy：behavior policy和target policy相同是为on-policy，（可以）不同为off-policy。此处为off。本质是采集数据时是否直接同步更新策略。揭示了model-free情况下，最好有探索性比较强的策略作为数据收集的策略。
* 应用：获得最优Q后完整贪心即为最优策略。这里max只是选择一个方向更新，不代表一定会向max方向走。区别于值迭代的能全局遍历，这里受环境起点限制常用ε-greedy作为onpolicy保证探索。也可以维护两个pi，一个探索收集数据，一个用max更新。

___

### 八、值函数近似

做什么：策略评估过程，给定 π，用函数/神经网络 v_hat(s,w) 去近似 vπ(s)

**MC / TD with 值函数近似（basic）**

* 算法：定义损失函数为均方误差，用于更新参数训练目标预测函数q^。J(w) = E[(vπ(S) - v_hat(S,w)) ^ 2] = ∑μ(s) * [vπ(s) − v_hat(s,w)) ^ 2]，S是随机变量，用采样近似E。
  * 问题：连续情况下每个V更新w都会影响所有V，不能像离散情况下遇到V就更新，需要有策略地更新。
  * 解决：用采样逼近E(E(S) = ∑μ(s) * s)，定义上人工选取合适的μ规定采样权重，让更重要的采样多出现。同时，实际用采样逼近的过程中μ隐含在采样出现的频率中，而通常用off-policy采样频率满足采样b策略而非此处定义，则人工加一个修正系数修正策略产生的数据分布差异，来满足这里的μ。
  * 常用分布策略：stationary distribution，一种平稳的分布权重策略，具体不展开。
* 优化（迭代过程）：更新w参数。GD展开后E中有未知f，换SGD用采样估计梯度下降。展开后得wt+1 = wt + αt * [vπ(st) − v^(st,wt)]∇v^(st,wt)迭代方程。
  * 问题：vπ(st)不知道
  * 解决：MC走episode递归得到每一个visit的真实vpi，且每一个visit可以独立更新w；TD自举迭代vpi。这两种方法解决后即为MC / TD的值函数近似完整思路。
* 性质：存储空间从所有V变成w；泛化能力优秀，一次采样就能同步更新所有V的逼近

**sarsa with 值函数近似**

* 算法：完全类比Q-learning，去掉采样的max即可。

**Q-learning with 值函数近似**

* 算法：
  1. 从均值方差出发，SGD展开+对应离散下一步a永远贪心，优化为wt+1 = wt + αt * [rt+1 + γ*maxq^(st+1, at+1, wt) − q^(st,at,wt)]∇q^(st, at, wt)底层形式。
  2. 构造新损失函数J(w) = E[(R + γ * maxq^(S', a, w) - q^(S, A, w)) ^ 2] 在J内部就用TD来解决新采样，且SGD展开时冻结q*(s')的梯度（更新梯度时新采样的估计作为常数），最终得到和上述一致的底层结果。这里额外引入一个特殊化的激活函数是为了适配NN框架，接口填函数进去。
* 应用：完整获得最优Q后全局贪心即为最优策略，on / off-policy均可

**Deep Q-learning（DQN）**

* 算法：在Q-learning with 值函数近似基础上进行了优化
  * 两个网络：维护target network采样用，定期用main network（一直维护）更新但频率低。保证稳定性
  * experience replay：维护一个定长的replay buffer，每一步采样加入，训练时从中按照一定分布mini batch。最终采样被用到的分布同时取决于采样出的分布 + 选取的分布，这里常用均匀分布抽取，维持采样出现的原有分布，最终会逼近整体的stationary distribution。
* 应用：同上。具体训练质量与episode设置（数据量）有关

___

### 九、策略梯度方法

做什么：policy-based，直接建立一个策略目标函数，优化目标函数直接得到最优策略

**metrics**

* 策略函数：定义pi(a|s, θ)，输入s输出a参数θ。
* 目标函数 / metrics：
  * 理解：全局最优π依然存在，但NN压缩策略空间可能表达不了，不再严格有“压过所有pi的全局最优V”。定义不同J(θ)，本身作为一种建模选择，适应不同任务。下述两者都是把所有状态的表现压缩成一个值。
  * average value：平均状态值（关心return），常用于规定起点任务。J(θ) = vpi_bar = ∑d(s) * vpi(s)，d(s)为s的权重且∑=1。d可以与pi无关，如平均或只看一个（起点）；也可以有关dpi(s)，常用stationary distribution，有关则同时求梯度。
  * average reward：平均单步reward，常用于连续运行。J(θ) = rpi_bar = ∑d(s) * rpi(s)，其中rpi_s=∑pi(a|s，θ) * r(s, a)。一定与策略有关。同时 = lim形式……。在discounted case中两者等价。
  

**基本策略梯度方法-REINFORCE（待完善）**

* 算法：选择不同J(θ)后，梯度上升，根据选择的J不同梯度计算不同。给出统一形式 θt+1=θt + α*∇J(θ)，展开梯度 + SGD用采样估计梯度得 θt+1 = θt + α *∇θ ln[pi(at | st, θt) * qt(st, at)]
* 应用：q未知，用MC 估计q，即为最简单的策略梯度方法。
* 性质：on-policy，采样和迭代的是同一个pi，且本身具有一定探索能力

___

### 十、Actor-Critic方法

**actor - critic (QAC)**

* 算法：在策略梯度 (actor) θt+1 = θt + α *∇θ ln[pi(at | st, θt) * qt(st, at)]基础上，引入TD-learning (critic) 对qt进行评估，代替传统迭代评估qt的方法。
* 性质：on-policy，同策略梯度方法

**Advantage actor - critic (A2C)（待完善）**

* 算法：在QAC基础上引入一个新的偏置b(S)，减少方差，采样时误差更小。梯度引入偏置后不会发生变化。

**off-policy actor - critic**

* 重要性采样：用采样估计期望时
  * 方法1，在目标分布下采样：大数定理求平均
  * 方法2（重要性采样），目标E~p0，使用p1采样时（off-policy）估计E：Ex~p0[X] = ∑p0(x) * x = ∑p1(x) * p0(x) / p1(x) * x = Ex~p1[p0(x) / p1(x) * x]，实际采样服从p1，采样带入后面函数求平均即为p0下的E[X]
* 算法：用重要性采样定理，实现on-policy到off-policy的转换。前面从MC/TD开始都可以这么做。

**Deterministic actor - critic (DPG)**

* 问题：当前策略的NN，输入s输出所有a的概率，只能在a有限个的情况下工作
* 思路：要求策略为确定性的 a = μ(s, θ)，直接输出a的确定性索引，从状态空间的点映射到动作空间的点
* 算法：重新定义目标函数定义 和 梯度计算方法， 定义平均状态值 J(θ) = E[vμ(s)] = ∑d0(s) * vμ(s)，同理推导梯度、算法框架等等。天然是off-policy，只关心a不使用明确策略

**TRPO && PPO 想法**

* 在原有actor - critic框架上，对每次actor的策略梯度更新幅度进行了限制
