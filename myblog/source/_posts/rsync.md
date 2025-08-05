---
title: rsync
date: 2025-08-05 17:43:37
tags:
categories: linux
---
# 命令主要参数
```shell
-a, ––archive	归档模式，表示以递归方式传输文件，并保持所有文件属性，等价于 -rlptgoD (注意不包括 -H)
-r, ––recursive	对子目录以递归模式处理
-l, ––links	保持符号链接文件
-H, ––hard-links	保持硬链接文件
-p, ––perms	保持文件权限
-t, ––times	保持文件时间信息
-g, ––group	保持文件属组信息
-o, ––owner	保持文件属主信息 (super-user only)
-D	保持设备文件和特殊文件 (super-user only)
-z, ––compress	在传输文件时进行压缩处理
––exclude=PATTERN	指定排除一个不需要传输的文件匹配模式
––exclude-from=FILE	从 FILE 中读取排除规则
––include=PATTERN	指定需要传输的文件匹配模式
––include-from=FILE	从 FILE 中读取包含规则
––copy-unsafe-links	拷贝指向SRC路径目录树以外的链接文件
––safe-links	忽略指向SRC路径目录树以外的链接文件（默认）
––existing	仅仅更新那些已经存在于接收端的文件，而不备份那些新创建的文件
––ignore-existing	忽略那些已经存在于接收端的文件，仅备份那些新创建的文件
-b, ––backup	当有变化时，对目标目录中的旧版文件进行备份
––backup-dir=DIR	与 -b 结合使用，将备份的文件存到 DIR 目录中
––link-dest=DIR	当文件未改变时基于 DIR 创建硬链接文件
––delete	删除那些接收端还有而发送端已经不存在的文件
––delete-before	接收者在传输之前进行删除操作 (默认)
––delete-during	接收者在传输过程中进行删除操作
––delete-after	接收者在传输之后进行删除操作
––delete-excluded	在接收方同时删除被排除的文件
-e, ––rsh=COMMAND	指定替代 rsh 的 shell 程序
––ignore-errors	即使出现 I/O 错误也进行删除
––partial	保留那些因故没有完全传输的文件，以是加快随后的再次传输
––progress	在传输时显示传输过程
-P	等价于 ––partial ––progress
––delay-updates	将正在更新的文件先保存到一个临时目录（默认为 “.~tmp~”），待传输完毕再更新目标文件
-v, ––verbose	详细输出模式
-q, ––quiet	精简输出模式
-h, ––human-readable	输出文件大小使用易读的单位（如，K，M等）
-n, ––dry-run	显示哪些文件将被传输
––list-only	仅仅列出文件而不进行复制
––rsyncpath=PROGRAM	指定远程服务器上的 rsync 命令所在路径
––password-file=FILE	从 FILE 中读取口令，以避免在终端上输入口令，通常在 cron 中连接 rsync 服务器时使用
-4, ––ipv4	使用 IPv4
-6, ––ipv6	使用 IPv6
```

# 命令行模式
```shell
#常用参数
rsync -avzSP \ #端点续传整合小文件
  --contimeout=120 \    # 连接超时 120 秒
  --timeout=60 \       # 数据传输超时 60 秒
  --delay-updates \    # 原子性替换文件
  --log-file=rsync.log  #日志记录
/cygdrive/c/     /cygdrive/d/ #windows必须加上/cygdrive前缀

#可选参数 --delay-updates 需要目标磁盘空间较大，对正在进行io操作的先跳过后更新
#--bwlimit=6000 限制带宽在6MB/s
#--exclude='*.log' 过滤文件不传输支持正则
#--delete  保证源和目标完全一致
#--password-file=/etc/rsyncd_users.db backuper@127.0.0.1::wwwroot

# /lib/systemd/system/rsync.service
[Unit]
Description=fast remote file copy program daemon
ConditionPathExists=/etc/rsyncd.conf

[Service]
ExecStart=/usr/bin/rsync --daemon --no-detach

[Install]
WantedBy=multi-user.target
```


