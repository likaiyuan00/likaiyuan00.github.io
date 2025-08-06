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
## 大量文件添加失败重试
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
**两种同步模式**<br>
**1.服务端推送；需要每个客户端启动rsync --deamon添加配置文件不然需要使用ssh模式涉及密码不安全,监控服务端文件变化去同步客户端节点大部分情况用这种，弊端就是节点较多服务端同步起来负载会比较高**<br>
**2.客户端推送；只需要在服务端配置rsync --deamon监控客户端文件变化去推送数据到服务端，由于多节点存在数据不一致这种情况建议不使用--delete不然一个节点操作删除，服务端也会被删除；或者使用客户端拉取这种情况只有通过定时任务实现就无法使用监控程序了**
## rsync+inotify
* 异地备份cdn节点少量同步；变动不频繁
```shell
inotify-tools 包含 inotifywatch  inotifywait两个命令
inotifywatch -v -t 60 -r /var/log #统计次数
inotifywait -mrq --format "%T %w%f %e" --timefmt "%F-%T" -e create,delete,move,modify,attrib /data/ |  
  while read TIME FILE EVENT; do
  echo "时间: $TIME | 文件: $FILE | 事件: $EVENT "
done
%T	时间戳（需配合 --timefmt 定义格式）
%w	监控目录的路径（绝对或相对路径）
%f	触发事件的文件名（不含路径）
%e	事件类型（多个事件用逗号分隔）

-m 持续监听
-r 使用递归形式监视目录
-q 减少冗余信息，只打印出需要的信息
-e 指定要监视的事件，多个时间使用逗号隔开
–timefmt 时间格式
–format 监听到的文件变化的信息

access	文件被读取
modify	文件内容被修改
attrib	文件元数据（如权限、时间戳）变更
create	文件/目录创建
delete	文件/目录删除
open, close	文件被打开或关闭


#在client运行脚本，client目录发送变化会同步到rsync服务端
cat > ./inotify_rsync.sh << 'EOF'  
#!/bin/bash
# Rsync配置
RSYNC_CMD="rsync -avzS --partial --delay-updates --delete --password-file=/etc/client.pass /data/ backuper@10.0.1.122::wwwroot"
LOG_FILE="/var/log/inotify_rsync.log"

# 创建日志文件（如果不存在）
touch "$LOG_FILE"

# 开始监控并处理事件
inotifywait -mrq --format "%T %w%f %e" --timefmt "%F-%T" -e create,delete,move,modify,attrib /data/ |  
while read TIME FILE EVENT; do
    # 记录事件到日志
    echo "[事件] 时间: $TIME | 文件: $FILE | 操作: $EVENT" >> "$LOG_FILE"
    
    # 执行rsync同步（添加错误重试机制）
    if ! $RSYNC_CMD >> "$LOG_FILE" 2>&1; then
        echo "[错误] 同步失败！时间: $(date '+%F-%T')" >> "$LOG_FILE"
    else
        echo "[同步] 成功完成！时间: $(date '+%F-%T')" >> "$LOG_FILE"
    fi
done
EOF
chmod +x inotify_rsync.sh
cat >  /etc/systemd/system/inotify_rsync.service << 'EOF'  
[Unit]
Description=Auto Sync Service via inotify+rsync

[Service]
Type=simple
User=root
ExecStart=/tmp/rsync_daemon/inotify_rsync.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF

```


