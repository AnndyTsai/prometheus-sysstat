# Jmeter/LR+Prometheus+Grafana+Sysstat构建性能监控系统

## 1：前言

### 1.1：Jmeter/LR能满足性能测试吗？

* Jmeter/LR肯定是可以满足一般的互联网企业的性能测试需求的
* Jmeter/LR能满足性能测试需求 但是未必同时也是优秀的监控工具
* 我们知道Jmeter也可以用来监控一些服务器的资源 但是这个数据很粗糙 只能看到一些最基本的情况或者Fail Pass的情况 看不到更深层次的数据，既然没有这些数据 我们怎么去分析性能瓶颈，怎么取针对性地优化？
* 综上我们在做性能测试的时候同时引入Prometheus+Grafana+Sysstat这些系统来收集数据 配合这些数据可以快速定位待瓶颈 开发才可以跟快速地定位问题


### 1.2：性能测试为什么要引入Prometheus+Grafana+Sysstat这样的监控系统

* Prometheus+Grafana一般被用于监控线上环境的一套组合系统,它可以监控到跟深层面 不仅仅是监控一些服务器资源这么简单
* Prometheus相对一些其他的监控工具 它的精度更高 更底层，采集数据的准确率那是杠杆地
* Sysstat一般被用来监控服务器硬件资源情况 在性能结合它的使用可以帮助我们快速分析性能瓶颈 方便开发做针对性的优化
* 综上 在性能测试时使用这套系统的目的就是提供尽可能详近的数据 方便相关人员分析数据 发现瓶颈 针对优化
* 关于Jmeter或者LoadRunner怎么去做性能测试 这个就不描述了


## 2:环境的安装

### 2.1：prometheus-Server端和node_exporter的安装

#### 2.1.1:设置时区

* timedatectl set-timezone Asia/Shanghai
* contab -e 添加命令：* * * * * ntpdate -u cn.pool.ntp.org
* 重启命令：service crond restart

#### 2.1.2:Prometheus-Server端的安装

* 下载prometheus-xxxx.linux-amd64.tar.gz
* 解压 tar -zxvf prometheus-xxxx.linux-amd64.tar.gz
* 进入prometheus文件夹
* 后台启动：setsid ./prometheus 或者setsid ./prometheus --config.file="prometheus.yml"
* prometheus-Server端默认启动在9090端口

#### 2.1.3:node_exporter的安装

* 下载node_exporter的tar包
* 解压
* 进入目录
* 后台运行：setsid ./node_exporter

#### 2.1.4:添加node_exporter到prometheus-Server

* 进入prometheus目录 打开prometheus.yml文件 添加如下内容
(```)
  - job_name: 'node_exporter'
    static_configs:
      - targets:
        - 'xx.xx.xx.xx:9100'
(```)
* 重启prometheus
	1：ps -ef|grep prometheus
	2: kill -9 PID
* 注意:安装prometheus-Server的机器不用再安装node_exporter 只需用在node机器上安装node_exporter 然后添加到prometheus-Server的prometheus.yml文件中即可

### 2.1:Grafana的安装与配置

#### 2.1.1：Grafana的安装

* 下载网站https://grafana.com/grafana/download
* Cnetos安装
	1：wget https://dl.grafana.com/oss/release/grafana-6.3.2-1.x86_64.rpm 
	2：sudo yum localinstall grafana-6.3.2-1.x86_64.rpm 
	3：启动server ：systemctl start grafana-server

	
#### 2.1.2：Grafana的基本配置

* Add data source-->Prometheus-->Settings中设置Name、URL 然后save and Test没报错即可
* Add data source-->Prometheus-->Dashboards中导入必须的3个dashboards



### 2.2:Sysstat的安装与使用

#### 2.2.1：Sysstat的安装与配置

* yum list sysstat 查看sysstat是否已被安装
* yum install -y sysstat
* 进入/etc/cron.d/目录下 打开sysstat文件 修改为每分钟运行一次收集数据
![sysstat-1](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/sysstat-1.png "修改收集数据精度")
* 重启sysstat：systemctl restart sysstat
* 生成的sa文件会默认保存在 /var/log/sa/路径下
	1：sa文件为每分钟生成的数据
	2: sar位每天生成的summary文件
	
#### 2.2.2：Sysstat的基本使用(excel文档上传到了github上 项目地址:)
	
##### 2.2.2.1：Sysstat的监控-CPU监控

* 查看excel sysstat.xls

##### 2.2.2.2：Sysstat的监控-内存监控

* 查看excel sysstat.xls

##### 2.2.2.3：Sysstat的监控-IO监控

* 查看excel sysstat.xls

##### 2.2.2.4：Sysstat的监控-NetWork监控

* 查看excel sysstat.xls


## 3:prometheus的使用

* prometheus的使用建立在prometheus与node_exporter均配置正常的情况下

### 3.1：监控公式的使用

### 3.1.1：CPU使用率的计算

#### 3.1.1.1：在计算CPU的使用率我们需要知道下列知识
*	1：CPU状态有8总 常见的是用户态和内核态 IO等待态以及idle状态
*	2：单位时间的CPU使用率计算：(内核态时间+用户态时间+其他5种状态时间)/单位时间x100% 
*	3：注意上面分子是没有包含idle状态的时间的
	
#### 3.1.1.2：现在我们如何使用prometheus的公式计算一分钟内的CPU使用率呢？

* 监控cpu每分钟的idle率公式如下

```
sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance)
```

* 截图如下：

![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Prometheus1.png "cpu每分钟的使用率")

