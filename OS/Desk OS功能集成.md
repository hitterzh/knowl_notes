Desk OS功能集成



#### 1.systemd：

```
/etc/init.d/apache2 start
or service apache2 start
```

缺点：一是启动时间长，`init`进程是串行启动，只有前一个进程启动完，才会启动下一个进程；二是启动脚本复杂，`init`进程只是执行启动脚本，脚本需要自己处理各种情况，这往往使得脚本变得很长。

Systemd 就是为了解决这些问题而诞生的。它的设计目标是，为系统的启动和管理提供一套完整的解决方案。Systemd 取代了`initd`，成为系统的第一个进程（PID 等于 1），其他进程都是它的子进程。

**命令组：**systemctl、systemd-analyze查看启动耗时、hostnamectl主机的信息、localectl本地化设置、timedatectl、loginctl

**资源Unit：**systemctl list-units

- Service unit：系统服务
- Target unit：多个 Unit 构成的一个组
- Device Unit：硬件设备
- Mount Unit：文件系统的挂载点
- Automount Unit：自动挂载点
- Path Unit：文件或路径
- Scope Unit：不是由 Systemd 启动的外部进程
- Slice Unit：进程组
- Snapshot Unit：Systemd 快照，可以切回某个快照
- Socket Unit：进程间通信的 socket
- Swap Unit：swap 文件
- Timer Unit：定时器

```
# 列出正在运行的 Unit
$ systemctl list-units
# 列出所有Unit，包括没有找到配置文件的或者启动失败的
$ systemctl list-units --all
# 列出所有没有运行的 Unit
$ systemctl list-units --all --state=inactive
# 列出所有加载失败的 Unit
$ systemctl list-units --failed
# 列出所有正在运行的、类型为 service 的 Unit
$ systemctl list-units --type=service
# 显示系统状态
$ systemctl status
# 显示单个 Unit 的状态
$ sysystemctl status bluetooth.service
# 显示远程主机的某个 Unit 的状态
$ systemctl -H root@rhel7.example.com status httpd.service
除了status命令，systemctl还提供了三个查询状态的简单方法，主要供脚本内部的判断语句使用。
# 显示某个 Unit 是否正在运行
$ systemctl is-active application.service
# 显示某个 Unit 是否处于启动失败状态
$ systemctl is-failed application.service
# 显示某个 Unit 服务是否建立了启动链接
$ systemctl is-enabled application.service
对于用户来说，最常用的是下面这些命令，用于启动和停止 Unit（主要是 service）。

# 立即启动一个服务
$ sudo systemctl start apache.service

# 立即停止一个服务
$ sudo systemctl stop apache.service

# 重启一个服务
$ sudo systemctl restart apache.service

# 杀死一个服务的所有子进程
$ sudo systemctl kill apache.service

# 重新加载一个服务的配置文件
$ sudo systemctl reload apache.service

# 重载所有修改过的配置文件
$ sudo systemctl daemon-reload

# 显示某个 Unit 的所有底层参数
$ systemctl show httpd.service

# 显示某个 Unit 的指定属性的值
$ systemctl show -p CPUShares httpd.service

# 设置某个 Unit 的指定属性
$ sudo systemctl set-property httpd.service CPUShares=500
systemctl list-dependencies命令列出一个 Unit 的所有依赖。

$ systemctl list-dependencies nginx.service
上面命令的输出结果之中，有些依赖是 Target 类型（详见下文），默认不会展开显示。如果要展开 Target，就需要使用--all参数。

$ systemctl list-dependencies --all nginx.service
```

**Unit配置：**

Systemd 默认从目录`/etc/systemd/system/`读取配置文件。但是，里面存放的大部分文件都是符号链接，指向目录`/usr/lib/systemd/system/`，真正的配置文件存放在那个目录。

