# CNSS Recruit 2019 Web WriteUp


> 由于技术有限，时间匆忙，本文可能存在各种问题。如发现问题（如思路错误、术语错误 或 我的解法凭空增加了解题难度 等），请各位 dalao 毫不犹豫的锤爆我的狗头，谢谢！
>
> 感谢 **ay3** 对 `产品经理` 一题解题思路的指正，感谢 dalao！

## check_in

----

web 的第一道签到题，打开就说了 hide，果断 `F12` 看源码，发现注释里有些东西

```html
<!--what is this : ./secret/f1ag_is_h4re.php-->
```

直接在url后添加 `/secret/f1ag_is_h4re.php` 进入~~下一关~~，这里再看源码并无收获，看一下 Cookie，发现多了一条名为 `interesting_string` 的 Cookie，查看其内容如下

```
Y25zc3t3ZWxjb21lX3RvX3dlYl9XMHJsZCF9
```

猜测为某种编码，尝试 base64 解码，成功获取 flag



## True_check_in

----

真正的签到题？打开发现是一个~~数学题~~，果断打开计算器算出正确答案

```
8589934592
```

输入答案时发现问题，输入框有长度限制，看源码，发现最大长度为 8，而我们的答案有十位，但这里长度限制在前端，打开 Burp Suite 一通抓包改包（也可审查元素直接改maxlength），输入正确答案，成功获取 flag



## warm_up

----

来热个身！看到题目即想到修改 `header`，这里用到了 `User-Agent`  、 `X-Forwarded-For` 和 `Referer`

首先在 `User-Agent` 中添加 cnss，或直接将其中浏览器名称改为 cnss

PS：这里应注意，不要将 UA 内容全部删除只写入 cnss，这样有可能服务器无法解析而拿不到 flag

然后添加下面两句

```
X-Forwarded-For: 233.233.233.233
Referer: www.notcnss.com
```

将包发出，成功获取 flag



## GAY' Profile

----

~~一眼望去，好GAY的题~~首先看源码，发现如下注释

``` html
<!-- Why not try to get /source ? -->
```

果断 url 后加 `/source`，拿到源码，发现在 `/file` 中通过name参数可以读取文件，结合 hint 中所说 flag 在 `/flag` 中构造 url

```
/file?name=../flag
```

别问我为什么 flag 在上一级目录，试一试就出来了（逃，成功获取 flag



## love_reading

----

拿到这道题，只给了两句话，乍一看比较懵

```
Evil robot take my love. I only know that he uses linux. 
```

细细分析这两句话中的提示，发现 `robot` 和 `linux` 应该有作用

先尝试在 url 后添加 `/robots.txt`， 出现了 `s4cret.php`，直接修改 url，~~惊喜的~~发现一片空白

读 hint 发现 `vim in linux` ，眼前一亮，即想到 vim 产生的临时文件

分别尝试 `/.s4cret.php.swp` 和 `/.index.php.swp`，发现 index.php 存在临时文件，用 vim 还原，得到源码

```
if(!(isset($_GET['key']) && isset($_GET['f']))){
    die();
}
else{
    $path = $_GET['key'];
    $data = file_get_contents($path);
    if($data != "zhi ma kai men!"){
        die("no no nope");
    }
    $f = $_GET['f'];
    include($f);
}
```

源码不长，发现有两个参数 `key` 和 `f`，分别调用 `file_get_contents()` 和 ` include()`，应从这两个地方入手，由于没有任何过滤机制，故直接考虑 PHP 伪协议

第一个参数应用自己的输入代替获取文件内容，可用 `php://input` 加 POST 传参直接绕过

第二个参数应为 `include()` 的任意文件读取，用 `php://filter` 即可

故构造以下url

```
/?key=php://input&f=php://filter/convert.base64-encode/resource=s4cret.php
```

然后 Burp Suite 抓包，请求方法改为 `POST`，请求体添加 zhi ma kai men!，获得一段 base64 编码的文本（或直接采用 HackBar 发送 POST 请求），解码后可看到一段神奇的注释，成功获得flag



## Gay Profile Plus

----

~~继续 gay~~看源码，继续 `/source`，发现加了一段神奇的语句

```python
if os.path.abspath(filename).startswith(os.getcwd()) and filename != './profile':
    return 'No No No', 422
```

