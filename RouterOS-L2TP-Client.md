# MikroTik RouterOS路由器创建L2TP隧道客户端并根据IP列表进行分流

## 创建非隧道IP列表

```shell
#获取IP列表
wget http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest
#创建一个以时间命名的新文件并在其中插入位置指示命令：IP功能下的防火墙模块中的地址列表,然后在插入清空CN列表的明白
echo -e "/ip firewall address-list \n remove [/ip firewall address-list find list=CN]" > address-list_`date +"%Y%m%d"`.rsc
#找到我们需要的IP并插入到新创建的文件中
grep "|CN|ipv4" delegated-apnic-latest | awk -F'|' '{print "add address="$4"/"32-int(log(int($5))/log(2))" disabled=no list=CN"}' >> address-list_`date +"%Y%m%d"`.rsc
#删除原始下载的文件
rm delegated-apnic-latest
```

打开路由器客户端WinBox，点击左侧Files按钮将新创建的address-list_{日期}.rsc文件拖到窗体里，文件会被上传到路由器中。  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-01.png)  
打开路由器的Terminal执行：

```shell
/import file-name=address-list_20191017.rsc
```

![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-02.png)  
验证一下IP列表是否导入了，IP列表的名称是CN。
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-03.png)  

## 创建L2TP链接

![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-04.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-05.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-06.png)  
IPsec Secret是预设的共享密钥，需要设置才行，点击OK后查看是否链接成功。
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-07.png)  

## 将需要进入隧道的信息包打标记

在左侧点击IP->Firewall->Mangle->点击加号：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-08.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-09.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-10.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-11.png)  

* Chain:prerouting：是指在路由之前分析
* Src.Address:192.168.0.2-192.168.0.250：来源IP是我自己私有网段
* Dst.Address:!192.168.0.0/16：目标地址不能是私有网段
* Dst.Address List:!CN：目标IP不在名为CN的IP列表中（刚刚导入的IP列表）
* Dst.Address Type:!local：目标IP不是本地类型
* Action:mark routing：在信息包中记录某些信息，达到标记信息包的目地。
* New Routing Mark:vpn：在符合条件的信息包中加入“vpn”标记（标记的内容自己随便写）。

## 创建路由规则

在左侧点击IP->Routes->点击加号：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-12.png) 

注：要确保默认路由的distance级别低于新增加路由的distance，比如默认路由是5，此处distance设为1

* Gateway：{L2TP链接的名字}：将符合条件的信息路由到L2TP链接的内网地址这里实际的IP是10.10.10.5（在上面可以找到对应的截图）
* Routing Mark:vpn：被标记的vpn的信息包都是符合条件的，都会被路由到L2TP链接的隧道。

当建立L2TP链接后会自动创建10.10.10.1和10.10.10.5的路由：

![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-13.png)  

## 创建NAT

在左侧点击IP->Firewall->NAT->点击加号：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-14.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-16.png)  
本来想用Mark筛选NAT的信息包，但没成功，可能是因为srcnat过程在路由之后发生，这时可能不能使用Routing Mark了。

* Chain:srcnat：是指要用NAT将信息包中来源IP（私有地址192.168.0.X）替换成L2TP链接的本地地址（10.10.10.5）。
* Out.Interface:{L2TP链接的名字}：信息包在路由器上出口是L2TP隧道（NAT-srcnat在路由之后，这时已经可以知道从哪个出口出了）。
* Action:src-nat：是指要修改信息包中的来源IP。
* To Addresses:10.10.10.5：要将信息包中的来源IP修改成10.10.10.5.

## 修改L2TP链接

因为每次L2TP客户端链接成功后，本地地址会改变，上面写的10.10.10.5是动态变化的IP，所以在链接后需要更新这个IP，在左边PPP->Profile，在列表里将default-encryption复制出一份新的起名为，然后在Scripts标签的On Up输入框内输入：
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-15.png)  

```shell
:local interfaceName [/interface get $interface name]
:local newaddress $"local-address"
:local curaddress [ /ip firewall nat get [/ip firewall nat find out-interface=$interfaceName] to-addresses ]
:log info "intefaceName = $interfaceName, newadd = $newaddress, old = $curaddress"
:if ($curaddress != $newaddress) do={
    /ip firewall nat set [ /ip firewall nat find out-interface=$interfaceName ] to-address=$newaddress
}
```

修改L2TP链接的Profile选项：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-17.png)  

## 按域名配置DNS线路

点击左边IP->Firewall->Layer7 Protocols->加号：
Name随便取，Regexp内输入：google.com|twitter.com|youtube.com|ytimg.com|blogger.com|blogspot.com|wordpress.com。
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-18.png)  
然后参照前面的步骤，Routing Mark、Route、NAT，再来一遍，区别是：

* Routing Mark使用Layer7 Protocols进行标记
* Route使用的新的Routing Mark筛选
* NAT修改的是目标地址，这里可以使用Routing Mark筛选，因为NAT-dstnat在路由之前执行。

Routing Mark：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-19.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-20.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-21.png)  
Route：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-22.png)  
NAT：  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-23.png)  
![Alt text](http://static.bluersw.com/images/RouterOS/RouterOS-L2TP-Client/RouterOS-L2TP-Client-24.png)  
