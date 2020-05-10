Two key network-core functions

​	routing

​	forwarding

Alternative core: circuit switching

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200429211649.png)

four sources of packet delay

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200429211926.png)

securty

​	packet sniffing:broadcast media 

​	ip spoofing: send packet with false source address



Application architectures

​	client-server
​	peer-to-peer (P2P)

​	TCP service:
reliable transport between sending and receiving process
flow control: sender won’t overwhelm receiver 
congestion control: throttle sender when network overloaded
does not provide: timing, minimum throughput guarantee, security
connection-oriented: setup required between client and server processes

UDP service:
unreliable data transfer between sending and receiving process
does not provide: reliability, flow control, congestion control, timing, throughput guarantee, security, or connection setup, 

Q: why bother?  Why is there a UDP?



securing tcp



web and http

​	non-persistent HTTP

​	persistent HTTP





---

### 国立清华大学

domain name(domain name server) -> ip address :  about 6 message to find ip address

3 message for connection establishment of tcp

4 

4 



point-to-point  multiple access

peer-to-peer



spanning tree：任何两个结点之间只有一条路径



封包最大值 每个网路不一样



statistical multiplexing:fifo rr priorities





bandwidth 

latency

