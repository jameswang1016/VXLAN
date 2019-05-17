一.题目
配置一个虚拟网络，要求：
- 1. 使用帐号user@yunify.com#XXX, 密码 pass4you,
      登陆https://console.qingcloud.com
- 2. 在上海1区，建立1个vpc，创建1个vxnet, 和3个虚拟主机。
     1. 把这3个虚拟主机当作计算节点，配置虚拟网络。
     2. 通过vpc的端口转发，让你能从公网ssh到虚拟主机
- 3. 每个主机里面:
     1. 建立一个linux bridge, 使用者brctl 命令。
     2. 使用 Linux network namespace 模拟虚拟机，方法： ip link 创建 veth网卡，分别挂载到bridge和netns里面
     3. 在netns里面配置ip地址, 比如 100.0.0.2/24
     4. 创建vxlan link，并添加到bridge里面
     5. 配置vxlan规则，让netns里面的地址100.0.0.0/24可以互通
- 4. 虚拟网络要能做到:
     1. 不用组播，使用单播发送vxlan udp报文
     2. 虚拟网络支持广播和组播
     3.  arp泛洪抑制

## 二.操作说明
- 1. 登录qingcloud 略

- 2.在上海1区，建立1个vpc，创建1个vxnet,和3个虚拟主机:
其中vpc的公网ip为:139.198.188.46。3个虚拟主机分别为:
i-rmdx66r8: ip地址为192.168.0.2，通过ssh连接的端口号为22；
i-28682imo: ip地址为192.168.0.3，通过ssh连接的端口号为8888；
i-l2lpi1qh: ip地址为192.168.0.4，通过ssh连接的端口号为7654。

- 3.按照题目要求的过程在每个主机内创建了bridge,netns模拟虚拟机,veth模拟网线连接bridge与netns,为netns配置ip地址,创建vxlan并添加到bridge中。
其中,三个主机内的netns配置的ip地址分别为:100.0.0.1/24，100.0.0.2/24，100.0.0.3/24。配置vxlan规则，使主机内netns内的地址可以互通，主机内netns可以互相ping通,如图1所示：
![pic1](https://share.weiyun.com/5iTAHOC)
图1:主机1内的netns分别ping通主机2主机3内的netns
- 4.1不用组播，使用单播发送vxlan udp报文，这里采用的是手动维护vtep组的方法,在创建vxlan规则的时候不使用remote或者group参数：
ip link add vxlan10 type vxlan id 100 dstport 4789 dev eth0 nolearning proxy
并且手动添加默认的fdb表项：
bridge fdb append 00:00:00:00:00:00 dev vxlan10 dst 192.168.0.3
bridge fdb append 00:00:00:00:00:00 dev vxlan10 dst 192.168.0.4

- 4.2虚拟网络支持广播和组播，首先让它支持广播功能，需要维护fdb表项,在表项中添加mac地址为ff:ff:ff:ff:ff:ff时的fdb表，例如：
bridge fdb append ff:ff:ff:ff:ff:ff dev vxlan10 dst 192.168.0.3
bridge fdb append ff:ff:ff:ff:ff:ff dev vxlan10 dst 192.168.0.4
注：要想ping广播地址有回应必须修改内核的参数：
ip netns exec net0 sysctl -w net.ipv4.icmp_echo_ignore_broadcasts=0
广播通过ping100.0.0.255这个广播地址来实现，广播的效果如图2所示：
[!pic2](https://share.weiyun.com/5ioV8eR)
图2：主机1内的netns ping广播地址100.0.0.255
4.2中要支持组播，在每个主机内的netns内设置如下：
ip netns exec net0 route add -net 224.0.0.0 netmask 240.0.0.0 dev eth1
Ping多播地址224.0.0.1，可以ping通，如图3所示：
[!pic3](https://share.weiyun.com/5BaqQKi)
图3：主机1内的netns ping组播地址224.0.0.1

- 4.3 arp泛洪抑制，我在vxlan规则中加了一个proxy的限定，手动维护arp表，当接收到arp请求时发现arp表里有就可以直接应答，而不是之前的泛洪。配置如下：
ip neigh add 100.0.0.2 lladdr 1a:f2:73:e9:26:a3 dev vxlan10
ip neigh add 100.0.0.3 lladdr 32:c5:29:05:fc:8d dev vxlan10
注：其中1a:f2:73:e9:26:a3为虚拟主机2中netns的虚拟网卡的mac地址，32:c5:29:05:fc:8d为虚拟主机3中netns的虚拟网卡的mac地址。
