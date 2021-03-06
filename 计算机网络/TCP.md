# 两个进程通信方式

* 管道
* 内存共享
* 信号量
* 消息队列

# IP寻址

* 本地网络寻址
  * A通过hosts将B计算机名转换为ip地址
  * 用自己的IP与子网掩码计算自己所处网段，比较B的IP与自己和子网掩码，发现处于相同网段
  * 在自己ARP缓存中查找是否有B的mac地址
  * 若没有，A启动ARP协议在本地网络arp广播来查询B的mac地址，获得后写入arp缓存
* 非本地网络寻址
  * A通过本机hosts或dns系统将B计算机名转为IP地址
  * 用自己IP与子网掩码计算自己网段与B比较，发现处于不同网段
  * A在自己arp缓存中查找是否有缺少网关（路由器本地接口）的MAC地址，若没找到则启动arp协议通过本地网络上的arp广播查询网关mac地址，获得后写入arp缓存表
  * 数据到达路由器后根据目的IP查找路由表，发送到下一跳路由，直到达到目的的的网络与主机

# TCP报文到达确认(ACK)机制

原文: <http://blog.csdn.net/wjtxt/article/details/6606022>

TCP数据包中的序列号（Sequence Number）不是以报文段来进行编号的，而是将连接生存周期内传输的所有数据当作一个字节流，序列号就是整个字节 流中每个字节的编号。一个TCP数据包中包含多个字节流的数据（即数据段），而且每个TCP数据包中的数据大小不一定相同。**在建立TCP连接的三次握手 过程中，通信双方各自已确定了初始的序号x和y，TCP每次传送的报文段中的序号字段值表示所要传送本报文中的第一个字节的序号。**

**TCP的报文到达确认（ACK），是对接收到的数据的最高序列号的确认，并向发送端返回一个下次接收时期望的TCP数据包的序列号（Ack Number）。**例如， 主机A发送的当前数据序号是400，数据长度是100，则接收端收到后会返回一个确认号是501的确认号给主机A。

**TCP提供的确认机制，可以在通信过程中可以不对每一个TCP数据包发出单独的确认包（Delayed ACK机制），而是在传送数据时，顺便把确认信息传出， 这样可以大大提高网络的利用率和传输效率。同时，TCP的确认机制，也可以一次确认多个数据报，例如，接收方收到了201，301，401的数据报，则只 需要对401的数据包进行确认即可，对401的数据包的确认也意味着401之前的所有数据包都已经确认，这样也可以提高系统的效率。**

**若发送方在规定时间内没有收到接收方的确认信息，就要将未被确认的数据包重新发送。接收方如果收到一个有差错的报文，则丢弃此报文，并不向发送方 发送确认信息。因此，TCP报文的重传机制是由设置的超时定时器来决定的，在定时的时间内没有收到确认信息，则进行重传。**这个定时的时间值的设定非 常重要，太大会使包重传的延时比较大，太小则可能没有来得及收到对方的确认包发送方就再次重传，会使网络陷入无休止的重传过程中。**接收方如果收到 了重复的报文，将会丢弃重复的报文，但是必须发回确认信息，否则对方会再次发送。**

TCP协议应当保证数据报按序到达接收方。

**如果接收方收到的数据报文没有错误，只是未按序号，这种现象如何处理呢**？TCP协议本身没有规定，而是由TCP 协议的实现者自己去确定。

通常有两种方法进行处理：

一是对没有按序号到达的报文直接丢弃，

二是将未按序号到达的数据包先放于缓冲区内，等待它前面 的序号包到达后，再将它交给应用进程。后一种方法将会提高系统的效率。

例如，发送方连续发送了每个报文中100个字节的TCP数据报，其序号分别是1， 101，201，…,701。假如其它7个数据报都收到了，而201这个数据报没有收到，则接收端应当对1和101这两个数据报进行确认，并将数据递交给相关的应用 进程，301至701这5个数据报则应当放于缓冲区，等到201这个数据报到达后，然后按序将201至701这些数据报递交给相关应用进程，并对701数据报进行 确认，确保了应用进程级的TCP数据的按序到达。

# 深入剖析TCP协议的send与recv