这里大概意思是，如果要读取的文件在当前工作目录，那么它必须是 `./profile`，但是题目告诉我们 flag 在 `./flag` 中，显然直接读取就行不通了，由抓包可知服务器为 Linux，可考虑 `/proc` 文件系统

> proc文件系统是一个伪文件系统，它只存在内存当中，而不占用外存空间。它以文件系统的方式为访问系统内核数据的操作提供接口。用户和应用程序可以通过proc得到系统的信息，并可以改变内核的某些参数。由于系统的信息，如进程，是动态改变的，所以用户或应用程序读取proc文件时，proc文件系统是动态从系统内核读出所需信息并提交的。（源自网络）

一般来说 `/proc` 目录下有以 PID 命名的目录，可以通过此目录获取进程所在目录的文件，但这里显然无法获取 PID，但源码中 `os.getcwd()` 提醒我们要用当前工作目录，于是构造以下 url

```
/file?name=/proc/self/cwd/flag
```

成功获得 flag



## baby_sql

----

作为 sqli 的第一题，本题并不难，未进行任何过滤，但添加了 `code` 参数进行验证，防止了直接 SqlMap，但只需进行简单 union 查询注入即可，但应注意，此处仅显示查询的第一条结果，故查询时应给予一个无内容的 `id` 

依次查询库名、表名、字段名，直接查询即可，成功获得 flag

PS：可采用 Python 脚本自动输入 `code` 参数，缩短操作时间（另三道 sqli 同理）



## 蟒蛇-Login_in

----

打开发现一个登录界面，需要提供密码和验证码，且知道验证码MD5后的前三位，看源码，有以下注释
```html
<!--验证码一共4位数字字母，密码都是数字，验证码正确你就知道密码位数了-->
```
那就先写一个计算验证码的 python 脚本
得到正确的验证码并输入后，得知密码为4位数字，由于没有任何其他提示，这里选择爆破法，只需从 `0000` 试到 `9999` 即可，将上面的脚本进行扩充（由于验证码随机，注意使用 `session`，且注意 `requests` 获取网页的编码格式）
输入获取到的正确密码和验证码，成功获得 flag



## Gay Profile Plus Plus

----

既然题干已经说了放在文件不安全，那就不可能直接从本地文件获取 flag 了，那如何把一个东西存储在文件之外呢？这里我们继续考虑 gay+ 的时候用过的 `/proc` 文件系统，因为它存在于内存中，当然就不是文件了
查看 `/proc` 文件系统的具体说明，发现在进程目录下存在一个环境变量，猜测 flag 可能存在于此，故构造 url 如下
```
/file?name=/proc/self/environ
```
发现 flag 果然在此，成功获得 flag

## 产品经理

----

查看源码，在 `db.php` 的注释中发现以下 sql 语句
```sql
...
name char(64)
...
INSERT INTO products VALUES('cnss', sha256(....), 'FLAG_HERE');
...
```
得知 `name` 的长度最大为 64 ， 而我们需要获取 `name` 为 `cnss`  处的 `description`
因题干说了使用了 PDO ，首先排除sql注入

观察源码，发现 `get_product` 和 `check_name_secret` 分离

```
function get_product($name) {
  global $db;
  $statement = $db->prepare(
    "SELECT name, description FROM products WHERE name = ?"
  );
  ...}
function check_name_secret($name, $secret) {
  global $db;
  $valid = false;
  $statement = $db->prepare(
    "SELECT name FROM products WHERE name = ? AND secret = ?"
  );
  ...}
```

即验证 `name` 和 `secret` 和获取信息并非同步进行，而获取信息时并不需要 `secret`，考虑到 Mysql 对尾空格的处理，`add` 以下内容

```
Name: cnss + 任意空格
Secret: 符合要求即可
Description: 任意
```

执行 `add` 操作后，`view` 以下内容

```
Name: cnss
Secret: 上一步设置的密码
```

执行 `view` 操作，成功获得 flag

## easy_sql

----

