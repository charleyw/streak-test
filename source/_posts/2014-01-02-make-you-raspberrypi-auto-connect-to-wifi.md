---
layout: post
title: "让你的树莓派连上WiFi"
date: 2014-01-02 13:02:02 +0800
comments: true
categories: raspberrypi
---

周一的时候树莓派总算是到手了，很早之前就了解过了，心里长草很多年了，但就是一直没出手。最近在搞Arduino的小玩意，我们做的这个东西需要网络通信（一个可以远程控制的机器人小车），总是需要借助上位机（一台android手机）的网络来接受命令。然后就突然想起了这货，[树莓派](http://zh.wikipedia.org/wiki/%E6%A0%91%E8%8E%93%E6%B4%BE)是**基于linux**的只有信用卡的大小计算机。你可以把这货当成一个正常linux服务器就是，基本上你平时在linux上能做到的，它都能做到，比如当成rails服务器，在上面运行rails程序什么的(rails我没试，不过确实是可以的，我是忠实的sinatra爱好者)。然后这货有usb口，再然后插上你在某宝买的usb无线网卡，它就可以用WiFi了。下面是我第一次，第二次以及第n次连上WiFi的过程。
### 第一次连上WiFi
第一次连wifi之前，你需要做一件事情，打开树莓派的terminal，你有两种选择：

* 通过HDMI连个显示，并且接个键盘，然后你就可以像用一台普通的pc一样用树莓派了
* 插个网线，然后通过树莓派的ip地址ssh进去。

		ssh pi@your_raspi_ip
        #password: raspberry
        
我是通过插网线的方式进去的，这个方法比较麻烦的地方是，你得去找到树莓派的获取到ip。我是在自己家连好后，从路由器的客户端列表里面找到树莓派的ip的，后面会讲到如何破。

在进到terminal后，你就可以开始安装软件，修改配置了

1. 可能需要安装的软件（因为我拿到手的时候，发现系统里已经有了，不知道是某宝的亲帮我装的，还是raspbian已经预装了）

		sudo apt-get install wireless-tools
2. 然后可以开始配置网络了，修改**/etc/network/interfaces**文件，把它修改成这个

        auto lo
        iface lo inet loopback

        auto eth0
        iface eth0 inet dhcp

        allow-hotplug wlan0
        auto wlan0
        iface wlan0 inet dhcp
		    wpa-ssid YOUR-SSID-HERE
		    wpa-psk YOUR-PASSWORD-HERE
主要是添加wpa-ssid和wpa-psk，直接把你要连接的wifi的ssid和对应密码写上就行了。
3. 重启网络

		/etc/init.d/networking restart
        # or: service networking restart
然后你应该就已经连上wifi了，如果没有连上：
    1. 检查时候你要连接的wifi是不是隐藏的wifi(不广播自己的ssid)，通过下面的命令检查你要连的wifi是不是在列表里，这种配置方法没办法连接隐藏的wifi
    		iwlist wlan0 scan
	2. 检查你的ssid和密码是否正确
    3. 检查你要连的wifi网络是否正常，检查usb wifi是不是正常
    3. 如果还连不上就google，我也帮不了你了

这样的配置在你重新启动树莓派后也能自动连接这个WiFi，这里连接WiFi使用的是wpa_supplicant
### 自动连接wifi多个WiFi网络（怎么设置wpa_supp）
当你经常切换到不同WiFi网络中时，你可以配置多个WiFi网络，让树莓派能自动连接到可用WiFi网络中。这里就要用到高大上的wpa_supplicant.conf了

1. 修改**/etc/wpa\_supplicant/wpa\_supplicant.conf**，下面是我使用的配置文件：

        ctrl_interface=/var/run/wpa_supplicant
        #ap_scan=1

        network={
               ssid="wo_shi_yige_wifi_ssid"
               scan_ssid=1
               psk="wo_shi_mi_ma"
               priority=5
        }

        network={
               ssid="pi"
               psk="onlyforpi"
               priority=1
        }
	* **ap_scan:**1是默认值，因此我注掉了
		* **1：**这个模式下总是先连接可见的WiFi，如果扫描完所有可见的网络之后都没有连接上，则开始连接隐藏WiFi。
    	* **2：**会按照network定义的顺序连接WiFi网络，遇到隐藏的将立刻开始连接，因此在这个模式下连接顺序不受priority影响
	* **ctrl_interface:**这个文件夹里面存的是一个当前使用的interface的socket文件，可以供其他程序使用读取WiFi状态信息
	* **network：**是一个连接一个WiFi网络的配置，可以有多个，wpa_supplicant会按照priority指定的优先级（数字越大越先连接）来连接，当然，在这个列表里面隐藏WiFi不受priority的影响，隐藏WiFi总是在可见WiFi不能连接时才开始连接。
		* **ssid**:网络的ssid
	    * **psk**:密码
	    * **priority**:连接优先级，越大越优先
    	* **scan_ssid:连接隐藏WiFi时需要指定该值为1**
1. 修改/etc/network/interfaces使用wpa_supplicant.conf来配置无线网络

		auto lo
		iface lo inet loopback
		
		auto eth0
		iface eth0 inet dhcp

		allow-hotplug wlan0
		auto wlan0
		iface wlan0 inet dhcp
			pre-up wpa_supplicant -Dwext -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf -B	
以后每次启动时，树莓派都会主动去连接配置文件中预定义的这些wifi网络。  

在这个配置里面有一个**ssid='pi'**网络，这是一个最低优先级网络，用来在陌生网络中配置树莓派的。当处在一个树莓派配置里面的没有的WiFi网络中时，我会自己创建一个叫pi的WiFi让树莓派连接进来，然后我便可以ssh进树莓派，添加网络配置，然后重启，就可以让树莓派加入到新的网络中。


### 保证任何时候可用（怎么上报自己的IP）
那么如何在你的树莓派加入新的网络后获取到它当前的ip地址呢？因为你在重新配置树莓派的网络并重启后，你跟树莓派的连接会断掉，因此你需要知道树莓派在新网络中的ip，从而使你能重新连接到树莓派。在网上很多免费提供的域名解析服务，你可以某个域名解析成你设置的ip地址。我是用的方案就是在每次树莓派启动后都会更新自己的域名对应的ip，我是用的[DNSDynamic](https://www.dnsdynamic.org)提供的服务。

1. 注册账号~~~~
2. 设置一个启动脚本来获取本机ip并且更新到DNSDynamic上：
	* 修改/etc/rc.local，添加如下内容：	
{% codeblock lang:bash %}	
IP=`hostname -I`
EMAIL=your_username_in_dnsdynamic
PASSWORD=your_password
DOMAIN=your_registered_domain.dnsdynamic.com
curl -v --user "$EMAIL:$PASSWORD" -k "https://www.dnsdynamic.org/api/?hostname=$DOMAIN&myip=$IP" > /var/log/update-dns.log 2>&1
{% endcodeblock %}

脚本后面的内容是调用dnsdynamic提供的api更新域名对应的ip地址

每一次树莓派启动之后都会执行这个脚本更新自己的ip地址，也可以将这段脚添加到cron job里定时更新ip,但是感觉好像没有必要。

3. 之后你就不用管ip地址了，可以通过域名直接ssh进树莓派：

		ssh pi@your_registered_domain.dnsdynamic.com

### 总结
上面的提供的方案其实一定程度依赖于网络（internet），如果树莓派连接到的wifi是没有internet连接的，那么就没办法通过dnsdynamic更新ip了，那么我们也就没有办法获取到它当前的ip，除非它使用静态ip.
在failover的网络（上面设置的名叫pi的wifi）设置上也可以通过另一个方式，就是在树莓派启动之后可以自己开启一个wifi AP，然后我们可以连接进去，进而做各种设置，arduino最新的板子arduino yun就是通过这种方式进行设置的。