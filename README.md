#Discuz!利用SSRF+缓存应用代码执行漏洞环境及PoC
##简介
```
Desc : Discuz_ssrf_redis_codeexec.

References : https://www.seebug.org/vuldb/ssvid-91879.
```
##Docker+Pentest

###漏洞复现

####漏洞详情
漏洞就是通过ssrf来操作redis，更改了全局变量的值，导致任意代码执行。文件：source\function\function_core.php(1080G)
```
function output_replace($content) {
		...
		$content = preg_replace($_G['setting']['output']['preg']['search'], $_G['setting']['output']['preg']['replace'], $content);}
	return $content;
}
```
当dz设置使用缓存后，初始化时会把缓存内容加入全局变量$_G，写入*_setting，访问/discuz/forum.php?mod=ajax&inajax=yes&action=getthreadtypes，直接Getshell。
####环境

Ubuntu 14.04 + LNMP + redis /mem+ Discuz X3.2!
```
./addons.sh install memcached #lnmp安装mem
./addons.sh install redis     #lnmp安装redis

```
Windows 7 + phpStudy + redis-cli.exe/memcached + Discuz X3.2!
配置php.ini
```
extension=php_igbinary.dll
extension=php_redis.dll
extension=php_memcache.dll
```
Kali(Debain) + Docker(mysql + redis + skyzhou/docker-discuz)
```bash
docker run --name dz-mysql -e MYSQL_ROOT_PASSWORD=root -d mysql

docker run --name dz-redis -d yfix/redis

docker run --name dz-ssrf --link dz-mysql:mysql -p 8888:80 -d dz-redis-init apache2 "-DFOREGROUND"
```
访问127.0.0.1:8888进行Discuz!的安装，将config/config_global.php中redis的地址和端口改为redis容器的地址和端口，在后台中 全局 -> 性能优化 -> 内存优化 中查看redis是否被启用，若已启用则搭建完成。

####过程中的疑问
* Dz的ssrf一旦有命令执行，直接getshell
* Redis的未授权访问，嵌套Lua脚本将会导致代码执行直接写shell/私钥
* ...

###修复意见
####Discuz及Dz的SSRF问题

* 及时升级Dz版本，尽量避免使用不明第三方插件
* 修改function_core.php文件,替换即可
```
if (preg_match("(/|#|\+|%).*(/|#|\+|%)e", $_G['setting']['output']['preg']['search']) !== FALSE) { die("request error"); } 
	$content = preg_replace($_G['setting']['output']['preg']['search'], $_G['setting']['output']['preg']['replace'], $content);
```
####Redis未授权访问

* 配置bind选项，限定可以连接Redis服务器的IP，修改 Redis 的默认端口6379
* 配置认证，也就是AUTH，设置密码，密码会以明文方式保存在Redis配置文件中
* 配置rename-command 配置项 “RENAME_CONFIG”，这样即使存在未授权访问，也能够给攻击者使用config 指令加大难度
* 好消息是Redis作者表示将会开发”real user”，区分普通用户和admin权限，普通用户将会被禁止运行某些命令，如config
* 照妖镜https://www.seebug.org/monster/?vul_id=89715

##PoC使用

使用pocsuite框架
Usage:
```
pocsuite -r dz_redis_exec.py -u url --verify
pocsuite -r dz_redis_exec.py -u url --attack
```
环境测试均success!

##参考

* http://pocsuite.org/
* https://github.com/imp0wd3r
* https://github.com/C1tas
* http://pan.baidu.com/s/1dFHOQUt
