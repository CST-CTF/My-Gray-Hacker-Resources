# 用Python的urllib2(by bt3)进行网络黑客攻击

Python的[urllib2](https://docs.python.org/2/library/urllib2.html)资料库是**the tool**,可以与web服务进行交互,有一些函数和类别来帮助处理url. **urllib2**在[httplib]顶端写入(https://docs.python.org/2/library/httplib.html)资料库(它定义了类实现的HTTP和HTTPs的客户端).反过来, **httplib**使用[socket](http://bt3gl.github.io/black-hat-python-networking-the-socket-module.html)资料库.

这一次我[介绍urllib2](#intro),然后我主要攻克两个问题:[如何从他们的安装文件映射网页应用程序](#map)和[如何强力爆破网页应用程序内容以找到隐藏的资源](#brute1).


-----
开始的最简单的方法是查看**urlopen**方法,返回一个在Python中类似于**file**的对象(附加三个方法: **geturl**,针对资源的URL; **info**,针对元信息;以及**getcode**,针对HTTP状态代码).

## <a name="intro"></a>urllib2 101





### 简单获取
让我们看看一个简单的例子,如何[获取](http://www.w3schools.com/tags/ref_httpmethods.asp)所需求的工作.这是直接用[urlopen]完成的(https://docs.python.org/2/library/urllib2.html#urllib2.urlopen):


```python
>>> import urllib2
>>> msg = urllib2.urlopen('http://www.google.com')
>>> print msg.read(100)
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="en"><head><meta content="Search the world's information, including (...)
```

请注意,不同于类似[scapy](http://bt3gl.github.io/black-hat-python-infinite-possibilities-with-the-scapy-module.html)或[socket](http://bt3gl.github.io/black-hat-python-the-socket-module.html)的模块,我们在URL(HTTP)中*需要指定协议*.

现在,让我们创造并且自定义输出:

```python
import urllib2

response = urllib2.urlopen('http://localhost:8080/')
print 'RESPONSE:', response
print 'URL     :', response.geturl()

headers = response.info()
print 'DATE    :', headers['date']
print 'HEADERS :'
print headers

data = response.read()
print 'LENGTH  :', len(data)
print 'DATA    :'
print data
```

将会导致这样:
```sh
RESPONSE: <addinfourl at 140210027950304 whose fp = <socket._fileobject object at 0x7f8530eec350>>
URL     : http://www.google.com
DATE    : Tue, 23 Dec 2014 15:04:32 GMT
HEADERS :
Date: Tue, 23 Dec 2014 15:04:32 GMT
Expires: -1
Cache-Control: private, max-age=0
Content-Type: text/html; charset=ISO-8859-1
Set-Cookie: PREF=ID=365306c56a0ffee1:FF=0:TM=1419951872:LM=1419951872:S=lyvP_3cexMCllrVl; expires=Thu, 22-Dec-2016 15:04:32 GMT; path=/; domain=.google.com
Set-Cookie: NID=67=fkMfihQT2bLXyqQ8PIge1TwighxcsI4XVUWQl-7KoqW5i3T-jrzUqrC_lrtO7zd0vph3AzSMxwz2LkdWFN479RREL94s0hqRq3kOroGsUO_tFzBhN1oR9bDRMnW3hqOx; expires=Wed, 01-Jul-2015 15:04:32 GMT; path=/; domain=.google.com; HttpOnly
Server: gws
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN
Alternate-Protocol: 80:quic,p=0.02
Connection: close

LENGTH  : 17393
DATA    :
<!doctype html>(...)
```


### 简单文章

[POST](http://www.w3schools.com/tags/ref_httpmethods.asp)需要向URL传输数据([往往参照](https://docs.python.org/2/howto/urllib2.html#data)[CGI](http://en.wikipedia.org/wiki/Common_Gateway_Interface)在网页应用程序中的脚本或形式).

POST需要,不同于GET的要求,通常有诸如改变系统的状态之类的副作用.但数据也可以通过在URL中进行编码来通过一个HTTP GET请求.

在HTTML形式下,数据需要被编码并且这一编码由[urllib](https://docs.python.org/2/library/urllib.html)的工具**urlencode** (这里使用的一种方法以进行GET字符串查询)完成:


```python
import urllib
import urllib2

data = { 'q':'query string', 'foo':'bar' }
encoded_data = urllib.urlencode(data)

url = 'http://localhost:8080/?' + encoded_data

response = urllib2.urlopen(url)
print response.read()
```

事实上,使用**urllib2**时,更有效定制**urlopen**类函数的方式是通过将**Request object**作为数据参数:

```python
data = { 'q':'query string', 'foo':'bar' }
encoded_data = urllib.urlencode(data)

req = urllib2.Request(url, encoded_data)

response = urllib2.urlopen(req)
print response.read()
```

这是一个**urllib2**和**urllib**之间的区别:前者可以接受**Request object**设置头文件的URL请求,而后一个只接受URL.



### 头文件

正如我们在上面所学到的,我们可以不仅可以用字符串还可以用[Request]等级(https://docs.python.org/2/library/urllib2.html#urllib2.Request)来创建一个GET请求.这就允许了我们,比如,定义自定义头.

制作我们自己的标题,使用头键和自定义值,我们创建了一个标题字典.然后我们创建一个请求对象到**urlopen**申请函数调用.

例如,让我们看看**用户代理**头是如何工作的(这就是浏览器识别自己的方式) :

```python
>>> headers = {}
>>> headers['User-Agent'] = 'Googlebot'
>>> request = urllib2.Request(url, headers=headers)
>>> response = urllib2.urlopen(request)
>>> print "The Headers are: ", response.info()
The Headers are:  Date: Tue, 23 Dec 2014 15:27:01 GMT
Expires: -1
Cache-Control: private, max-age=0
Content-Type: text/html; charset=UTF-8
Set-Cookie: PREF=ID=8929a796c6fba710:FF=0:TM=1419953221:LM=1419953221:S=oEh5NKUEIEBinpwX; expires=Thu, 22-Dec-2016 15:27:01 GMT; path=/; domain=.google.com
Set-Cookie: NID=67=QhRTCRsa254cvvos3EXz8PkKnjQ6qKblw4qegtPfe1WNagQ2p0GlD1io9viogAGbFm7RVDRAieauowuaNEJS3aySZMnogy9oSvwkODi3uV3NeiHwZG_neZlu2SkO9MWX; expires=Wed, 01-Jul-2015 15:27:01 GMT; path=/; domain=.google.com; HttpOnly
P3P: CP="This is not a P3P policy! See http://www.google.com/support/accounts/bin/answer.py?hl=en&answer=151657 for more info."
Server: gws
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN
Alternate-Protocol: 80:quic,p=0.02
Connection: close
>>> print "The Date is: ", response.info()['date']
The Date is:  Tue, 23 Dec 2014 15:27:01 GMT
>>> print "The Server is: ", response.info()['server']
The Server is:  gws
>>> response.close()
```


我们也可以用**add_headers**方法添加标题:

```
>>> request = urllib2.Request('http://www.google.com/')
>>> request.add_header('Referer', 'http://www.python.org/')
>>> request.add_header('User-agent', 'Mozilla/5.0')
>>> response = urllib2.urlopen(request)
```


### HTTP身份验证

当需要身份验证时,这个过程中服务器发送消息头(和**401错误代码**) 请求.响应同时也指定了**认证方案**和**范围**.像这样:

```
WWW-Authenticate: SCHEME realm="REALM".
```

然后，客户机使用该域的名称和密码重新尝试请求,在请求中包含一个标题.在请求中包含一个标题:

1) 创建密码管理器,

```
passwd_mgr = urllib2.HTTPPasswordMgrWithDefaultRealm()
```

2) 添加用户名和密码,

```
top_url = "http://example.com/"
passwd_mgr.add_password(None, top_url, username, password)
```

3) 创建一个身份验证处理程序,

```
handler = urllib2.HTTPBasicAuthHandler(password_mgr)
```

4) 创建一个*opener* (OpenerDirector实例),

```
opener = urllib2.build_opener(handler)
```

5) 使用打开器获取URL,

```
opener.open(a_url)
```

6) 安装器,

```
urllib2.install_opener(opener)
```

7) 最后,打开页面(身份验证被自动处理):