经过试验，发现许多字符被屏蔽，比较重要的有（都有方法绕过或不使用）
```
and  -  *  空格
```
其中 `空格` 可用 `%0a` 代替，注释符号 `--` 可用 `%23` 代替
由于 `and`  不可用，这里考虑报错注入，且由于 `*` 不可用，考虑 xml 的报错，经测试，xml 的报错注入可用，故构造如下 payload
```
1%27%0aor%0a(extractvalue(1,concat(0x7e,(......),0x7e)))%23
```
只需将 `......` 处修改为查询语句，即可在报错中看到查询结果
依次查询库名、表名、字段名，成功获得 flag

## True_Love

----

不得不说，出题人真的有心了 ~~但是先别看这个心了看题吧~~
从第一个界面目测找不到什么有用信息，不妨打开题目所说的 `/admin` ，看到一个登陆界面，且第一行就说
```
只有admin才能拿到flag
```
那就让我们成为 admin 不就好了
查看 cookie 发现一条 session ， base64解码，发现第一段为用户名，考虑 session 伪造，而现在的问题在于不知道密钥，那就先拿到密钥
随便修改 `/` 后的内容，发现 `404` 界面中有出错的 url ，由于后端为 python ，尝试模板注入，先构造 `{{1+1}}` 得到结果 `2` ，模板构造漏洞存在，直接构造 `{{config}}` ，即可得到密钥
通过 flask 框架构造 session ，替换本题网页中的 session，刷新页面即可发现 flag 出现在 url 中，成功获得 flag

## normal_sql

----

本题验证码变成了给定其md5的前四位，无非是增加了些脚本行数和运行时间，问题不大
依旧过滤了众多字符，且并无错误信息，故考虑延时盲注，构造如下 payload
```
1'and(if(ascii(substr((select(...)from(...)),index,1))>0,1,sleep(3)))%23
```
修改 `...` `index` 和 `0` 处的值，通过网页加载时间，判断盲注结果
依次查询库名、表名、字段名，成功获得 flag

## Gay Profile Plus Plus Plus

----

emmm...ts后端可还行
源码提供了那就直接康康，发现在 `/file` 中有以下内容
```typescript
if (!file && !ctx.__proto__.name) {
        ctx.body = 'No No No...';
        ctx.status = 422;
        return;
    } else if (file && file.indexOf('flag') !== -1){
        ctx.body = 'No No No...';
        ctx.status = 403;
        return;
    } else if (!file){
        file = ctx.__proto__.name;
    }
```
大概一看我们就会发现，flag 应该是放在一个文件中，而你不能通过这个文件名读取它，但是可以通过 `ctx.__proto__.name` 继承 `name` 属性，所以考虑原型链污染
再看 `/save` 的部分，可以发现输入数据会通过 `JSON.parse` 解析，这样就为我们进行原型链污染提供了可能，故构造以下 `JSON字符串`
```JSON
{"type": "hhh", "content": {"constructor": {"prototype": {"name": filename}}}}
```
只需修改 `filename` 的值提交，再在 `/file` 处将 `name` 留空，即可读取任意文件，首先尝试 `flag` 无果，再尝试 `../flag` ，成功获得 flag

## hard_sql

----

题干直接告诉我们 `Make it wrong` ，首先考虑的是报错注入，但是错误信息根本没有回显，出错只会有 `Something wrong` 的提示
这里可以考虑让其在特定情况下报错，首先考虑 `if` ，发现被过滤，于是使用 `case when` 的用法
构造以下 payload
```sql
1' and case (ascii(substr((something),index,1))) when asc then pow(999,999) end %23
```
这里注意 `select` 和 `information` 会被过滤，但只需通过大小写变化绕过即可
上述语句在查询的字符的 ascii 等于给出的数字时，才会执行 `pow(999,999)` ，此时才会产生报错
通过修改 `something` `index` 和 `asc` 可以爆出库名和表名，但当尝试爆出字段名的时候发现 `column` 被过滤且无法绕过，所以尝试无列名注入（虽然我也不懂为什么无列名注入的题只有一列，但是还是这么做了）,将 `something` 改为如下 payload 即可进行无列名注入
```sql
Select `1` from (Select 1 union select * from database_name.table_name) as a limit 1,1
```
依次爆出 flag 的每一位即可，成功获得 flag

## [BOSS] Get Me Inside

----

[Get ♂ Me ♂ Inside](/2019/10/24/cnss-recruit-2019-web-boss/)