## 滑动窗口的概念

TCP数据包的TCP头部有一个window字段，它主要是用来告诉对方自己能接收多大的数据（注意只有TCP包中的数据部分占用这个空间），这个字段在通信双方建立连接时协商确定，并且在通信过程中不断更新，故取名为滑动窗口。有了这个字段，数据发送方就知道自己该不该发送数据，以及该发多少数据了。**TCP协议的流量控制正是通过滑动窗口实现，从而保证通信双方的接收缓冲区不会溢出，数据不会丢失。**

由于窗口大小在TCP头部只有16位来表示，所以它的最大值是65536，但是对于一些情况来说需要使用更大的滑动窗口，这时候就要使用扩展的滑动窗口，如光纤高速通信网络，或者是卫星长连接网络，需要窗口尽可能的大。这时会使用扩展的32位的滑动窗口大小。

## 滑动窗口移动规则

1、**窗口合拢**：在收到对端数据后，自己确认了数据的正确性，这些数据会被存储到接收缓冲区，等待应用程序获取。但这时候因为已经确认了数据的正确性，需要向对方发送确认响应ACK，又因为这些数据还没有被应用进程取走，这时候便需要进行窗口合拢，缓冲区的窗口左边缘向右滑动。注意响应的ACK序号是对方发送数据包的序号，一个对方发送的序号，可能因为窗口张开会被响应（ACK）多次。

2、**窗口张开**：窗口收缩后，应用进程一旦从缓冲区(滑动窗口区或接收缓冲区)中取出数据，TCP的滑动窗口需要进行扩张，这时候窗口的右边缘向右扩张，实际上窗口这是一个环形缓冲区，窗口的右边缘扩张会使用原来被应用进程取走内容的缓冲区。**在窗口进行扩张后，需要使用ACK通知对端，这时候ACK的序号依然是上次确认收到包的序号。**

3、**窗口收缩**，窗口的右边缘向左滑动，称为窗口收缩，HostRequirement RFC强烈建议不要这样做，但TCP必须能够在某一端产生这种情况时进行处理。

## send行为

默认情况下，send的功能是拷贝指定长度的数据到发送缓冲区，只有当数据被全部拷贝完成后函数才会正确返回，否则进入阻塞状态或等待超时。如果你想修改这种默认行为，将数据直接发送到目标机器，可以将发送缓冲区大小设为0（或通过TCP_NODELAY禁用Nagle算法），这样当send返回时，就表示数据已经正确的、完整的到达了目标机器。注意，这里只表示数据到达目标机器网络缓冲区，并不表示数据已经被对方应用层接收了。

协议层在数据发送过程中，根据对方的滑动窗口，再结合MSS值共同确定TCP报文中数据段的长度，以确保对方接收缓冲区不会溢出。当本方发送缓冲区尚有数据没有发送，而对方滑动窗口已经为0时，协议层将启动探测机制，即每隔一段时间向对方发送一个字节的数据，时间间隔会从刚开始的30s调整为1分钟，最后稳定在2分钟。这个探测机制不仅可以检测到对方滑动窗口是否变化，同时也可以发现对方是否有异常退出的情况。

push标志指示接收端应尽快将数据提交给应用层。如果send函数提交的待发送数据量较小，例如小于1460B（参照MSS值确定），那么协议层会将该报文中的TCP头部的push字段置为1；如果待发送的数据量较大，需要拆成多个数据段发送时，协议层只会将最后一个分段报文的TCP头部的push字段置1。

## recv行为

默认情况下，recv的功能是从接收缓冲区读取(其实就是拷贝)指定长度的数据。如果将接收缓冲区大小设为0，recv将直接从协议缓冲区(滑动窗口区)读取数据，避免了数据从协议缓冲区到接收缓冲区的拷贝。recv返回的条件有两种：

\1. recv函数传入的应用层接收缓冲区已经读满

\2. 协议层接收到push字段为1的TCP报文，此时recv返回值为实际接收的数据长度

