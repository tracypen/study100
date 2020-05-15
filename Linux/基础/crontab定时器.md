> crontab是Unix和Linux用于设置周期性被执行的指令。通过crontab命令，可以在固定间隔时间执行指定的系统指令或shell脚本。时间间隔的单位可以是分钟、小时、日、月、周及以上的任意组合。 

- 安装

~~~
yum install crontabs 
~~~

- 服务操作说明：

~~~
service crond start   ## 启动服务 
service crond stop    ## 关闭服务 
service crond restart ## 重启服务 
service crond reload  ## 重新载入配置 
service crond status  ## 查看crontab服务状态：
chkconfig crond --list  ## 查看crontab服务是否已设置为开机启动 
chkconfig crond on    ## 加入开机自动启动 
~~~

- 命令格式 

~~~
crontab [-u user] file crontab [-u user] [ -e | -l | -r ] 
~~~

- 参数说明：

~~~
-u user：用来设定某个用户的crontab服务  file：file是命令文件的名字,表示将file做为crontab的任务列表文件并载入crontab。
-e：编辑某个用户的crontab文件内容。如果不指定用户，则表示编辑当前用户的crontab文件。
-l：显示某个用户的crontab文件内容。如果不指定用户，则表示显示当前用户的crontab文件内容。
-r：删除定时任务配置，从/var/spool/cron目录中删除某个用户的crontab文件，如果不指定用户，则默认删除当前用户的crontab文件。 
-i：在删除用户的crontab文件时给确认提示。 
~~~

- 配置说明、实例 

> 分  时  日  月  周  命令  第1列表示分钟1～59 每分钟用*或者 */1表示  第2列表示小时0～23（0表示0点）第3列表示日期1～31  第4列表示月份1～12  第5列标识号星期0～6（0表示星期天）  第6列要运行的命令配置实例：

~~~properties
*/1 * * * * date >> /root/date.txt
每分钟执行一次date命令
30 21 * * * /usr/local/etc/rc.d/httpd restart  
每晚的21:30重启apache。  
45 4 1,10,22 * * /usr/local/etc/rc.d/httpd restart  
每月1、10、22日的4 : 45重启apache。  
10 1 * * 6,0 /usr/local/etc/rc.d/httpd restart  
每周六、周日的1 : 10重启apache。  
0,30 18-23 * * * /usr/local/etc/rc.d/httpd restart  
每天18 : 00至23 : 00之间每隔30分钟重启apache。  
* 23-7/1 * * * /usr/local/etc/rc.d/httpd restart  
晚上11点到早上7点之间，每隔一小时重启apache
~~~

