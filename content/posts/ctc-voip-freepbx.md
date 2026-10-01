---
title: "中国电信VOIP接入FreePbx系统"
date: 2022-05-25
lastmod: 2026-10-01
categories: 
  - "voip"
tags: 
  - "freepbx"
  - "voip"
  - "上海电信"
---

> **2026-10-01 更新：IMS 域名由 sh.ctcims.cn 变更为 sh2.ctcims.cn**
>
> 上海电信对 IMS 做过割接（大约在2023年底到2024年上半年之间），域名从 `sh.ctcims.cn` 换成了 `sh2.ctcims.cn`，SIP代理也相应变为 `BAC13.bq.sh2.ctcims.cn`。按本文旧版配置的中继会注册不上：旧代理地址对 REGISTER、OPTIONS 一律不回包，抓包只能看到本机不断重发，Asterisk 日志里是 `No response received`。
>
> 修改方法是把 dnsmasq 和 Trunk 配置里所有的 `sh.ctcims.cn` 换成 `sh2.ctcims.cn`，鉴权密码不用变。正文中的配置已按新域名更新，截图仍是当时旧域名的，请以文字为准。各人的代理服务器名可能不同，以光猫“宽带电话设置”里显示的为准。

大概在2019年左右，上海电信已经将所有家庭固话都已经替换为VOIP系统，也就是通过光猫的FSX口将传统PSTN系统的电话机接入电信的VOIP系统，而升级VOIP系统对于电信公司而言也是价值大于付出的。传统电话线在维护方面就是一个大头，特别是老旧小区，铜缆电话线的故障率比较高。而现在的光猫+VOIP系统大大减少线路的故障率，一个楼道就一条或两条光缆，再配置光分器就可接入整栋楼的用户。

##### 一、光猫配置

首先使用超级密码登录光猫，具体获取超级密码方面就不赘述了，网络上有很多这样的资源，这次主要关注电信VOIP系统上的设置。上海电信的VOIP系统是处于电信内网内工作而并非在公网上，光猫上就是配置在VLAN46的网络上。

在“状态”->"网络侧信息"内记录VID\_46的DNS地址，我这里是15.192.252.188

![](/images/image-20-1024x228.png)

在“网络”->“宽带设置”内的“连接名称”，选上到带VID\_46的项目，修改模式为Bridge，Vlan ID保持原有的46，服务模式选择OTHER，取消勾选左右的端口绑定，并在数据绑定列表内勾选所有端口。

![](/images/image-1024x823.png)

![](/images/image-1-1024x353.png)

这里要说明下绑定端口和绑定数据的区别，绑定端口就是绑定对应端口后，出端口带tag的数据做untag，进端口且不包含tag的数据打上tag，绑定数据就是勾选的端口会将带tag的数据直接传送出去，其实就是Cisco上的trunk口的概念。

将端口配置好后用PC网口插上光猫，DHCP拿到的应该还是光猫192.168.1.1分配的IP，如果需要直接拿到电信VOIP内网分配的地址，可以勾选绑端口到需要的端口，这样就可以在对应的端口拿到电信内网IP了。

转到"应用"->“宽带电话设置”，记录SIP外部代理服务器地址，用户号码，鉴权用户名，鉴权密码等。鉴权密码要看光猫型号，有些可以通过F12获取，有些需要telnet登录光猫获取。

![](/images/image-17-1024x935.png)

##### 二、交换机配置

当前环境使用的是MikroTik的交换机（其他品牌设备可以以此类推），先到VLANs里面创建VLAN46接入电信内网，VLAN100用于对接光猫本体没有tag的数据以方便配置，VLAN85对应IPTV则不在本次讨论范围。

![](/images/image-2-1024x172.png)

转到VLAN，把需要对接光猫的端口的VLAN号改成100（根据前面创建的VLAN号配置），VLAN Receive改为any，这样会将光猫不带tag数据传到交换机的VLAN100内，带tag的数据则传到交换机上已经创建过对应VLAN ID的VLAN内。

![](/images/image-3-1024x460.png)