## rsync+lsyncd
* 适合大量数据同步场景；变动频繁<br>
[官方文档参数解读](https://lsyncd.github.io/lsyncd/manual/config/layer4/)
```shell
#ubuntu /etc/lsyncd/lsyncd.conf.lua
cat > /etc/lsyncd.conf << 'EOF'
settings {
  logfile = "/var/log/lsyncd.log",
  statusFile = "/var/log/lsyncd.status",
  insist = true,
  statusInterval = 10
}

sync {
  default.rsync,
  source = "/tmp/rsync_ly",
  target = "backuper@10.0.1.122::wwwroot",
  -- excludeFrom = "/etc/rsyncd.d/rsync_exclude.lst",
  delete = true,
  delay = 20,
  maxDelays = 1,
  rsync = {
    binary = "/usr/bin/rsync",
    archive = true,
    compress = true,
    verbose = false,
    password_file = "/etc/client.pass",
    _extra    = {
            "--partial",          
            "--timeout=300"    
  --          "--bwlimit=5000"      
        },
  }
}
EOF

========================================================

-- 全局配置：
settings {
        logfile ="/var/log/lsyncd/lsyncd.log", -- 定义日志文件
        statusFile ="/var/log/lsyncd/lsyncd.status",  -- 定义状态文件
        pidfile = "/var/log/lsyncd/lsyncd.pid",-- 定义pid文件
        inotifyMode = "CloseWrite",-- 指定inotify监控的事件，默认是CloseWrite，还可以是Modify或CloseWrite or Modify
      	maxProcesses = 7,-- 同步进程的最大个数。假如同时有20个文件需要同步，而maxProcesses = 8，则最大能看到有8个rysnc进程
        nodaemon =true,-- 表示不启用守护模式，默认；
        maxDelays = 1, --  累计到多少所监控的事件激活一次同步，即使后面的delay延迟时间还未到
        inist = ture --keep running at startup although one or more targets failed due to not being reachable.  一般不用配置
       }

-- sync部分配置：
sync {
      default.rsync,     -- rsync、rsyncssh、direct三种模式：
    -- default.rsync ：本地目录间同步，使用rsync，也可以达到使用ssh形式的远程rsync效果，或daemon方式连接远程rsyncd进程；
    -- default.direct ：本地目录间同步，使用cp、rm等命令完成差异文件备份；
    -- default.rsyncssh ：同步到远程主机目录，rsync的ssh模式，需要使用key来认证；
      source = "/tmp/src", -- source 同步的源目录，使用绝对路径
      target = "/tmp/dest", -- target 定义目的地址.对应不同的模式有几种写法:
    	-- /tmp/dest ：本地目录同步，可用于direct和rsync模式；
    	-- 10.4.7.10:/tmp/dest ：同步到远程服务器目录，可用于rsync和rsyncssh模式，拼接的命令类似于/usr/bin/rsync -ltsd --delete --include-from=- --exclude=* SOURCE TARGET，剩下的就是rsync的内容了，比如指定username，免密码同步；
   		-- 10.4.7.10::module ：同步到远程服务器目录，用于rsync模式；
      init = true,  -- init 这是一个优化选项，当init = false，只同步进程启动以后发生改动事件的文件，原有的目录即使有差异也不会同步。默认是true；
      delay = 3, -- delay 累计事件，等待rsync同步延时时间，默认15秒（最大累计到1000个不可合并的事件）。也就是15s内监控目录下发生的改动，会累积到一次rsync同步，避免过于频繁的同步。（可合并的意思是，15s内两次修改了同一文件，最后只同步最新的文件）;
      excludeFrom = "/etc/rsyncd.d/rsync_exclude.lst",  -- excludeFrom 排除选项，后面指定排除的列表文件，如excludeFrom = "/etc/lsyncd.exclude"，如果是简单的排除，可以使用exclude = LIST。这里的排除规则写法与原生rsync有点不同，更为简单：
		-- 监控路径里的任何部分匹配到一个文本，都会被排除，例如/bin/foo/bar可以匹配规则foo
		-- 如果规则以斜线/开头，则从头开始要匹配全部
		-- 如果规则以/结尾，则要匹配监控路径的末尾
		-- ?匹配任何字符，但不包括/
		-- *匹配0或多个字符，但不包括/
		-- **匹配0或多个字符，可以是/
      delete	=	'running',  -- delete 为了保持target与souce完全同步，Lsyncd默认会delete = true来允许同步删除。它除了false，还有startup、running值：
      -- delete	=	true       # 在目标上删除源中没有的内容。在启动时以及在正常操作期间删除的内容
      -- delete	=	false      # 不会删除目标上的任何文件。不在启动时也不在正常操作上
      -- delete	=	'startup'  # Lsyncd将在启动时删除目标上的文件，但不会在正常操作时删除
      -- delete	=	'running'  # Lsyncd在启动时不会删除目标上的文件，但会删除正常操作期间删除的文件

    
-- rsync部分配置：    
      -- delete和exclude本来都是rsync的选项，上面是配置在sync中的，这样做的原因是为了减少rsync的开销
      rsync = {
             bwlimit=200, -- bwlimit 限速，单位kb/s，与rsync相同（这么重要的选项在文档里竟然没有标出）；
             binary = "/usr/bin/rsync", -- rsync可执行程序地址，默认/usr/bin/rsync
             archive = true, -- 默认false，以递归方式传输文件，并保持所有文件属性
             compress = true,-- 压缩传输默认为true。在带宽与cpu负载之间权衡，本地目录同步可以考虑把它设为false；
             verbose = true,--同步详细模式输出
        	 perms = true -- perms 保留文件权限,默认为true；
      }
}

#-- excludeFrom = "/etc/rsyncd.d/rsync_exclude.lst", 
#*.log
#/cache/
#/temp/
#.git/

```