协议层收到TCP数据包后(保存在滑动窗口区)，本方的滑动窗口合拢（窗口值减小）；当协议层将数据拷贝到接收缓冲区(滑动窗口区—>接收缓冲区)，或者应用层调用recv接收数据(接收缓冲区—>应用层缓冲区，滑动窗口区—>应用层缓冲区)后，本方的滑动窗口张开(窗口值增大)。收到数据更新window后，协议层向对方发送ACK确认。

协议层的数据接收动作完全由发送动作驱动，是一个被动行为。在应用层没有任何干涉行为的情况下（比如recv操作等），协议层能够接收并保存的最大数据大小是窗口大小与接收缓冲区大小之和。Windows系统的窗口大小默认是64K，接收缓冲区默认为8K，所以默认情况下协议层最多能够被动接收并保存72K的数据。



# TCP三次握手

* **第一次握手**
  * 主机A发送位码为syn＝1,随机产生seq number=x的数据包到服务器，主机B由SYN=1知道，A要求建立联机，此时状态为SYN_SENT； 
* **第二次握手**
  * 主机B收到请求后要确认联机信息，向A发送ack number=(主机A的seq+1),syn=1,ack=1,随机产生seq=y的包，此时状态由LISTEN变为SYN_RECV； 
* **第三次握手**
  * 主机A收到后检查ack number是否正确，即第一次发送的seq number+1,以及位码ack是否为1，若正确，主机A会再发送ack number=(主机B的seq+1),ack=1，主机B收到后确认seq值与ack=1则连接建立成功，双方状态ESTABLISHED。

完成三次握手，主机A与主机B开始传送数据

## 为什么两次就建立连接还要三次握手呢？

* 初始化Sequence num 值

* 防止已失效的连接请求报文又突然传递服务器。

  * 所谓“防止已失效的连接请求报文又突然传递服务器。”是这样一种情况： 

    A客户端发出连接请求，因为连接请求报文丢失而未等到确认。于是A再次重传了连接请求，建立了连接。数据传输完毕后，释放了连接。现在假设那第一个请求只是因为网路节点长时间滞留了，使得它在第二个连接释放后才到达B服务器，那么B会以为这是一个新的连接请求，于是就向A发了个连接确认，注意了：如果没有最后一次的确认B会一厢情愿的以为连接已经建立，可人家A同学一看那个B给的是什么呀！跟自己没关系，简单粗暴的丢掉。这时B孩子还傻傻的等着A给他发数据，就这样，B白白浪费的大把的时光和资源。 

    那B会一直傻等吗？当然不是，它的等待也是有限的，答案就是保活计时器。

# TCP四次挥手