# 后台server模式
## linux配置
```shell
cat >> /etc/rsyncd.conf  << EOF
uid = root					     
gid = root					    
use chroot = yes					
address = 0.0.0.0			
port 873						    
log file = /var/log/rsyncd.log		
pid file = /var/run/rsyncd.pid		
hosts allow = *		
[wwwroot]					        
path = /data/				
comment = Document Root 
read only = yes					    
dont compress = *.gz *.bz2 *.tgz *.zip *.rar *.z  
auth users = backuper srs			
secrets file = /etc/rsyncd_users.db			      
EOF
echo 'backuper:backuperpasswd' |tee -a /etc/rsyncd_users.db	
chmod 600 /etc/rsyncd_users.db
#客户端操作	
#echo 'backuperpasswd' |tee /etc/client.pass && chmod 600 /etc/client.pass
#拉取
#rsync -avzPS --password-file=/etc/client.pass backuper@127.0.0.1::wwwroot /tmp/rsync_daemon
#发送；相当于把/tmp/rsync_daemon同步到/data/ read only 要设置成no
#rsync -avzPS --password-file=/etc/client.pass /tmp/rsync_daemon backuper@127.0.0.1::wwwroot 
==================================================================
#正常配置里面不能有注释
uid = root					     
gid = root					    
use chroot = yes					#禁锢在源目录
address = 0.0.0.0			#监听地址，监听本机地址
port 873						    #监听端口 tcp/udp 873，
log file = /var/log/rsyncd.log		#日志文件位置
pid file = /var/run/rsyncd.pid		#存放进程 ID 的文件位置
hosts allow = *		#允许同步的客户机网段
max connections = 5 #最大五个连接，默认没有限制
[wwwroot]					        #共享模块名称
path = /data				#源目录的实际路径（同步的目录）
comment = Document Root 
read only = yes					    #是否为只读
dont compress = *.gz *.bz2 *.tgz *.zip *.rar *.z  #同步时不再压缩的文件类型
auth users = backuper srs			#授权账户，多个账号以空格分隔
secrets file = /etc/rsyncd_users.db			      #存放账户信息的数据文件
EOF
```
## windows配置
[cygwin下载地址](https://www.cygwin.com/setup-x86_64.exe)
```shell
#windows
uid = Administrator    
gid = Users  
use chroot = yes
address = 0.0.0.0
port 873    
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
hosts allow = *
[wwwroot]        
path = /cygdrive/c/data
comment = Document Root 
read only = yes    
dont compress = *.gz *.bz2 *.tgz *.zip *.rar *.z  
auth users = backuper srs
secrets file = /etc/rsyncd_users.db
#在cygwin终端执行
#echo 'backuperpasswd' |tee /etc/client.pass && chmod 600 /etc/client.pass      
#rsync -avzPS --password-file=/etc/client.pass backuper@127.0.0.1::wwwroot /cygdrive/c/rsync_daemon
```

# 额外
```shell
#!/bin/bash

# 配置参数
PASSWORD='pass'
#REMOTE_PATH="administrator@1.1.1.1:/cygdrive/c/data"
REMOTE_PATH="root@127.0.0.1:/data"
LOCAL_PATH="/tmp/rsync/"
#windows必选
#RSYNC_PATH="C:\\cygwin64\\bin\\rsync.exe"
LOG_FILE="rsync.log"

# 最大重试次数
MAX_RETRIES=10
# 重试间隔（秒）
RETRY_INTERVAL=60

# 循环重试
retry=0
while [ $retry -lt $MAX_RETRIES ]; do
  echo "$(date) - 第 $((retry+1)) 次尝试同步..." | tee -a "$LOG_FILE"
  
  # 执行同步命令
  sshpass -p "$PASSWORD" rsync -avzSP \
    --log-file="$LOG_FILE" \
  #  --rsync-path="$RSYNC_PATH" \
    --timeout=30 \
    --contimeout=30 \
    "$REMOTE_PATH" "$LOCAL_PATH"
  
  # 检查退出状态码
  if [ $? -eq 0 ]; then
    echo "$(date) - 同步成功！" | tee -a "$LOG_FILE"
    exit 0
  else
    echo "$(date) - 同步失败，${RETRY_INTERVAL}秒后重试..." | tee -a "$LOG_FILE"
    sleep $RETRY_INTERVAL
    ((retry++))
  fi
done

echo "$(date) - 达到最大重试次数，同步终止。" | tee -a "$LOG_FILE"
exit 1

```
