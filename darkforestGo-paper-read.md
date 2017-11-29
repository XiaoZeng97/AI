# Paper Read-darkforestGo：《BETTER COMPUTER GO PLAYER WITH NEURAL NET- WORK AND LONG-TERM PREDICTION》

darkforestGo是Facebook推出的围棋AI，主要思想是使用DCNN进行long-term prediction。论文[《BETTER COMPUTER GO PLAYER WITH NEURAL NET- WORK AND LONG-TERM PREDICTION》](https://arxiv.org/abs/1511.06410)发表在ICLR 2016. 代码在[GitHub](https://github.com/facebookresearch/darkforestGo)上开源。这里有一个 2016 USGC Computer Go special session on August 4, 2016，Facebook's Yuandong Tian的[演讲视频](https://www.youtube.com/watch?v=_DBnE2goAu4)，介绍了darkforestGo。

## Introduction

Go的难度在于high beaching factors和对小的变化敏感的subtle board situations. 传统的方法是使用Monte-Carlo Tree Search，但是这种方法会导致过于注重local的信息，最近，DCNN(Deep Convolution Netural Network) 被证明可以在Go上得到很好的效果。darkforestGo就构造了一个结合了MCTS和DCNN的hybrid model。

## Rule

- 整体: 棋子直线紧邻的点上，如果有同色棋子存在，则它们便相互连接成一个不可分割的整体。它们的气也应一并计算。
- 气: 一个棋子在棋盘上，与它直线紧邻的空点是这个棋子的“气”。
![标注黑白子的气](./go.PNG)
- 提子: 子直线紧邻的点上，如果有异色棋子存在，这口气就不复存在。如所有的气均为对方所占据，便呈无气状态。无气状态的棋子不能在棋盘上存在。
- 规则搜索网站: [Sensei’s Library](https://senseis.xmp.net/)

## difficulty
- 分钟多
- 敏感性大（棋盘上小的部分改变会导致很大结果变化）
=> 导致很大的搜索话费

## Method
DCNN+MCTS
将棋盘视为19*19的Multiple channels image，每个channel编码一个不同层面的board information。将目前的棋盘状态作为输入，预测后面k步作为输出。根据之前的工作，使用了如下的feature set：
![Alt text](/images/darkforestgo_1.png)
最终得到standard set中21个channel，加入extend set后得到25个channel.
整个model长成这个样子：
![Alt text](/images/darkforestgo_2.png)

### Feature Channels

上面的表格就展示了从目前棋盘中得到的features，每个feature（除了history information和position mask）是一个19\*19的binary map，每个单元的值为[0,1]的实数。history information为exp(-t\*0.1),t是棋子放置的时间；position mask为exp(-1/2*l^2)，其中l^2为到棋盘中心的距离的平方。

### Network Architecture

上图所示的就是使用的网络模型，使用12层全卷积网络。每层都跟着加上一层ReLU非线性。除第一层外其余的宽度均为w=384.没有使用weight sharing和pooling。输出层仅使用一个softmax层进行预测，减小parameter的数量。

### Long Term Planning

只预测下一步限制了低层的信息获得，所以我们预测接下来的k步移动。每一次移动都以一层单独的softmax输出。

### Traning

使用16个CPU线程来准备minibatch，每一个线程随机在dataset中模拟300场游戏。在每一个minibatch中，每一个thread都随机的选择300场游戏中的一场，根据游戏记录模拟一步，提取特征，然后将下k步作为批量中的输入/输出对。 如果游戏已经结束（或者剩余的次数小于k步），我们随机从训练集中挑选一个（替换）并继续。 batch size为256，使用90度间隔的旋转和水平/垂直翻转的数据增强。 对于每种情况，数据增强可以产生8种不同的情况。

对于解决局部极小的情况，在initialize game的时候从几个不同的stages开始。

### Monte Carlo Tree Search

由于缺乏搜索，可以明显的看出DCNN的效果不是很好，所以我们使用了目前Go中state-of-art的方法MCTS。

将MCTS和DCNN结合的一个问题是MCTS的运行速度远快于DCNN，需要频繁的进行通信从而同时进行。对此有两种解决方法：
* **asynchronized implementation**：在Maddison等人使用的异步实现（2015）中，MCTS将新扩展的节点发送到DCNN，但不被DCNN阻塞。MCTS将使用自己的树搜索策略，直到DCNN评估到达并更新移动。这给出了高的推出率，但是DCNN评估生效有一段时间滞后，并且尚不清楚在给定数量的MCTS rollouts中评估了多少个棋盘的情况。
* **synchronized implementation**：MCTS将等到DCNN评估叶节点的board situation，然后展开叶节点。默认策略可以在DCNN评估之前/之后运行。这是慢得多，但保证每个节点都扩大使用DCNN的高品质的建议。

下面是一个使用了DCNN的MCTS过程：
![Alt text](/images/darkforestgo_3.png)
(a) A game tree. 对于每个节点m/n表示模拟n场游戏，其中m场黑子胜。root表示当前游戏状态。
(b) 从根开始搜索。使用树策略选择从当前状态移动到下一个游戏状态，直到它选择新的移动并扩展新的叶节点。从叶节点，我们运行默认策略，直到游戏结束（黑子胜）。同时，叶节点状态被发送到DCNN服务器进行评估。对于同步实现，在返回评估之后，此新节点可用于树策略。
(c) 沿着搜索轨迹的统计资料也相应更新。

* **树策略**：移动首先按照DCNN置信度排序，然后按顺序进行挑选，直到累计概率超过0.8，或达到最大移动次数。 然后我们使用UCT选择移动扩大（在UCT中不使用DCNN置信度）。 在[0，σ]中均匀分布的噪声被添加到胜率以强制搜索线程快速发散并且不锁定在等待DCNN评估的同一节点中。这极大地加速了树搜索（在实验中σ= 0.05）。
* **默认策略**：遵循Pachi的实施，我们使用3x3模式，对手分数点，检测点数和避免自我违约。Pachi的完整默认策略的性能略好。 由于多线程的非确定性，DCNN + MCTS和DCNN之间的博弈总是不同的。

## Experiments

dataset: KGS dataset(~170k games)，在此paper中使用了2012年之前的作为training set(144,748)，2013-2015年的作为test set(26,814)；
GoGoD dataset(~80k games)，training set 75,172，test set 2,592.

实验结果显示darkforestGo的performance很好。

## Weekness
尽管有MCTS的参与，模型仍然有一些弱点。 
1. DCNN的top-3/5移动可能不包含关键的local move来保存/杀死local self/enemy group，因此local tactics依然比较弱。 有时，当需要tight local battle时，bot会毫无意义地tenuki（移动到别处）。 
2. DCNN往往对KO移动有high confidence，即使它们是无用的。This enables DCNN to play single ko fights decently, by following the pattern of playing the ko, playing ko threats and playing the ko again. But it also gets confused in the presence of double ko. (这句话真的不知道怎么用中文表达。。。)
3.  当bot失败时，其他MCTS bot也会进行bad moves，从而导致loses更多。
