---
title:  "守望先锋等FPS游戏的网络同步"
---

在一个采用C/S架构的游戏中，客户端和服务端的游戏状态有差异是不可避免的。客户端和服务端各自都维护了一份游戏状态。这两份游戏状态依赖网络包通信保持同步。但由于各客户端到服务端的时延具有不确定性，游戏状态同步变得非常困难。通常服务端在游戏拓扑中承载的是状态仲裁者的角色，客户端玩家看到的“经验证”的有效游戏状态总是延后于服务端的游戏状态。

网络时延是必然存在的，所以游戏状态的不同步也是必然存在的。但我们可以通过技术手段尽量减轻不同步问题对用户体验带来的影响。

技术术语：

1）**Latency**：Latency指的是数据包从客户端发送到服务端再收到服务端回包所用的时间，通常被称为RTT。虽然单程的数据包传输时间并不总是等于RTT/2，但是简单起见我们可以认为两者是相等的。下文说到Latency都是说一个RTT时间，单程Latency则是指RTT/2。

80年代有个工具叫ping使用ICMP echo测试延迟，所以人们常把RTT和ping联系起来。ping这个指令现在还在用。

2）**Hit Box**：角色的3D模型代表了哪些区域是参与到“命中”计算的。你看不到hit box，你只能看到模型。hit box可能比模型大，也可能比模型小，也有可能很不精确，这都取决于具体的实现。我们知道，tick rate会影响命中判定，但是hit box不精确可能对玩家在是否命中方面的感受影响更大。

3）**Tick Rate**：Tick Rate指游戏服务端更新游戏状态的频率。单位是hertz。如果服务器的Tick Rate是64，这就意味着服务端每秒钟最多向客户端发送64次数据包。这些同步数据包包括了游戏状态更新，比如player和场景对象位置等。一次tick的长度就是其持续时间，单位为ms。

比如，64 rate时tick长度是15.6ms，20 rate时是50ms，10 rate时是100ms

4）**Client Update Rate**：这是客户端接收服务端更新的频率。比如说，如果client update rate是20，而服务器tick rate是64，那么从体验上来说，这个客户端实际是在和一个tick rate为20的服务器联机。通常这个是配在客户端本地的，也有可能是写死的。

5）**Framerate**：这个是指客户端每秒最多可以渲染多少帧，通常被称为FPS

6）**Refresh Rate**：显示设备每秒钟刷新多少次。单位为hertz。如果framerate是30，一个显示频率为60的设备将把每个画面显示两次。反过来，如果framerate是120，但是显示频率为60，那么显示设备只能显示每秒60帧。显示设备的频率比framerate大，提升framerate才有意义。大多数显示设备频率是60或120。

7）**Interpolation**：这是一种平滑场景对象移动的技术。实际上内插值所做的就是在场景对象的两个位置之间做插值，以让运动过程平滑。插值延迟通常是2tick，也不尽然。举个内插值的例子，如果一个玩家沿着一条直线移动，在tick1的时候位置在0.5m，在tick2的时候位置在1m，内插值的作用就是让客户端看起来是平滑的从0.5m移到1m。但是服务器实际看到的是离散的位置，要么在0.5m或1m，不可能在中间的某个位置。如果没有插值，游戏的抖动将非常明显，特别是在从服务端更新了一个运动对象的位置后。内插值只在客户端做，实际上减慢了将整个游戏状态绘制到屏幕上的速率。

8）**Extrapolation**：这是客户端补偿延迟的另一种技术。客户端将场景对象的位置做外插值，这样就不会导致绘制的时候没有更新到新数据。通常优先使用内插值，特别是FPS游戏，因为玩家的移动是不可预期的，外插值的结果可能通常是错的。

9）**Lag Compensation**：延迟补偿是服务端减小客户端延迟影响的一种方法。如果没有延迟补偿，或者延迟补偿做的不好，由于客户端看到的是经过延迟后的游戏状态，玩家要命中目标就必须使用一些预判技巧。实际上，延迟补偿所做的，就是当服务器从客户端收到操作（比如开枪）后，将操作发生时间往回调一个单向时延的时间。服务端游戏状态和客户端游戏状态的时间差异（也被称为"Client Delay"）可用下式给出：