##### 三、FreePbx环境配置

这次是在QNAP NAS的虚拟机上安装FreePBX，并将电信VOIP内网和家庭内网一并接入FreePbx系统，方便FreePBX系统配置和安装更新等。

1、在NAS上打开网络和虚拟交换机，转到“Network”->“Interface”，选择一个不用的接口，点Add VLAN，输入VLAN ID 46后保存。

![](/images/image-4-1024x786.png)

实际配置好后的接口可以参考下图，可以看到接口已经通过DHCP获取到了地址。

![](/images/image-5-1024x266.png)

2、转到“Virtual Switch”，点“Add”新建一个Virtual Switch，选择"Advance Mode“

![](/images/image-6-1024x903.png)

在Physical Adapter内勾选前面设定过VLAN的Adapter，然后点Next。

![](/images/image-7-1024x916.png)

单选Do Not assign IP Address，不需要给虚拟交换机分配IP。

![](/images/image-8-1024x918.png)

不勾选Enable NAT和Enable DHCP Server

![](/images/image-9-1024x908.png)

后续直接Next到最后确认界面，点Apply完成配置。

![](/images/image-10-1024x909.png)

3、新建虚拟机并安装系统

虚拟按照一般设定进行建立和安装，安装系统方面就不讲述了，这里我是安装的是FreePBX16 Asterisk19，安装完成后设定系统为固定IP，方便后续SIP Phone接入。

在虚拟机系统安装完成后，再进入设定界面修改配置。在“网路”选单内可以看到默认建立的网卡，绑定到接入家庭内网的虚拟交换机。

![](/images/image-11-1024x514.png)

我们点“新增装置”，选择网路，类别选VirtIO，虚拟交换机选择前期设定VLAN46的虚拟交换机，这样我们虚拟机就加载了两张网卡，一张是对应家庭内网，另一张对应电信VOIP专网。

![](/images/image-12-1024x576.png)

到FreePBX内我们将新新添加网卡设定为DHCP，SSH登录系统输入ip addr，可以发现系统已经从电信VOIP内网获取到了IP地址

![](/images/image-13-1024x344.png)

这时候会如果尝试ping外网会发现网路不通，在ip ro后发现默认路由表已经从原先eth0变为eth1，默认出口变为电信内网。这个问题主要是因为VOIP内网是在eth1接口，且设定的是自动获取IP，在获取到IP后会冲掉eth0设定的默认网关。

![](/images/image-14-1024x197.png)

修改 /etc/sysconfig/network 添加 GATEWAY=xx.xx.xx.xx (家庭内网接口的网关地址)

![](/images/image-15.png)

systemctl restart network 重启网卡，这时候ip ro就可以发现默认网关已经恢复我们需要的设定，并可以ping通外网。

4、FreePBX系统设定

在上述系统接口设定完毕后，我们先nslookup我们的SIP代理服务器地址（我这里是bac13.bq.sh2.ctcims.cn），会返回找不到结果，问题是在查询dns用的是家庭内网dns(10.11.0.1)，内网dns是没有这个域名解析结果的，需要通过VOIP内网的dhcp获取的dns地址进行查询才可以获取有效数据。

![](/images/image-18-1024x258.png)

在/etc/resolv.conf，修改 nameserver后地址为127.0.0.1 我们需要将系统的dns查询服务器设定为本地dnsmasaq，让dnsmasq来处理查询问题。

![](/images/image-19-1024x242.png)

systemctl start dnsmasq 开启dnsmasq服务， systemctl enable dnsmasq 设定dnsmasq开机自动启动。

修改 /etc/dnsmasq.conf ，在文件末尾添加以下设定。

```
# 使用10.11.0.1作为默认DNS地址
server=10.11.0.1 
# SIP代理台服务器通过指定DNS服务器查询解析
server=/BAC13.bq.sh2.ctcims.cn/15.192.252.188    
server=/BAC13.bq.sh2.ctcims.cn/15.192.251.188
```

![](/images/image-21-1024x524.png)

