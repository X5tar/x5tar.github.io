# Git 学习笔记


## 〇、获取本地仓库

1. `git init` 初始化仓库
2. `git clone <url>` 从远程仓库克隆（url 可采用 git、http、https 和 ssh 等协议）

## 一、本地文件修改、提交及查看等

1. `git status` 查看已跟踪文件状态（修改、新加、改名、删除等）

2. `git add <file>` 追踪新文件

3. `.gitignore` 文件中添加被忽略的文件

   - 以 `#` 开头的内容为注释，被 Git 自动忽略
   - 可使用 Glob 表达式进行模式匹配
   - 模式最后跟斜杠（ `/` ）表示目录
   - 模式前加叹号（ `!` ） 表示取反

4. `git diff` 当前文件与暂存区之间的差异

   - `--cached` / `--staged` 暂存区与上次提交之间的差异

5. `git commit` 提交更改

   - `-m` 添加本次提交的说明（若无该参数将打开文本编辑器进行添加）
   - `-a` 提交所有已暂存的文件（跳过 `git add` 过程）
   - `--amend` 撤销最后的提交并重新提交

6. `git rm` 从 仓库（暂存区）和 工作目录 中删除

   - `-f` 强制删除（已修改并放入暂存区的文件）
   - `--cached` 只从仓库（暂存区）中删除，保留在工作目录中

7. `git mv` 移动文件（`git mv oldfile newfile` 相当于运行以下三条命令）

   ```shell
   mv oldfile newfile
   git rm oldfile
   gie add newfile
   ```

8. `git log` 查看提交历史

   - `-p` 显示每次提交的内容差异

   - `-<X>` (X 为数字) 只显示最近 X 次提交

   - `--stat` 仅显示简要的增改行数统计

   - `--pretty=` 指定显示的格式

     - `oneline` `short` `full` `fuller` 调整展示信息的多少

     - `format:""` 自定义要显示的格式（可使用如下占位符）

       ```
       %H 提交对象（commit）的完整哈希字串
       %h 提交对象的简短哈希字串
       %T 树对象（tree）的完整哈希字串
       %t 树对象的简短哈希字串
       %P 父对象（parent）的完整哈希字串
       %p 父对象的简短哈希字串
       %an 作者（author）的名字
       %ae 作者的电子邮件地址
       %ad 作者修订日期（可以用 -date= 选项定制格式）
       %ar 作者修订日期，按多久以前的方式显示
       %cn 提交者(committer)的名字
       %ce 提交者的电子邮件地址
       %cd 提交日期
       %cr 提交日期，按多久以前的方式显示
       %s 提交说明
       ```

   - `--graph` 用图形表示所在分支及合并历史

   - `--relative-date` 使用相对时间表示

   - `--name-only` 仅显示新增、修改、删除的文件的文件名

   - `--name-status` 显示新增、修改、删除的文件的文件名和执行的操作（A D M）

   - `--abbrev-commit` 仅显示 SHA-1 的前几个字符

   - `--shortstat` 仅显示总的行数修改统计（不显示单文件的变化）

   - `--since=` `--after=` 仅显示指定时间后的提交

   - `--until=` `--before=` 仅显示指定时间之前的提交

   - `--author=` 仅显示指定作者的提交

   - `--committer=` 仅显示指定提交者的提交

   - `--no-merges` 排除来自合并的提交

   - `--merges` 只显示来自合并的提交

   - `--grep=` 匹配提交信息中的关键字

   - 最后单独的 `--` 后的所有内容为限定的路径或文件名

9. `git reset HEAD <file>` 取消对文件的暂存

10. `git checkout -- <file>` 取消对文件的修改（仅限于修改后未暂存的文件）

## 二、本地与远程仓库之间的操作

1. `git remote` 列出所有远程库的名字
   - `-v` `--verbose` 显示对应的地址
   - `add <shortname> <url>` 添加新的远程仓库
   - `show <remote-name>` 查看某个远程仓库的详细信息
   - `rename <old-name> <new-name>` 修改远程仓库在本地的简称
   - `rm <remote-name>` 移除某个远程仓库
2. `git fetch <remote-name>` 拉取远程仓库有但本地没有的数据（不合并）
3. `git pull` 拉取跟踪的远程仓库对应分支的数据并合并
4. `git push <remote-name> <branch-name>` 将本地的某分支推送到远程仓库（默认为 origin  master）

## 三、标签有关操作

1. `git tag` 列出现有标签
   - `-l ''` 模式搜索符合条件的标签
   - `-a <tag-name> [SHA-1]` 添加（补加）含附注的标签
   - `-s <tag-name>` 同上添加标签并用 GPG 签署标签
   - `-m` 标签的附注说明
   - `<tag-name>` 添加轻量级标签（无需附注）
   - `-v <tag-name>` 验证已签署的标签
2. `git show <tag-name>` 查看相应标签的版本信息
3. `git push <remote-name> <tag-name>` 将本地的某标签推送到远程仓库
   - `--tags` 推送所有本地新增的标签

## 四、分支有关操作

1. `git branch` 列出所有分支和当前指向分支

   - `<branch-name>` 创建新分支

2. `git checkout`

   - `<branch-name>` 切换到某分支

   - `-b <branch-name>` 创建并切换到分支（相当于一下两条命令）

     ```shell
     git branch <branch-name>
     git checkout <branch-name>
     ```

   - `-b <branch-name> <remote-name>/<remote-branch>` 从远程分支创建分支并跟踪

   - `--track <remote-name>/<remote-branch>` 从远程分支创建**同名**分支并跟踪

   - `-d <branch-name>` 删除某分支

   - `-v` 显示每个分支最后一次提交的信息

3. `git merge <branch-name>` 将某分支合并到当前分支（可能出现冲突需要**手动处理**）

4. `git push <remote-name> <branch-name>:<remote-branch>` 将本地分支推送到命名不同的远程分支（本地分支留空推送空分支，即为删除远程分支）