* 监控cpu每分钟的总的用户使用率公式如下

```
(1-((sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance)) / (sum(increase(node_cpu_seconds_total[1m])) by (instance))))*100
```

* 截图如下：

![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Prometheus2.png "cpu每分钟的使用率")

* 基于上述我们举一反三 获取其他CPU状态的使用率

	1：获取用户态每分钟的CPU使用率
	```
	(sum(increase(node_cpu_seconds_total{mode="user"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance))*100
	```
	
	2：获取内核每分钟的CPU使用率
	```
	(sum(increase(node_cpu_seconds_total{mode="system"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance))*100
	```
	
	3：获取IO等待态每分钟的CPU使用率
	```
	(sum(increase(node_cpu_seconds_total{mode="iowait"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance))*100
	```


	
## 4:Prometheus结合Grafana的配置


### 4.1：Grafana添加Prometheus

* Add data source-->Prometheus-->Settings中设置Name、URL 然后save and Test没报错即可
* Add data source-->Prometheus-->Dashboards中导入必须的3个dashboards
* 具体的设置请查考下列截图

![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Grafana-DataSiurce-setting1.png  "截图设置1")
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Grafana-DataSiurce-setting2.png  "截图设置2")
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Grafana-DataSiurce-setting3.png  "截图设置3")
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Grafana-DataSiurce-setting4.png  "截图设置4")

### 4.2:Grafana新添加dashboard

* 新建DashBoard
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Dashboard1.png  "新建DashBoard")

* 新建DashBoard
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Dashboard2.png  "新建DashBoard")

* 新建DashBoard查询的设置
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Dashboard3.png  "查询的设置")

* 新建DashBoard修改title和描述
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Dashboard4.png  "修改title和描述")

* 新建DashBoard视图的设置
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Dashboard5.png  "新建DashBoard")

* 保存

* 在刚刚新建的DashBoard上添加新的监控选项
![ext3](https://github.com/AnndyTsai/prometheus-sysstat/blob/master/Dashboard6.png  "添加监控选项")


## 5:Prometheus结合Grafana的使用

### 5.1：CPU的监控

* 针对CPU的监控 主要由以下三部分 具体Grafana的设置请参考4

	* 监控cpu每分钟的总的用户使用率公式如下
	```
	(1-((sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance)) / (sum(increase(node_cpu_seconds_total[1m])) by (instance))))*100
	```


	* 获取user态每分钟的CPU使用率(主要是应用测试启动太多 导致进程多 从而导致user态的CPU使用时间占比较高)
	```
	(sum(increase(node_cpu_seconds_total{mode="iowait"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance))*100
	```

	* 获取IO等待态每分钟的CPU使用率(当磁盘IO比较高的时候会出现IO等待时间较长 导致IOWAIT状态的CPU使用占比较高)
	```
	(sum(increase(node_cpu_seconds_total{mode="iowait"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance))*100
	```


### 5.2：Memory的监控

* 针对内存的监控 主要由剩余的可用内存组成
	1:剩余内存的计算：剩余可用内存=free+Buffers+Cached （）
	```
	(1-((node_memory_Buffers_bytes + node_memory_Cached_bytes + node_memory_MemFree_bytes)/node_memory_MemTotal_bytes))*100
	```
	
### 5.3：IO的监控

* 磁盘使用率
	```
	100 - (node_filesystem_free_bytes{mountpoint="/",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{mountpoint="/",fstype=~"ext4|xfs"} * 100)
	```

* 磁盘IO读写(1min数据读写的量 单位MB/S) 一般情况下IO高的情况下会导致CPU的IOWAIT升高
	```
	((rate(node_nfsd_disk_bytes_read_total[1m])+rate(node_nfsd_disk_bytes_written_total[1m]))/1024/1024) > 0
	```
	
* 网络IO传输的监控（网络读写IO的速率 Mb/S）
	```
	rate(node_network_transmit_bytes_total[1m])/1024/1024
	```
	
### 5.4：文件句柄的监控

* 文件句柄介绍
	1：文件句柄：文件句柄即文件描述符，每当一个进程打开一个文件时(Linux任何资源都可以看做文件)系统会分配一个唯一的整数型文件描述符 用来标示进程使用的这个文件，PID即文件句柄

* 文件句柄使用率
	```
	(node_filefd_allocated/node_filefd_maximum)*100
	```
	
## 6:Prometheus结合Grafana监控进程

### 6.1：process-exporter的安装配置

* 下载process-exporter：https://github.com/ncabatoff/process-exporter/releases/download/v0.4.0/process-exporter-0.4.0.linux-amd64.tar.gz

* 创建 process-name.yaml 文件

* 添加内容(具体怎么添加 参考https://www.cnblogs.com/danny-djy/p/11149818.html)

* 启动process-exporter：setsid ./process-exporter -config.path=process-name.yaml -web.listen-address=":8540" (启动在8540端口)

* 添加下列到Prometheus配置文件中prometheus.yml文件中 然后重启Prometheus
	```
	  - job_name: 'process_exporter'
		static_configs:
		- targets:
			- 'IP:8540'
	```
	
### 6.2：process-exporter的使用

* 进程的所有信息几乎都可以监控 这个就不再介绍了 baidu

## 7：结束语

* Jmeter/LR能完成对业务层的性能监控 但是结合Prometheus和sysstat可以完成对系统层面 硬件层面的监控
* Prometheus和sysstat能提供更底层的数据 让分析人员有数据可依
* Prometheus还能监控很多数据 socket TCP http https的数据 只要是Linux能提供的数据Prometheus都能监控