```
pagehandle = urllib2.urlopen(top_url)
```




### 错误处理

**urllib2**错误处理:

```python
>>> request = urllib2.Request('http://www.false_server.com')
>>> try:
        urllib2.urlopen(request)
>>> except urllib2.URLError, e:
        print e.reason
(4, 'getaddrinfo failed')
```

来自服务器的每个HTTP响应都包含一个数字的[状态码](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes).默认的处理程序负责处理这些响应和其他的响应, **urlopen**提出了一个**HTTPError** (**URLError**的子集合).


### 其他可用的方法

**urllib2**库中其他可用的方法:

* **install_opener**和**build_opener**:安装并返回一个OpenDirector实例.

* ** URLError**和**HTTPError**: 分别地对问题提出异常,处理外来的HTTP错误,和进行处理[HTTP错误响应](https://docs.python.org/2/howto/urllib2.html#error-codes).

* **HTTPCookieProcessor**: 处理HTTP cookies.

* **HTTPProxyHandler**: 向代理发送请求.

* **AbstractBasicAuthHandler**, **HTTPBasicAuthHandler**, **ProxyBasicAuthHandler**, **HTTPDigestAuthHandler**, **AbstractDigestAuthHandler**, **ProxyDigestAuthHandler**: 处理认证.

* **HTTPPasswordMgr**和**HTTPPasswordMgrWithDefaultRealm**: 保存一个域 URL 用户和密码映射的数据库.

* **HTTPHandler**, **HTTPSHandler**, **FileHandler**, **FTPHandler**, **UnknownHandler**: 处理消息.


**Request**目标的可用方法:

* **add_data**, **has_data**和**get_data**:处理请求数据.
* **add_header**, **add_unredirected_header**, **has_header**, **get_header**, **header_items**: 处理头数据.
* **get_full_url**, **get_type**, **get_host**, **get_selector**, **set_proxy**, **get_origin_req_host**: 处理URL数据.


我们不要忘记**urllib**的[urlparse](http://pymotw.com/2/urlparse/index.html#module-urlparse), 它提供了分析URL字符串的函数. **urlparse**在几个可选组件中中断URL字符串: **scheme** (实例: http), **location** (实例: www.python.org:80), **path** (实例: index.html), **query**和**fragment**.

其他常见的函数是**urljoin**和**urlsplit**.




---

## <a name="map"></a>从安装包中映射网页应用程序

[内容管理系统](http://en.wikipedia.org/wiki/Content_management_system)是让创建博客或简单网站变得容易的平台.它们在共享宿主环境中很常见.但是, 当所有的安全过程都没有遵循时, 对于攻击者来说，他们可以很容易地访问服务器.

在这部分中，我们将构建一个扫描器用以搜索远程目标上可访问的所有文件,以下是下载的网页应用程序的结构. 这是基于[Black Hat Python]的一个例子(http://www.nostarch.com/blackhatpython).

这种类型的扫描器可以通过[.htaccess](http://en.wikipedia.org/wiki/Htaccess)显示安装文件,和未处理的目录, 以及其他一些对攻击有用的文件.



### 制作扫描仪

在我们的扫描脚本中,我们利用Python的[Queue](https://docs.python.org/2/library/queue.html)对象来构建大量的项目和多个线程，挑选项目进行处理. 这将使扫描仪运行得非常快.步骤如下:

1) 我们定义目标URL(在本例中，我们借用了书上的例子),线程的数量, 我们下载并提取webapp的本地目录以及带有我们不感兴趣的文件扩展的过滤器:

```python
import urllib2
import Queue
import os
import threading

THREADS = 10
TARGET = 'http://www.blackhatpython.com'
DIRECTORY = '/home/User/Desktop/wordpress'
FILTERS = ['.jpg', '.css', '.gif', '.png']
```

2) 我们用一个循环来定义一个函数，这个循环一直执行，直到有路径的队列是空的. 在每次迭代中，我们都会得到其中一条路径并将它添加到目标URL中，以查看它是否存在(输出HTTP状态码):

```python
def test_remote():
    while not web_paths.empty():
        path = web_paths.get()
        url = '%s%s' % (TARGET, path)
        request = urllib2.Request(url)

        try:
            response = urllib2.urlopen(request)
            content = response.read()
            print '[%d] => %s' % (response.code, path)
            response.close()
        except urllib2.HTTPError as error:
            fail_count += 1
            print "Failed" + str(error.code)
```

3) 主循环首先为路径创建队列然后用**os.walk**方法在网页应用程序的本地版本中映射所有文件和目录,将名称添加到队列中(经过我们的自定义扩展列表筛选后):

```python
if __name__ == '__main__':
    os.chdir(DIRECTORY)
    web_paths = Queue.Queue()
    for r, d, f in os.walk('.'):
        for files in f:
            remote_path = '%s/%s' %(r, files)
            if remote_path[0] == '.':
                remote_path = remote_path[1:]
            if os.path.splitext(files)[1] not in FILTERS:
                web_paths.put(remote_path)
```

4) 最后, 我们创建将被发送到函数的线程**test_remote**.循环被保留，直到路径队列为空:

```python
    for i in range(THREADS):
        print 'Spawning thread number: ' + str(i+1)
        t = threading.Thread(target=test_remote)
        t.start()
```


### 测试扫描仪

现在我们准备测试我们的扫描器. 我们下载并测试三个网页应用程序: [WordPress](https://en-ca.wordpress.org/download/), [Drupal](https://www.drupal.org/project/download),和[Joomla 3.1.1](http://www.joomla.org/announcements/release-news/5499-joomla-3-1-1-stable-released.html).


运行第一个Joomla给出以下结果:

```sh
$ python mapping_web_app_install.py
Spawning thread number: 1
Spawning thread number: 2
Spawning thread number: 3
Spawning thread number: 4
Spawning thread number: 5
Spawning thread number: 6
Spawning thread number: 7
Spawning thread number: 8
Spawning thread number: 9
Spawning thread number: 10
[200] => /web.config.txt
[200] => /modules/mod_whosonline/helper.php
[200] => /LICENSE.txt
[200] => /README.txt
[200] => /modules/mod_whosonline/mod_whosonline.xml
[200] => /joomla.xml
[200] => /robots.txt.dist
(...)
```


用Wordpress运行:

```sh
(...)
[200] => /wp-links-opml.php
[200] => /index.php
[200] => /wp-config-sample.php
[200] => /wp-load.php
[200] => /license.txt
[200] => /wp-mail.php
[200] => /xmlrpc.php
[200] => /wp-trackback.php
[200] => /wp-cron.php
[200] => /wp-admin/custom-background.php
[200] => /wp-settings.php
[200] => /wp-activate.php
(...)
```

最后,用Drupal运行,我们只得到5个文件:

```sh
(...)
[200] => /download.install
[200] => /LICENSE.txt
[200] => /README.txt
[200] => /download.module
[200] => /download.info
```

在所有这些结果中，我们能够找到一些不错的结果，包括XML和txt文件. 这个侦察可以是攻击的开始.酷毙了.




-----

## <a name="brute1"></a>强力网页应用程序的内容

一般来说，我们不知道在web服务器中可以访问的文件的结构(我们没有像之前的例子那样使用网页应用程序). 通常我们可以部署一只蜘蛛, 就像在[Burp suite](http://portswigger.net/burp/)中,爬上目标并找到它们.然而，这可能无法找到诸如此类的敏感文件, 例如, 开发/配置文件和调试脚本.

发现敏感文件的最佳方式是蛮力共同的文件名和目录.我们该怎么做呢?

当我们已经有了目录和文件的单词列表时，任务就变得很简单了. 这些列表可以从源代码就像[DirBurster](https://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)项目或是[SVNDigger](https://www.netsparker.com/blog/web-security/svn-digger-better-lists-for-forced-browsing/)中下载.


因为浏览第三方网站是不合法的,我们将会使用可供测试的*play*网站.若干例子(来自[here](http://blog.taddong.com/2011/10/hacking-vulnerable-web-applications.html)):

* [testphp.vulnweb.com](http://testphp.vulnweb.com)
* [testasp.vulnweb.com](http://testasp.vulnweb.com)
* [testaspnet.vulnweb.com](http://testaspnet.vulnweb.com)
* [testphp.vulnweb.com](http://testphp.vulnweb.com)
* [crackme.cenzic.com](http://crackme.cenzic.com)
* [google-gruyere.appspot.com/start](http://google-gruyere.appspot.com/start)
* [www.hacking-lab.com/events/registerform.html](https://www.hacking-lab.com/events/registerform.html?eventid=245)
* [hack.me](https://hack.me)
* [www.hackthissite.org](http://www.hackthissite.org)
* [zero.webappsecurity.com](http://zero.webappsecurity.com)
* [demo.testfire.net](http://demo.testfire.net)
* [www.webscantest.com](http://www.webscantest.com)
* [hackademic1.teilar.gr](hackademic1.teilar.gr)
* [pentesteracademylab.appspot.com](http://pentesteracademylab.appspot.com)


### 写脚本
在我们的脚本中，我们接受文件的通用名称列表和目录并使用它们来尝试在服务器上发现可到达的路径.

以如以前同样的方式,我们可以通过创建线程池来发现内容以达到一个合理的速度.

我们脚本的步骤是:


1) 我们定义目标, 线程的数量, 单词列表的路径 (我提供的 [here](https://github.com/bt3gl/My-Gray-Hacker-Resources/tree/master/Other_Hackings/useful_lists/files_and_dir_lists)), 一个流氓用户代理, 以及我们想要查看的扩展的过滤列表:

```python
import urllib2
import threading
import Queue
import urllib

THREADS = 10
TARGETS  = 'http://testphp.vulnweb.com'
WORDLIST_FILE = '../files_and_dir_lists/SVNDigger/all.txt'
USER_AGENT = 'Mozilla/5.0 (X11; Linux x86_64l rv:19.0) Gecko/20100101 Firefox/19.0'
EXTENSIONS = ['.php', '.bak', '.orig', '.inc']
```

2) 我们创建一个函数来读取我们的单词列表, 然后将每个单词添加到队列中,返回此队列:

```python
def build_wordlist(WORDLIST_FILE):
    f = open(WORDLIST_FILE, 'rb')
    raw_words = f.readlines()
    f.close()
    words = Queue.Queue()
    for word in raw_words:
        word = word.rstrip()
        words.put(word)
    return words
```

3) 我们创建了一个循环超过队列大小的函数, 检查它是一个目录还是一个文件(使用扩展列表),然后强力爆破这些URL:


```python
def dir_bruter(word_queue, TARGET, EXTENSIONS=None):
    while not word_queue.empty():
        attempt = word_queue.get()
        attempt_list = []
        if '.' not in attempt:
            attempt_list.append('/%s/' %attempt)
        else:
            attempt_list.append('/%s' %attempt)
        if EXTENSIONS:
            for extension in EXTENSIONS:
                attempt_list.append('/%s%s' %(attempt, extension))
        for brute in attempt_list:
            url = '%s%s' %(TARGET, urllib.quote(brute))
            try:
                headers = {}
                headers['User-Agent'] = USER_AGENT
                r = urllib2.Request(url, headers = headers)
                response = urllib2.urlopen(r)
                if len(response.read()):
                    print '[%d] => %s' %(response.code, url)
            except urllib2.URLError, e:
                if hasattr(e, 'code') and e.code != 404:
                    print '[! %d] => %s' %(e.code, url)
                pass
```

4) 在主循环中,我们建立单词列表然后为我们的**dir_bruter**函数创建环境:

```python
if __name__ == '__main__':
    word_queue = build_wordlist(WORDLIST_FILE)

    for i in range(THREADS):
        print 'Thread ' + str(i)
        t = threading.Thread(target=dir_bruter, args=(word_queue, target))
        t.start()
```

### 运行脚本

在网页应用程序目标中运行此操作会打印出这样的东西:

```sh
$ python brute_forcing_locations.py
[200] => http://testphp.vulnweb.com/CVS
[200] => http://testphp.vulnweb.com/admin
[200] => http://testphp.vulnweb.com/script
[200] => http://testphp.vulnweb.com/images
[200] => http://testphp.vulnweb.com/pictures
[200] => http://testphp.vulnweb.com/cart.php
[200] => http://testphp.vulnweb.com/userinfo.php
!!! 403 => http://testphp.vulnweb.com/cgi-bin/
(...)
```

非常整洁!







-----

## 其他:

- [Form Contents](http://www.w3.org/TR/REC-html40/interact/forms.html#h-17.13.4)
- [A robot.txt parser](http://pymotw.com/2/robotparser/index.html#module-robotparser)
- [stackoverflow](http://stackoverflow.com/questions/tagged/urllib2)
- [Black Hat Python](http://www.nostarch.com/blackhatpython).