![深度截图_选择区域_20190609112731.png](https://i.loli.net/2019/06/09/5cfc7cb4489e369847.png)

1. 第一次分手：主机1（可以使客户端，也可以是服务器端），设置Sequence Number和Acknowledgment Number，向主机2发送一个FIN报文段；此时，主机1进入FIN_WAIT_1状态；这表示主机1没有数据要发送给主机2了；
2. 第二次分手：主机2收到了主机1发送的FIN报文段，向主机1回一个ACK报文段，Acknowledgment Number为Sequence Number加1；主机1进入FIN_WAIT_2状态；主机2告诉主机1，我“同意”你的关闭请求；
3. 第三次分手：主机2向主机1发送FIN报文段，请求关闭连接，同时主机2进入LAST_ACK状态；
4. 第四次分手：主机1收到主机2发送的FIN报文段，向主机2发送ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段以后，就关闭连接；此时，主机1等待2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，主机1也可以关闭连接了。

## 为什么需要四交握手才能关闭连接

* 因为全双工,发送方和接收方都需要FIN报文和ACK报文

## 服务器出现大量CLOSE_WAIT状态的原因

* 对方关闭socket连接,我方忙于读或写,没有关闭读写
  * 检查代码,特别是释放资源的代码
  * 检查配置,特别是处理请求的线程配置

## 为什么要有四次挥手的TIME_WAIT的状态 

* 保证最后一个的一个ACK报文能到达B**。**
  * **这个ACK报文有可能丢失，因而使得处在LAST_ACK状态得不到对已发送的FIN+ACK报文的确认，**B会超时重传这个FIN+ACk ,而A就能在这TIME_WAIT时间（2MSL）里收到这个重传的报文，A就可以重传一次确认，如果没有这个TIME_WAIT， 那B重传的FIN_ACK，可A早就走了，自然不会再重发确认，这样B就无法按照正常步骤进入CLOSE 状态。 
* 防止“**已失效的报文连接请求**”
  * A在TIME_WAIT中，经过这2MSL的时间，就可以使本链接持续的时间内产生的所有连接消失，**这样就可以使下一个新的连接中不会出现这样旧的连接请求报文段**。 
* **谁先关闭谁就有一个TIME_WAIT的状态；**


在linux的网络编程中，如果服务器如果先关闭，你会发现，现在想要立马再次启动服务器，就会报错说这个端口号被占用着，那就是因为有这个TIME_WAIT，2msl的时间.那么怎么解决 ？ 

解决：setsockopt（）函数。在这就不多说了。

# 拥塞控制

* 拥塞控制三种动作
  * 收到一新确认，网络正常
    * 此时增加单次发送量
      * 若单次发送量小于倍增阈值，发送量X2，指数增
      * 否则发送量+1，线性增
  * 收到三条重复确认，网络繁忙
    * 单次发送量减半，倍增阈值=单次发送量（进入线性增长期）
  * 确认未收到，超时。网络更加繁忙
    * 倍增阈值=单次发送量/2，单次发送量=1

* TCP的AIMD（加性增窗、乘性减窗）策略

  ```c
  While(Sending_Not_Finish){
  	if(Not_Loss_Packet){
  		CongWin++;
  	}else
  		CongWin=[CongWin/2]; //[]的意思是取整
  }
  ```

  

# 可靠性传输-差错控制

* 检验和
  * 每个报文段有16位检验和，若检验和无效，则由终点TCP挂失报文。

* 确认   
  * 积累确认（ACK）
    * ​  接收方通告期望接收下一节点seqnum，忽略失序到达报文。
    * 在TCP首部的32位ACK字段用于积累确认，它的值仅在ACK标志为1时才有效
  * 选择确认（SACK`selective acknowledgment`）
    * SACK要报告失序的数据块及重复的报文段
  * 产生确认的情况

* 重传（差错控制核心）
  * 一个报文段发送时，会被保存到一个队列中，直至被确认为止
  * 重传计时器超时或发送方收到一报文段三个重复ACK，该报文段重传​
  * 类型
    * RTO重传（超时重传）
      * 发送方TCP计时器时间到，TCP发送队列最前面报文段（序列号最小），并重启计时器
      * RTO值是动态的，根据报文段的往返时间（RTT）更新
    * 三个重复的ACK报文段（快重传）
      * 若某报文段有3个重复确认，立即重传并重启RTO计时器，而不用等待计时器超时

* 其它
  * TCP不会将失序到达报文段丢了，会暂时保存并标为失序，直至失序报文段到齐

# 子网掩码的计算及与子网数、主机数关系

子网掩码是一个32位地址，是与IP地址结合使用的一种技术。它的主要作用有两个，**一是用于屏蔽IP地址的一部分以区别**[**网络标识**](http://baike.baidu.com/view/1120331.htm)**和**[**主机**](http://baike.baidu.com/view/23880.htm)**标识，并说明该IP地址是在**[**局域网**](http://baike.baidu.com/view/788.htm)**上，还是在远程网上。二是用于将一个大的IP网络划分为若干小的子网络。**

 使用子网是为了减少IP的浪费。因为随着互联网的发展，越来越多的网络产生，有的网络多则几百台，有的只有区区几台，这样就浪费了很多IP地址，所以要划分子网。使用子网可以提高网络应用的效率。

通过IP 地址的二进制与子网掩码的二进制进行与运算，确定某个设备的网络地址和主机号，也就是说通过子网掩码分辨一个网络的网络部分和主机部分。子网掩码一旦设置，网络地址和主机地址就固定了。

1、利用子网数目计算子网掩码

把B类地址172.16.0.0划分成30个子网络，它的子网掩码是多少？

①将子网络数目30转换成二进制表示11110

②统计一下这个二进制的数共有5位

③注意：当二进制数中只有一个1的时候，所统计的位数需要减1（例如：10000要统计为4位）

④将B类地址的子网掩码255.255.0.0主机地址部分的前5位变成1

⑤这就得到了所要的子网掩码（11111111.11111111.11111000.00000000）255.255.248.0。

# DHCP工作过程的六个主要步骤

DHCP分为两个部分：一个是服务器端，另一个是客户端。

所有客户机的IP地址设定资料都由DHCP服务器集中管理，并负责处理客户端的DHCP请求；而客户端则会使用从服务器分配下来的IP地址。

## 1. DHCP服务器IP分配方式

DHCP服务器提供三种IP分配方式：

- 自动分配（Automatic Allocation） 自动分配是当DHCP客户端第一次成功地从DHCP服务器端分配到一个IP地址之后，就永远使用这个地址。
- 动态分配（Dynamic Allocation） 动态分配是当DHCP客户端第一次从DHCP服务器分配到IP地址后，并非永久地使用该地址，每次使用完后，DHCP客户端就得释放这个IP地址，以给其他客户端使用。
- 手动分配 手动分配是由DHCP服务器管理员专门为客户端指定IP地址。

## 2. DHCP服务工作流程

## 

![img](https://gss0.bdstatic.com/94o3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=d19ffbc2d21373f0e13267cdc566209e/5ab5c9ea15ce36d3d5e5e08939f33a87e850b1a1.jpg)

1. DHCP Client以广播的方式发出DHCP Discover报文。

2. 所有的DHCP Server都能够接收到DHCP Client发送的DHCP Discover报文，所有的DHCP Server都会给出响应，向DHCP Client发送一个DHCP Offer报文。

   DHCP Offer报文中“Your(Client) IP Address”字段就是DHCP Server能够提供给DHCP Client使用的IP地址，且DHCP Server会将自己的IP地址放在“option”字段中以便DHCP Client区分不同的DHCP Server。DHCP Server在发出此报文后会存在一个已分配IP地址的纪录。

3. DHCP Client只能处理其中的一个DHCP Offer报文，一般的原则是DHCP Client处理最先收到的DHCP Offer报文。

   DHCP Client会发出一个广播的DHCP Request报文，在选项字段中会加入选中的DHCP Server的IP地址和需要的IP地址。

4. DHCP Server收到DHCP Request报文后，判断选项字段中的IP地址是否与自己的地址相同。如果不相同，DHCP Server不做任何处理只清除相应IP地址分配记录；如果相同，DHCP Server就会向DHCP Client响应一个DHCP ACK报文，并在选项字段中增加IP地址的使用租期信息。

5. DHCP Client接收到DHCP ACK报文后，检查DHCP Server分配的IP地址是否能够使用。如果可以使用，则DHCP Client成功获得IP地址并根据IP地址使用租期自动启动续延过程；如果DHCP Client发现分配的IP地址已经被使用，则DHCP Client向DHCPServer发出DHCP Decline报文，通知DHCP Server禁用这个IP地址，然后DHCP Client开始新的地址申请过程。

6. DHCP Client在成功获取IP地址后，随时可以通过发送DHCP Release报文释放自己的IP地址，DHCP Server收到DHCP Release报文后，会回收相应的IP地址并重新分配。

# ICMP Internet控制[报文](https://baike.baidu.com/item/%E6%8A%A5%E6%96%87/3164352)协议



ICMP是（Internet Control Message Protocol）Internet控制[报文](https://baike.baidu.com/item/%E6%8A%A5%E6%96%87/3164352)协议。它是[TCP/IP协议簇](https://baike.baidu.com/item/TCP%2FIP%E5%8D%8F%E8%AE%AE%E7%B0%87)的一个子协议，用于在IP[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA/455151)、[路由](https://baike.baidu.com/item/%E8%B7%AF%E7%94%B1)器之间传递控制消息。控制消息是指[网络通](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C%E9%80%9A)不通、[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA/455151)是否可达、[路由](https://baike.baidu.com/item/%E8%B7%AF%E7%94%B1/363497)是否可用等网络本身的消息。这些控制消息虽然并不传输用户数据，但是对于用户数据的传递起着重要的作用。





