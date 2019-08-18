###  zookeeper 选主过程
zk核心机制包含：恢复模式（选主流程）和广播模式（同步流程）。当服务刚启动、leader崩溃、follower不足半数时，系统自动进入选主流程，此时对外不提供服务，当leader选举出来之后，系统进入同步流程，server同步完最新对的状态之后，对外提供服务。  
选举策略主要基于poxas算法，一种成为leaderElection算法，一种成为FastLeaderElection算法，系统默认使用fast算法。
#### Paxos算法
##### 基本定义
算法中的参与者主要分为三个角色，同时每个参与者又可兼领多个角色:  
1) proposer 提出提案，提案信息包括提案编号和提议对的value；  
2）acceptor 收到提案之后可以接受提按；  
3）learner 只能学习被批准的提案  
算法保证一致性的基本语义：  
1）决议只有在被proposers提出后才能被批准  
2）在一次paxos算法的执行实例中，只批准一个value  
3）leaners 只能活动的被批准的value  

##### 基本算法（basic paxos）
算法分为两个结算（决议的提出和批准）
1）prepare 阶段
1. 当proposer希望提出方案V1，首先发出prepare请求至大多数的Acceptor。请求内容为序列号<SN1>  
2. 当Acceptor接收到prepare请求<SN1>时，检查自身上次回复过的prepare请求<SN2>
  - 如果SN2>SN1 请忽略此请求，直接结束本次批准过程
  - 否则检查上次批准的accept请求<SNx, Vx> 并且回复<SNx, Vx> 如果之前没有进行过批准，则简单恢复<OK>
2) accept批准阶段  
1. 经过一段时间收到一些Acceptor回复，回复可能分为以下几种：
  - 回复数量满足多数派，并且所有的回复都是<OK>，则Porposer发出accept请求，请求内容为议案<SN1，V1>;
  - 回复数量满足多数派，但有的回复为：<SN2，V2>，<SN3，V3>……则Porposer找到所有回复中超过半数的那个，假设为<SNx，Vx>，则发出accept请求，请求内容为议案<SN1，Vx>;
  - 回复数量不满足多数派，Proposer尝试增加序列号为SN1+，转1继续执行;
