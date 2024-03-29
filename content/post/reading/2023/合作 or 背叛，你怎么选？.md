---
title: 合作 or 背叛，你怎么选？
description: 《合作的进化》是罗伯特·阿克塞尔罗德的一本博弈论经典之作，作者以“重复囚徒困境”游戏作为切入点，讲述了在与他者的持续交往中，应该选择怎样的合作策略才能得到较好的收益。本书的一个重要结论是，人们相互作用越频繁，合作的可能性就越大。
date: 2023-12-28
categories:
  - 读书的地方
tags:
  - 读书笔记
  - 博弈论
---
 
> 《合作的进化》是罗伯特·阿克塞尔罗德的一本博弈论经典之作，作者以“重复囚徒困境”游戏作为切入点，讲述了在与他者的持续交往中，应该选择怎样的合作策略才能得到较好的收益。本书的一个重要结论是，人们相互作用越频繁，合作的可能性就越大。
>
> 本文是对《合作的进化》核心内容的总结。

## 前言

先从两个简单的问题开始：

- 在与他人的持续交往中，人什么时候应该合作，什么时候只需为自己着想？
- 一个人会继续帮助他的一位从来不思回报的朋友吗？

合作，是我们常常要面对的事，小到个人，大到国家。合作虽然能带来好处，但并不总是能建立起来。

两国之间的贸易壁垒就是一个很好的例子。显然，自由贸易对两国都能带来好处，但问题是，无论谁单方面取消壁垒，都会导致自己国家处于不利的贸易状态。所以，无论一个国家如何做，另一个国家保持它的贸易壁垒总是比较有利的。因此，**每个国家都有利益动机保持贸易壁垒**。

同理，在人与人之间、公司之间，也会存在类似的场景，我们可以用一个简单的游戏来进行模拟，那就是著名的“**囚徒困境**”游戏。

## 囚徒困境

在“囚徒困境”游戏中，有两个对策者，每人都有两个选择：合作或背叛，并且都需要在不知道对方选择的情况下做出自己的选择。每种选择对应的收益如下图所示：

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-12-23-041151.png)

从图中可以看出，当双方都选择合作时，集体收益是最高的 6；当一人选择合作，另一人选择背叛时，背叛一方的个人收益是最高的 5。*之所以称为“困境”，是因为不管对方怎么选择，从个人的角度，背叛都是更好的选择，而双方背叛却会导致最低的集体收益*。

从上述结论来看，背叛是必然的选择，那么为什么现实世界中还会存在这么多的合作呢？

其实，上述结论的前提是，对策者双方只相遇一次，在只考虑当前回合的最大收益情况下，背叛是必然的。但如果双方会在将来相遇多次，那么就要考虑后续的收益，情况将变得复杂，我们将这称为“**重复囚徒困境**”。

为了探讨“重复囚徒困境”下的最优策略，作者组织了重复囚徒困境游戏的计算机竞赛，并邀请众多博弈论专家参与。

## “一报还一报”

在计算机竞赛中，参加者提交一个能给出每一回合策略（合作或者背叛）的程序，该程序在作策略选择时可以利用对局的历史。竞赛是循环的，即每个程序都会与其他程序相遇，每轮比赛有 200 次对局，以此来模拟出“重复”。

竞赛的结果有点出乎意料，**胜者是“一报还一报”策略，它是所有提交程序中最简单的策略，结果却是最好的**。

“一报还一报”的策略是，*最开始选择合作，随后按对方上一回合的选择去做*，如果对方出现背叛，那么“一报还一报”也会在下一回合背叛来进行“报复”，但只会报复一个回合。下图举例了“一报还一报”与其他策略的 6 个回合的对局情况。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-12-27-050859.png)

从上述例子来看，“一报还一报”策略在每一个对局之中都不是得分领先的一方，甚至在遇到全背叛策略 A 时 6 个回合的得分仅仅 3 分。但在单回合不占优的情况下，“一报还一报”依然在竞赛中却能获得胜利。

> 值得注意的是，在所有竞赛参与者中，善良策略（不首先背叛）的平均得分要比非善良策略（首先背叛）的平均得分高 25%！前 8 名的策略都是善良的。而在善良策略中，得分最低的是最少宽容性的规则，一个采用永久报复的完全不宽容的规则，。

“一报还一报”策略成功的原因是它综合**了善良性、报复性、宽容性和清晰性**。善良性防止它陷入不必要的麻烦；报复性使对方试着背叛一次后就不敢再背叛；宽容性有助于重新恢复合作；清晰性使它容易被对方理解，从而引出长期的合作。

## 合作 or 背叛，你怎么选？

那么，在人与人相处的时候，我们应该怎么做呢？作者给出了 4 点建议:

- 不要嫉妒
- 不要首先背叛
- 对合作与背叛都要以回报
- 不要耍小聪明

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-12-28-140012.png)

### 不要嫉妒

人们总是习惯考虑**零和博弈**，这种情况下，一个人赢，另一个就输，比如下棋。但在现实生活中，**大多数情况属于非零和博弈**，也即可以双赢，也能双输。这种情况下，合作是可能的，但并不总能实现。

人们倾向于采用**相对的标准**，**经常把对方的成功与自己的成功联系起来**，从而导致了嫉妒。当嫉妒出现时，人会企图抵消对方已经得到的优势，通常通过背叛来实现。但背叛往往会导致更多的背叛，进而是双输的局面。**因此嫉妒无疑是自我毁灭**。

> 要求自己比对方做得好不是一个很好的标准，除非你的目的是消灭对方。

更好的做法是，把你所做的，与处在相同情况下的其他人所做的，进行比较。对于相同的策略，你是否已经做得最好了？其他人在这种情况下能做得更好吗？

因此，**只要你自己能做得更好，就让对方做得和你一样或略好些**，就像“一报还一报”在每个对局中并不是优胜的一方，但并不妨碍它能赢得竞赛的胜利。

我们没有理由去嫉妒他人的成功，***因为在长时间的“重复囚徒困境”中，他人的成功是你自己成功的前提***！

### 不要首先背叛

竞赛和理论分析的结果都表明，在非零和博弈中，善良策略是表现更好的一方，善良性能够避免不必要的冲。非善良策略在开头还显得挺有希望，但是时间一长它就摧毁了它自己赖以成功的基础。

### 对合作与背叛都要给予回报

竞赛中，“一报还一报”的成功给我们一个简单且有力的建议：**要回报**。

*对善良者报以合作，对非善良者报以背叛*。

值得注意的是，在报以背叛时，我们**要保持惩罚与宽恕的平衡**。“一报还一报”总是在对方每次背叛之后只背叛一次，而不是永久背叛，因此它成功了。

但并不是说只背叛一次就是最优的，**最优的宽恕水平与环境有关**，如果周边绝大多数都是非善良的策略，那么，太多的宽恕反而会付出更多的代价。

### 不要耍小聪明

在“囚徒困境”的情形中，人们容易耍小聪明，然而复杂的规则并不比简单的规则做得更好，赢得竞赛的反而是最简单、最易理解的“一报还一报”策略。

原因主要在于，在零和博弈中（如下棋），对手任何的无效行为都会转换成你的收益，因此隐藏你的企图是很有用的，对手越是怀疑，策略越是没效；而在非零和博弈（如“重复囚徒困境”）中，你需要从对方的合作中得到好处，**就必须鼓励合作，而对手越是清楚你愿意合作，合作就越容易促成**。

“一报还一报”在竞赛中得到巨大成功的原因之一，就是它具有很大的**清晰性**，非常容易被对方理解。

## 最后

从前文可以得出这样的结论：在“重复囚徒困境”，也即非零和博弈中，合作是过更好的选择。

> 作者在书中举了一个现实的例子：在第一次世界大战堑壕战中，敌对的士兵经常表现出很大的克制，双方似乎都默契地执行着“自己活也让别人活”的策略。这种现象是堑壕战的特产。因为在堑壕战中，敌对双方都需要长时间对峙，符合“重复囚徒困境”的条件。而在其他类型战争，如突袭战、闪电战，则并不会出现类似的情况。

为了更好地促进合作，罗伯特·阿克塞尔罗德也提出了几个个建议：

- **增大未来的影响**。如果未来相对于现在是足够重要的，或者双方有更高的频率在未来相遇的话，人们在做决策时就会更加考虑对未来的影响，从而更能促进合作。
- **改变收益值**。比如增大对背叛的惩罚，当背叛的惩罚大到不管对方如何选择，从短期来看合作都是最好的策略时，就不会再有困境了。
- **教育人们相互关心和回报**。促进合作的极好的方法依然是从教育入手，在孩童时代就教育人们要关心他人的利益、懂得回报，形成价值观。

读到这里，合作或者背叛应该怎么选，你清楚了吗？
**如果还没有，推荐大家玩一下《[信任的进化](https://dccxi.com/trust/)》这款游戏，改编自《合作的进化》，通过图形化的表达让你更易理解书中的内容。**

### 文章配图

可以在 [用Keynote画出手绘风格的配图](https://mp.weixin.qq.com/s/-sYW-oa6KzTR9LNdMWCSnQ) 中找到文章的绘图方法。

> #### 参考
>
> [1] [合作的进化](https://weread.qq.com/web/bookDetail/92c32a40813ab6faeg013e9b), 罗伯特·阿克塞尔罗德
>
> 更多文章请关注微信公众号：**元闰子的邀请**



