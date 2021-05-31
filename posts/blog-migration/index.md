# 博客迁移记




> 前几天猛然发现自己的博客已经好多天没更新了，翻了一圈发现博客源码也被我弄丢了，由于不想再配一遍 Hexo 和依赖的各种库（~~其实就是懒~~，索性直接把博客迁移到了 [Hugo](https://gohugo.io/)

## 0x00 安装 Hugo

Windows 用户只能说 [Scoop](https://scoop.sh/) 真香

```powershell
scoop install hugo-extended
```

至于为什么要安装 extened 版本，其实我最开始装的是另一个版本，然后发现好多主题都用不了，比如普通版本就没有 scss 支持，Google 了一圈才发现需要装 extended

顺带一提，原来 Hugo 的 Stars 已经超过 Jekyll 了

## 0x01 选主题

众所周知，搭博客最难的一步是选一个好看的主题（曾经因为没找到好看的主题而放弃了 Wordpress

把官方的主题列表翻了个底朝天，最后选了这个最符合我审美的 [LoveIt](https://github.com/dillonzq/LoveIt)，国人开发的，还是中文文档看着舒服

简单看了一下介绍，功能挺齐全的，搜索评论这些功能都是开箱即用，还记得之前 Hexo 那个主题为了加个搜索功能自己改了半天模板

主题的文档写的挺全面的，各个配置项都有明确的介绍，用到什么就到文档里搜索一下就好，这里说两个文档里没有涉及到的问题

### Algolia Search 的配置

在博客里配置基于 [Algolia](https://www.algolia.com/) 的搜索功能只需要在配置文件里加几个相关的配置项就可，这里说一下将生成的 json 上传到 Algolia 以应用搜索的操作

这里博客的作者推荐了一个基于 nodejs 的库 [Algolia Atomic](https://github.com/chrisdmacrae/atomic-algolia)，由于懒得配 nodejs 而且 Algolia 官方 API 其实非常完善，我就直接用 python 糊了一个小脚本，由于我把博客的部署直接放到了 Github Action 上，所以就约等于每次部署都自动上传了，这里就随便放一下糊出来的脚本

```python
from algoliasearch.search_client import SearchClient
import json
import os
import sys


def read_json(filename: str) -> list:
    with open(filename, 'r', encoding='utf-8') as file:
        return json.load(file)


def main():
    if len(sys.argv) != 2:
        print("Filename is needed.")
        exit(1)
    filename = sys.argv[1]

    ALGOLIA_APPID = os.getenv('ALGOLIA_APPID')
    ALGOLIA_ADMIN_KEY = os.getenv('ALGOLIA_ADMIN_KEY')
    ALGOLIA_INDEX = os.getenv('ALGOLIA_INDEX')
    if ALGOLIA_APPID == None or ALGOLIA_ADMIN_KEY == None or ALGOLIA_INDEX == None:
        print("APPID and ADMIN_KEY and INDEX is needed.")
        exit(1)
    
    client = SearchClient.create(ALGOLIA_APPID, ALGOLIA_ADMIN_KEY)
    index = client.init_index(ALGOLIA_INDEX)

    print("=== Clear Objects ===")
    res = index.clear_objects()
    print(res.raw_responses)

    objects = read_json(filename)
    print("=== Add Objects ===")
    res = index.save_objects(objects, {'autoGenerateObjectIDIfNotExist': True})
    print(res.raw_responses)


if __name__ == '__main__':
    main()
```

### 文章的加密

~~有的时候写的博客不想被别人看到怎么办呢，加密一下就好了~~

文章的加密对于基于 nodejs 这种脚本语言的 Hexo 来说简直轻而易举，npm 装个插件就完事了

但是 Hugo 是用 Golang 写的，没有那么好的插件化操作，找了一下找到了这个 [Hugo Encryptor](https://github.com/Li4n0/hugo_encryptor)，还是 Vidar 的 Web 大佬写的（我直接膜

只是需要每次 build 之后多运行一个脚本，Github Action 加一行就好了

## 0x02 自动化部署

Hugo 的官方文档贴心的提供了 Github Action 部署的详细文档，我就在它的基础上稍作修改，一个是多运行了一些脚本（文章加密、上传 json），还有就是我的博客源码和 Github Pages 放在了不同仓库，需要推送到另一个仓库

这里把我的 workflow 配置也贴一下吧（随便糊的，大佬轻喷

```yaml
name: Blog

on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0
      
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Setup Others
        run: |
          sudo apt install python3
          pip3 install --upgrade 'algoliasearch>=2.0,<3.0' --user
          pip3 install setuptools --user
          pip3 install -r plugins/hugo_encryptor/requirements.txt --user

      - name: Build
        env:
          ALGOLIA_ADMIN_KEY: ${{ secrets.ALGOLIA_ADMIN_KEY }}
          ALGOLIA_APPID: ${{ secrets.ALGOLIA_APPID }}
          ALGOLIA_INDEX: ${{ secrets.ALGOLIA_INDEX }}
        run: |
          hugo
          python3 plugins/hugo_encryptor/hugo-encryptor.py
          python3 plugins/algolia_upload/upload.py public/index.json

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: X5tar/x5tar.github.io
          publish_branch: master
          publish_dir: ./public
          full_commit_message: ${{ github.event.head_commit.message }}
```

## 0x03 尾记

这次迁移整体没遇到什么障碍，加起来也就不到两个小时就完成了，而且主要时间都用在选主题和复制粘贴原来的文章了，以后遇到什么问题再来更吧

溜了溜了，明天就考密码学了现在还没学完（