```
systemctl enable命令用于在上面两个目录之间，建立符号链接关系。

$ sudo systemctl enable clamd@scan.service
# 等同于
$ sudo ln -s '/usr/lib/systemd/system/clamd@scan.service' '/etc/systemd/system/multi-user.target.wants/clamd@scan.service'
与之对应的，systemctl disable命令用于在两个目录之间，撤销符号链接关系，相当于撤销开机启动。

$ sudo systemctl disable clamd@scan.service
配置文件的后缀名，就是该 Unit 的种类，比如sshd.socket。如果省略，Systemd 默认后缀名为.service，所以sshd会被理解成sshd.service。
```

```
[Unit]区块通常是配置文件的第一个区块，用来定义 Unit 的元数据，以及配置与其他 Unit 的关系。它的主要字段如下。

Description：简短描述
Documentation：文档地址
Requires：当前 Unit 依赖的其他 Unit，如果它们没有运行，当前 Unit 会启动失败
Wants：与当前 Unit 配合的其他 Unit，如果它们没有运行，当前 Unit 不会启动失败
BindsTo：与Requires类似，它指定的 Unit 如果退出，会导致当前 Unit 停止运行
Before：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之后启动
After：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之前启动
Conflicts：这里指定的 Unit 不能与当前 Unit 同时运行
Condition...：当前 Unit 运行必须满足的条件，否则不会运行
Assert...：当前 Unit 运行必须满足的条件，否则会报启动失败
[Install]通常是配置文件的最后一个区块，用来定义如何启动，以及是否开机启动。它的主要字段如下。

WantedBy：它的值是一个或多个 Target，当前 Unit 激活时（enable）符号链接会放入/etc/systemd/system目录下面以 Target 名 + .wants后缀构成的子目录中
RequiredBy：它的值是一个或多个 Target，当前 Unit 激活时，符号链接会放入/etc/systemd/system目录下面以 Target 名 + .required后缀构成的子目录中
Alias：当前 Unit 可用于启动的别名
Also：当前 Unit 激活（enable）时，会被同时激活的其他 Unit
[Service]区块用来 Service 的配置，只有 Service 类型的 Unit 才有这个区块。它的主要字段如下。

Type：定义启动时的进程行为。它有以下几种值。
Type=simple：默认值，执行ExecStart指定的命令，启动主进程
Type=forking：以 fork 方式从父进程创建子进程，创建后父进程会立即退出
Type=oneshot：一次性进程，Systemd 会等当前服务退出，再继续往下执行
Type=dbus：当前服务通过D-Bus启动
Type=notify：当前服务启动完毕，会通知Systemd，再继续往下执行
Type=idle：若有其他任务执行完毕，当前服务才会运行
ExecStart：启动当前服务的命令
ExecStartPre：启动当前服务之前执行的命令
ExecStartPost：启动当前服务之后执行的命令
ExecReload：重启当前服务时执行的命令
ExecStop：停止当前服务时执行的命令
ExecStopPost：停止当其服务之后执行的命令
RestartSec：自动重启当前服务间隔的秒数
Restart：定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog
TimeoutSec：定义 Systemd 停止当前服务之前等待的秒数
Environment：指定环境变量
Unit 配置文件的完整字段清单，请参考官方文档。
```

它与`init`进程的主要差别如下。

> （1）默认的 RunLevel（在`/etc/inittab`文件设置）现在被默认的 Target 取代，位置是`/etc/systemd/system/default.target`，通常符号链接到`graphical.target`（图形界面）或者`multi-user.target`（多用户命令行）。
>
> （2）启动脚本的位置，以前是`/etc/init.d`目录，符号链接到不同的 RunLevel 目录 （比如`/etc/rc3.d`、`/etc/rc5.d`等），现在则存放在`/lib/systemd/system`和`/etc/systemd/system`目录。
>
> （3）配置文件的位置，以前`init`进程的配置文件是`/etc/inittab`，各种服务的配置文件存放在`/etc/sysconfig`目录。现在的配置文件主要存放在`/lib/systemd`目录，在`/etc/systemd`目录里面的修改可以覆盖原始设置。

https://www.cnblogs.com/zwcry/p/9602756.html