保存后 systemctl retsrat dnsmasq 重启dnsmaqs即可恢复dns解析

接着修改/etc/sysconfig/network-scripts/route-eth1 添加 15.192.0.0/16 via 28.67.255.254 dev eth1 (指定15.192.0.0网段的出口路由)

![](/images/image-22.png)

保存后systemctl restart network

再次nslookup我们的SIP代理服务器地址，会发现已经有返回了

![](/images/image-23-1024x357.png)

##### 四、FreePBX配置

在网络和系统都配置好的基础上，可以进行FreePBX的trunk配置，将固话账号配置到本地SIP交换系统内。

在"Connectivity"->"Trunks"内点击”Add Trunk“->"chan\_pjsip"，Trunk Name随意填写

![](/images/image-25-2048x1036-1-1024x518.png)

转到”pjsip Settings“->"General"

```
Username=+8621xxxxxxxx
Auth username=+8621xxxxxxxx@sh2.ctcims.cn
Secret=xxxxxxxxxx (我这里是20个字符)
Authentication=Outbound
Registration=Send
SIP Server=sh2.ctcims.cn
SIP Server Port=5060
```

![](/images/image-26-1024x692.png)

转到”pjsip Settings“->"Advanced"

```
Outbound Proxy=sip:SIP_PROXY地址:5060\;lr (注意格式，前后都需要)
From Domain=sh2.ctcims.cn
From User=+8621xxxxxxxx
Client URI=sip:+8621xxxxxxxx@sh2.ctcims.cn:5060
AOR=sip:+8621xxxxxxxx@sh2.ctcims.cn:5060
Match (Permit)=15.192.0.0/16 (根据实际VOIP内网网段)
```

![](/images/image-27-1024x831.png)

Submit后，可以在"Report"->"Asterisk Info" 内看到Trunk的注册情况

![](/images/image-28-1024x356.png)

后续再配置出入局路由和分机，就可以完美使用拉。

##### 五、调试和总结

1、在前期修改光猫配置中，我们可以将VLAN46设定端口绑定，直接将PC接入到绑定的端口，测试是否可以DHCP获取到IP。

2、在获取到VOIP内网的PC上使用MicroSIP测试SIP账号和SIPProxy是否正确，是否可以正常呼入和呼出。

3、SIP Proxy地址一般设定了防Ping，直接ping地址是不会回包的，使用nslookup测试dns解析，使用traceroute测试SIP Proxy路由，检查路由是否从对应VLAN46端口出本地。

4、SIP在注册时候，使用的是sip:+8621xxxxxxxx@sh2.ctcims.cn:5060 的request格式向服务器发包注册，若不填写SIP Proxy，则发送请求到sh2.ctcims.cn这个服务器地址，若填写SIP Proxy，则发送请求到SIP Proxy的服务器地址。在电信VOIP内网中，是解析不出sh2.ctcims.cn这个地址的，所以SIP Server这个必须正确填写，否则无法注册。

![](/images/image-29-1024x525.png)

5、在SIP trunk注册不上，可以使用tcp dump抓包，检查sip在注册过程中是否有问题。

6、（2026-10-01补充）如果抓包发现 REGISTER 发出去后完全没有回包，先用VOIP内网的DNS确认代理服务器名是否还有效，例如 nslookup BAC13.bq.sh2.ctcims.cn 15.192.252.188 。正常的代理服务器对未带鉴权的 REGISTER 会回 401，对 OPTIONS 会回 200；已经下线的旧地址则什么都不回。电信VOIP内网的DNS对不存在的域名也是不应答（超时），而不是返回找不到。

最后感谢 [“](http://zhmail.com/2016/10/14/freepbx-13-ctc-sip-account-trunk-setting)[文卓的笔记-中国电信SIP账号在FreePBX 13中的中继设置](http://zhmail.com/2016/10/14/freepbx-13-ctc-sip-account-trunk-setting/)[”](http://zhmail.com/2016/10/14/freepbx-13-ctc-sip-account-trunk-setting) 给我带来不少思路，特别是SIP Proxy方面的问题