ClientDelay = (1/2 * Latency) + InterpolationDelay

延迟补偿的实际操作步骤：

1. Player A看到Player B向一个角落跑去
2. Player A开枪，其客户端把这个操作发送给服务器
3. 假定A的延迟的一半是Xms，那么Xms后服务器将收到Player A的操作
4. 服务器从记录的历史信息中找到A开枪时B所在的位置。一般情况下，服务器应该往回看 (Xms + Player A's interpolation delay) 来回滚到A开枪时的游戏状态。但是这个时间是可以调的，取决于开发者希望延迟补偿算法如何工作。
5. 服务器判定这次的开枪是否命中。如果子弹的轨迹和目标模型的hit box相交，就认为是命中了。在这个例子中，我们假定命中了。在Player B看来，他觉得自己已经躲到墙后面了。但是Player B看到的游戏状态所处的时间和Server认定的开枪时间是有差异的，可以表示为：
   (1/2 * PlayerALatency + 1/2 * PlayerBLatency + TimeSinceLastTick)
6. 在下一次tick中，服务器使用计算结果更新所有客户端：Player A看到自己命中了目标，Player B看到自己掉血或挂掉了。

需要注意的是，如果两个玩家对射，而且都命中了，游戏如何处理就取决于实现了。比如说在CS:GO中，如果先收到的射击操作命中了目标玩家，那么后续收到的那个玩家的射击就会被丢弃。这样就避免了两个玩家的射击请求在同一帧，然后都命中，都挂掉。在Overwatch中，这种情况是可能的。这里是有取舍的。

按照CS:GO的做法，网络较好的玩家是有很大优势的。经常会有“我在挂掉前打中了目标，但是他没死”的情况。你甚至在挂掉前能听到你的枪响和命中的声音，却没对目标造成伤害。

若是在Overwatch中，玩家反应时间的差异对结果影响较小。比如说，如果服务器tick rate是64，若Player A比Player B早15ms射击，那么双方的射击都是在同一个15.6ms tick之内，所以最终结果是双方都命中，都死掉了。

如果延迟补偿过度，就会出现“我朝目标早前的位置开枪，却还是命中他了”。
若延迟补偿不足，则会出现“我必须对目标的移动做预判，这样才能命中”。
服务器做延迟补偿所记录的历史数据应该是有限的，不然高延迟的玩家会明显拖累其他玩家的游戏体验。

在Overwatch中，服务端延迟补偿也被称为Favoring the shooter([https://www.vg247.com/2016/04/05/overwatch-devs-talk-netcode-and-favouring-the-shooter/](https://link.zhihu.com/?target=https%3A//www.vg247.com/2016/04/05/overwatch-devs-talk-netcode-and-favouring-the-shooter/), [https://www.pcgamesn.com/overwatch/overwatch-netcode](https://link.zhihu.com/?target=https%3A//www.pcgamesn.com/overwatch/overwatch-netcode))，也就是说，如果你在自己屏幕上瞄准了目标并射击，那么很大概率将命中目标。也有例外情况。比如，若你射击目标的那一刻，目标跳跃躲开了，这时服务器认为目标做了一个完美的闪避，可能会被判断未命中。所以计算命中时并不总是使用射击那一刻的信息。这是为了玩家体验打的补丁。

如果你是要设计一套同步方案，根据设计目的不同可能有不同的方案。公平性、即时反馈、网络流量等都可能是重要的设计目标。可以参考以下因素：

1）网络链接。延迟越低越好。选择一个延迟最低的服务器开始游戏是很重要的。网络上的拥塞程度也会导致网络延迟。延迟补偿可以帮助解决“射击和命中”的问题，但是如果你的网络不好，更多的情况下，你可能会体验到“已经跑到墙后面还是被打中”或者“我先射击但还是死掉了”的情况。

2）如果你的客户端frame rate很低（只要低于显示设备刷新频率或跟他差不多），会导致感受延迟变大，通常比tick rate带来的问题更严重。

3）尽量使用内插值。大多数游戏使用的内插值间隔是tick间隔的两倍，主要考虑到如果一个数据包丢掉了，玩家的移动中断也不会在屏幕上表现出来。如果网络状况很好，没有丢包，把插值间隔设置为tick间隔是没有问题的。但是如果有丢包，就会导致抖动。比如在CS:GO中，这对体验的影响比把服务端tick rate从20调高到64带来的体验影响更明显。如果这个值设的太低，会导致极大的抖动。

4）如果有可能，你应该增加游戏的client update rate来优化体验。其代价是CPU和带宽消耗。对于客户端来说，除非你家的网络带宽非常低，增加CPU和带宽消耗是可以接受的。

5）如果你的显示设备刷新率是60hz，那么很有可能你根本感受不到tick rate在64和128会有什么差异，因为由于tick rate差异导致的改变根本无法通过你的显示设备体现出来。

6）通常来说，服务端tick rate越高，用户交互就越流畅，也更准确。当然网络同步量也越大。如果我们对比tick rate64（CS:GO比赛）和20（Overwatch Beta服务器宣传的帧率），两者因为帧率差异导致的最大可感受延迟是35ms.平均情况下是17.5ms.大多数人是察觉不到其中的差异的，但是有经验的玩家通常是能感受到的。高的tick rate并不会影响到延迟补偿的工作。所以有时候，你还是会有明明自己已经跑到墙后面了可是还是死了的体验。把tick rate提高到64并不能解决这个问题。

7）Responsiveness: 当你按下按键的时候，需要能立刻看到反馈。这对动作游戏和FPS游戏都是非常重要的。有多个因素会影响即时反馈。首先，客户端发送玩家的输入应该是即时的。其次，客户端不等服务端回应就根据玩家的输入做状态预测和插值。在Overwatch中，客户端会维护一个历史纪录用于验证客户端预测的准确性。最后，服务端tick rate也会影响反馈。投射物的模拟也应和玩家做类似处理，并加上飞行时间，让玩家对反馈产生的时间有预期。

8）处理丢包。在Overwatch中，丢包是通过在客户端加速“命令帧”和在服务端设置命令缓存来解决的([http://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and](https://link.zhihu.com/?target=http%3A//www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and), [http://www.gad.qq.com/article/detail/28682](https://link.zhihu.com/?target=http%3A//www.gad.qq.com/article/detail/28682)). 首先，系统采用确定性模拟技术，将时间量化为“命令帧”。每个命令帧都固定为16毫秒（比赛时是7毫秒）。服务端和客户端模拟都运行在保持同步的时钟和这个量化值之上，保持固定的更新频率。当客户端意识到丢包时，会比约定频率更快的模拟，而服务端则将命令缓冲区增大。客户端发送指令的频率加快，而服务端缓冲变大以容忍更多的丢包。客户端的指令数据包包含了未经服务端确认过的所有指令，这样服务端就有机会在实际模拟并发送确认包前更新缓冲区。



**延迟改进**

暴雪表示会采用一些技术来改进延迟的情况：

- 把网络状况相近的玩家匹配到一起，这样相对公平
- 提供60帧tick的服务器，目前是20帧的服务器
- 网络稳定时候，直接使用客户端指令，而不是缓存48ms的
- 网络波动时候，回溯加一个上限，比如250ms，不再是无限回溯了



- [Overwatch - Gameplay Architecture and Netcode - GDCVault](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)
- [《守望先锋》架构设计与网络同步 - GAD](http://gad.qq.com/article/detail/28682)
- [《守望先锋》中的网络脚本化的武器和技能系统 - GAD](http://gad.qq.com/article/detail/28219)
- [Networking Scripted Weapons and Abilities in Overwatch - GDC Vault](https://www.gdcvault.com/play/1024653/Networking-Scripted-Weapons-and-Abilities)
- [浅谈《守望先锋》中的 ECS 架构 - 云风的 BLOG](https://blog.codingnow.com/2017/06/overwatch_ecs.html)
- [GDC 2017 技术选荐合辑 - 知乎专栏](https://zhuanlan.zhihu.com/p/25703934)
- [守望先锋等 FPS 游戏的网络同步 - 知乎专栏](https://zhuanlan.zhihu.com/p/28825322)
- [A guide to understanding netcode - GAMEREPLAYS.ORG](https://www.gamereplays.org/overwatch/portals.php?show=page&name=overwatch-a-guide-to-understanding-netcode)