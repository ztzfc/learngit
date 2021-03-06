# CAP >>>>>> 分布式事物

## CAP原则
>CAP 原则又称 CAP 定理，指的是在一个分布式系统中， Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可得兼。

- 一致性（C）
>在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本） 

一致性是因为有并发读写才有的问题，因此在理解一致性的问题时，一定要注意结合考虑并发读写的场景。
- 可用性（A）
>在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）,就是只要收到用户的请求，服务器就必须给出回应

- 分区容错性（P）
>多数分布式系统都分布在多个子网络。每个子网络就叫做一个区（partition）。分区容错的意思是，区间通信可能失败。比如，一台服务器放在中国，另一台服务器放在美国，这就是两个区，它们之间可能无法通信。
>这个是无法避免的，所以P必须成立。根据CAP定理，剩下的C和A无法共存

举个栗子： 

1. 一个分布式系统里面，节点组成的网络本来应该是连通的。然而可能因为一些故障，使得有些节点之间不连通了，整个网络就分成了几块区域。数据就散布在了这些不连通的区域中。这就叫分区。
2. 当你一个数据项只在一个节点中保存，那么分区出现后，和这个节点不连通的部分就访问不到这个数据了，出现分区容错。
3. 解决分区容错的办法就是一个数据项复制到多个节点上，那么出现分区之后，这一数据项就可能分布到各个区里。容忍性就提高了。
4. 然而，要把数据复制到多个节点，就会带来一致性的问题，就是多个节点上面的数据可能是不一致的。要保证一致，每次写操作就都要等待全部节点写成功，会阻塞其他访问，降低可用性。
5. 总的来说就是，数据存在的节点越多，分区容忍性越高，但要复制更新的数据就越多，一致性就越难保证。为了保证一致性，更新所有节点数据所需要的时间就越长，可用性就会降低。

## 一致性分析

- 强一致性
>任何一次读都能读到某个数据的最近一次写的数据。系统中的所有进程，看到的操作顺序，都和全局时钟下的顺序一致。简言之，在任意时刻，所有节点中的数据是一样的。

- 弱一致性
>数据更新后，如果能容忍后续的访问只能访问到部分或者全部访问不到，则是弱一致性

- 最终一致性
>不保证在任意时刻任意节点上的同一份数据都是相同的，但是随着时间的迁移，不同节点上的同一份数据总是在向趋同的方向变化。简单说，就是在一段时间后，节点间的数据会最终达到一致状态。

- 弱一致性和最终一致性区别

经过一定时间窗口后，数据达到一致是最终一致性，当然这个时间窗口越小越好。经过时间窗口后，数据达不到一致，那就是弱一致性

## 节点数选择

`降低达到最终一致性的时间窗口，对提高系统的可用度和用户体验非常重要的`

N — 数据复制的份数
W — 更新数据是需要保证写完成的节点数
R — 读取数据的时候需要读取的节点数

- 如果W+R>N，写的节点和读的节点重叠，则是强一致性。例如对于典型的一主一备同步复制的关系型数据库，N=2,W=2,R=1，则不管读的是主库还是备库的数据，都是一致的。

- 如果W+R<=N，则是弱一致性。例如对于一主一备异步复制的关系型数据库，N=2,W=1,R=1，则如果读的是备库，就可能无法读取主库已经更新过的数据，所以是弱一致性。

- 对于分布式系统，为了保证高可用性，一般设置N>=3。不同的N,W,R组合，是在可用性和一致性之间取一个平衡，以适应不同的应用场景。

- 如果N=W,R=1，任何一个写节点失效，都会导致写失败，因此可用性会降低，但是由于数据分布的N个节点是同步写入的，因此可以保证强一致性。

- 如果N=R,W=1，只需要一个节点写入成功即可，写性能和可用性都比较高。但是读取其他节点的进程可能不能获取更新后的数据，因此是弱一致性。这种情况下，如果W<(N+1)/2，并且写入的节点不重叠的话，则会存在写冲突  

## 分布式事物 `TODO`


###### 参考

- https://blog.csdn.net/DaDe13621060726/article/details/107482496
- https://www.cnblogs.com/incognitor/p/9759956.html
- https://www.huaweicloud.com/articles/383a27a85de5a2d11d4e601b5847c3c9.html
- https://xiaomi-info.github.io/2020/01/02/distributed-transaction/ `TODO`
- https://www.cnblogs.com/rickiyang/p/13704868.html `TODO`
