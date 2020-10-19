chap0x01 
**基于VirtualBox的网络攻防基础环境搭建实例讲解**

**实验目的**
* 掌握 VirtualBox 虚拟机的安装与使用；
* 掌握 VirtualBox 的虚拟网络类型和按需配置；
* 掌握 VirtualBox 的虚拟硬盘多重加载；


**实验环境**: 
```
Victim: kali-linux-2018.3-amd64.iso   
Gateway: kali-linux-2018.3-amd64.iso   
Attacker: kali-linux-2018.3-amd64.iso   
```

**实验步骤**：

1.配置一块安装了kali的 .vdi 硬盘多重加载，并在virtualbox中新建三台虚拟机，分别设置为Victim、Gateway、Attacker。

2.分别给三台虚拟机添加网卡实现要求的功能。

3.开启 **Gateway** ipv4转发功能，添加 **Gateway** 防火墙NAT规则。


**具体步骤说明**：

   一、配置一块安装了kali的 .vdi 硬盘多重加载，并在virtualbox中新建三台虚拟机，分别设置为Victim、Gateway、Attacker。
   
        首先释放对应的  .vdi 硬盘，然后将该硬盘的类型改为多重加载，然后新建虚拟机的时候选用多重加载的虚拟硬盘。
   ![1](/i/1.png)
   
   二、分别给三台虚拟机添加网卡
   
   首先给三台虚拟机添加网卡，网卡类型如下

       Victim：internal  networking mode (eth0)
       Gateway：internal  networking mode (eth0)、Network Address Translation mode(eth1)
       Attacker：Network Address Translation(eth1)


对三台虚拟机网卡的配置如下：

```
 Victim：
      eth0   ip: 192.168.1.2   gateway:  192.168.1.1  (内部网卡)
 Gateway：
      eth0   ip: 192.168.1.1        (内部网卡)
      eth1   ip: 10.0.2.4           (NAT 网卡)
 Attacker：
      eth0   ip: 10.0.2.5           (NAT 网卡)
```

配置成功截图：

Victim

![3](/i/3.PNG)


Gateway

![4](/i/4.PNG)

Attacker

![5](/i/5.PNG)




   
   三、开启 **Gateway** ipv4转发功能，添加**Gateway**防火墙NAT规则，设置网关转发局域网192.168.1.0/24中的数据包。
   
       默认情况下，linux的三层包转发功能是关闭的，所以网关如果收到目的地址不是本机网卡ip的时候，会直接将数据包丢弃。如果要让**Gateway**实现转发，需要改变 Gateway 的一个系统参数以打开 ipv4 转发功能。


         使用以下三条指令都可以打开
          #方法1
          echo 1 > /proc/sys/net/ipv4/ip_forward

          #方法2
          sysctl net.ipv4.ip_forward=1

          #方法3
          #编辑 vim /etc/sysctl.conf  
          #添加一条语句 net.ipv4.ip_forward =1
           sysctl -p

   
   添加网关防火墙NAT规则
   

```
iptables -t nat -A POSTROUTING　-s 192.168.1.0/24 -o eth1 -j  MASQUERADE
```
配置对应的网卡开启和关闭防火墙规则，保存现有的iptables 防火墙NAT规则。
```
cat << EOF >  /etc/network/if-pre-up.d/firewall
#!/bin/sh
/usr/sbin/iptables-restore  <  /etc/iptables.rules
exit 0
EOF
chmod +x  /etc/network/if-pre-up.d/firewall


cat << EOF >  /etc/network/if-post-down.d/firewall
#!/bin/sh
/usr/sbin/iptables-save  -c  >  /etc/iptables.rules
exit 0
EOF
chmod +x  /etc/network/if-post-down.d/firewall

```


添加完成后，查看路由规则已经被配置好，截图如下：
![6](/i/7.PNG)


   
  

连通性验证截图如下:

 -  靶机可以直接访问攻击者主机
 
  ![8](/i/靶机到攻击者.PNG)
-  靶机可以上网

![8.5](/i/靶机上网.png)

 -  攻击者主机无法直接访问靶机
 
![9](/i/攻击者到靶机.PNG)
 
 - 网关可以直接访问攻击者主机和靶机
 
 网关到靶机
 ![10](/i/网关到靶机.PNG)

网关到攻击者
![11](/i/网关到攻击者.png)

 - 靶机的所有对外上下行流量必须经过网关
 
     靶机的网关ip 被设置为Gateway  eth0 的ip，靶机本身不能上网，靶机只能经由 Gateway 对外访问。
     
 -  所有节点均可以访问互联网
 
     Victim 可以通过 Gateway 上网。
     Gateway  自身具有 NAT 网卡，可通过主机网络上网。
     Attacker  自身具有 NAT 网卡，可通过主机网络上网。
     
 


我查阅的资料

[iptables讲解](http://blog.51cto.com/wwdhks/1154032)

[iptables使用手册](http://ipset.netfilter.org/iptables.man.html)

