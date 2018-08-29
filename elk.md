# ELK
 
## 工作原理

项目主页： https://www.elastic.co

elk体系覆盖了日志数据收集，解析、索引建立，条件查询、页面展示。<br>

elk主要由三部分组成：

 * 1、数据收集（logstash）<br>
 	功能：
 	* 1、收集日志数据；
 	* 2、根据设定的正则规则解析日志内容
 	* 3、简单统计；
 	* 4、输出解析好的日志内容到下游。<br>下游分两类，一类是消息队列。另一类是索引器。
 	
 ---
 
 * 2、建立索引（elasticsearch）<br>
 	功能：
 	* 1、建立索引日志
 	* 2、根据查询条件，返回查询结果。
 	
 * 3、数据查询展示（kibana）<br>
 	功能：
 	* 1、页面展示。查询条件可定制，页面展示可定制
 	
---

## 系统架构

### 数据量小
	
	  数据收集         建立索引                        数据展示
	logstash1  =>                   <=  查询条件
	               elasticsearch                    kibana             
	                                =>  返回结果
	logstashn  =>
    
### 数据量大

	数据收集                   数据汇总      建立索引                 数据展示
	logstash1 =>                                     <=  查询条件
	             消息队列 => logstash => elasticsearch             kibana             
	                                                 =>  返回结果
	logstashn =>
    

说明：数据收集可以有许多不同的形式

---

# logstash

## 安装运行

 * 安装前提条件 java(基本运行条件)； ruby, rubygem 这两个是在安装插件时候需要  
 * 到下载页面中 https://www.elastic.co/downloads。解压。
 * 解压后的目录说明：
 	* bin 可执行文件
 	* config 配置文件目录。能用的是logstash.yml。logstash 的系统配置
 * 运行命令：
 	./bin/logstash -f xx -d
  * 线上部署
 	* 在11,12,13 /usr/local/sport/logstash-5.2.0 目录下。 
 	* 重启先找到进程ps -aux|grep logstash. 然后kill
 	* 配置文件位于项目路径下的conf/logfile.conf

---

## 可定制配置文件

	input {
     	# 输入源配置
	}

	filter {
     	#  过滤信息
	}

	output {
     	# 结果输出
	}

	input, filter , output 都有相应的插件进行工作。根据自己实际需要，选择相关插件。具体配置需要依据所选择的插件而定

	输入源一般有 beat, file, redis, kafka
	输出一般有 file, kafka, redis, file, elastic search 

---
	
## 插件讲解
### input

#### file插件

file  插件可以处理日志文件切割的情况。因为是定期查看文件变化。
	
	主要的配置参数：
		path => [“path1”, “path2”]   #日志文件具体地址。
		exclude => “xx”     # 日志文件目录中需要过滤掉那些规则的文件。
		sincedb_path => “/dev/null” # 日志文件已经处理过的位置。本例中是不记录已经处理过的。测试中用的。实际中可以不配
		start_position => “beginning” or “end” # 日志文件是从头开始（全量）处理， 还是从当前文件末尾（增量）处理
		stat_interval => 1  # 默认1秒钟去看一下日志文件是否发生了更改。
		delimiter => “\n” # 默认分隔符
		discover_interval => 15 # 默认15秒去日志目录文件中去查找是否有新的文件生成。 只有在日志为目录或则 模糊规则前提下有用。
		add_field => {} #  添加额外的数据
		tags => []  # 添加额外的标签

---
#### beats插件
	
	主要配置:
		host => “192.168.xx.xx” # log stash 部署的IP地址。beats 客户端和logstash 都部署在一台机器可以不配置，分开就得配置。
		port => 5044  # 监听的端口号
		add_field => {} #  添加额外的数据
		tags => []  # 添加额外的标签
---

##插件讲解
### filter插件

#### grok 

grok根据是正则表达式来解析日志数据。logstash grok 插件有若干已经定义好的正则规则。可以借鉴使用
https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns. 

匹配规则
%{正则: field名称}  

	grok {
     	match => { “message” => { “%{正则: field名称} ....%{正则: field名称}” } }  # 匹配规则
     	patterns_dir => []  #  自己定义的匹配正则规则目录
	}

---
##插件讲解
### filter插件
#### aggregate  
有事件，才有统计结果。统计周期内没有事件到来，统计结果不产生。
需要人为安装。注意，安装前要先安装ruby 。否则安装不成功。<br>
安装命令： bin/logstash-plugin install logstash-filter-aggregate.

    主要配置：
		task_id => “%{field}” # 按那个字段来统计
		code => “” # 统计处理规则
		end_of_task => true | false  # 关闭统计  这个针对的是有开始 结束标志的 统计事件
		timeout => 60  #  统计周期  这个是针对没有明确开始，结束标志的统计  
		push_map_as_event_on_timeot => true  # 统计时间持续时间到了后，立即统计结果作为event 事件
		push_previous_map_as_event_b => true # 统计时间持续时间到了后，再下次事件到来时发送统计结果event 事件
		aggregate_maps_path => “xx” #  统计中间结果存放地址。防止统计周期中的服务停止，再开启后，统计结果清零。
		
---
##插件讲解
### filter插件
#### mutate 数据修改插件

	主要配置：
		covert => { “field”, “integer”| “float” | “string”| “boolean”} # 转换字段格式类型
		rename => { “field_old”, “field_new”} # 更改字段名称
		merge => { “dest_field” => “added_field”} # 合并字段
		replace => {“field”=> “xxx”} # 更改字段内容
		update => { “field” => “”} # 更新字段内容。只有字段已经存在才有效。
		add_tag => [ ] # 添加标签
		add_field => {}  # 添加字段
		gsub => [ 
 			“field1”, “需要替换内容”， “替换后的内容”，
 			“field2”, “需要替换内容”， “替换后的内容” ，
		]               # 根据替换规则处理string 字段。


---
##插件讲解
### output 插件
####elasticsearch  
	
	主要配置：
		hosts => [“uri1”, “url2" ]  #   ip:port
		index => “” # 索引文件名称。  尽量的把索引文件分细点。便于kibana 查找。

		absolute_healthcheck_path => true | false   # 检查elastic search  服务状态
		healthcheck_path => “/“ # elastic search 健康检查地址

		action => “index” | “delete” | “create” | “update”  #  默认index  。
			index 把document( logstash 中的event json格式 )建立索引
			delete 删除索引
			create 如果document id 已经存在，创建索引时会出错。
			update 根据document id更新索引

		cacert => “”   # certificate 文件地址。 和elastic search  服务器进行安全认证

		document_id => “” # 记录id 默认可以不写，系统自己赋值
 
		flush_size => # 缓冲区大小。超过缓冲区大小写一次索引。
		idle_flush_time => 1 # 缓冲区刷新时间

---

# elasticsearch

## 安装运行
 
 * 到下载页面中 https://www.elastic.co/downloads。解压。
 * 解压后的目录说明：
 	* bin 可执行文件
 	* config 配置文件目录。
 		* elasticsearch.yml。elasticsearch 的系统配置。
 		* jvm.options 。虚拟机运行可以修改jvm 的虚拟内存大小。
 	* data 索引文件目录。 可通过elasticsearch.yml来更改
 	* logs 日志文件目录。可通过elasticsearch.yml来更改

---

 * 运行
 	./bin/elasticsearch -d 
 * 线上部署
 	* 在21 /usr/local/sport/elk/elasticsearch-5.2.0 目录下。 
 	* 通过 nohup 方式运行。 重启先找到进程ps -aux|grep elk. 然后kill
 	* data, log 目录 在/opt/sports/elk/elasticsearch/
 
---

# kibana

## 安装运行
 * 到下载页面中 https://www.elastic.co/downloads。解压。
 * 解压后的目录说明：
 	* bin 可执行文件
 	* config 配置文件目录。kibana.yml。kibana 的系统配置。配置说明：
 		* server.basePath: "/kibana" 。 nginx 转发请求时需要重写url .把添加的/kibana 给去掉。
 * 运行
 	./bin/kibana  默认 5061 端口
 * 线上部署在21 /usr/local/sport/elk/kibana-5.2.0 目录下。 通过 nohup 方式运行。 重启先找到进程
    netstat -aonp|grep 5601 然后kill
	
---

## 查询

整体流程：

 * 1、首先指定以后的查询是基于那些索引文件的。 (对应页面的discover和management)
 * 2、创建自己定制的可视化查询项目。(对应页面的visualize) 
 * 3、在页面中，组合展示步骤2中创建的可视化查询项目。(对应页面的dashboard)
 
 查询条件样例
字符串字段: 正则  url:/api/v1/(android|iphone)/*   <br>
数字字段: >, <, !=, 判断条件  response:>1  response:<1 <br>
 
多个查询条件可以通过AND 或则 OR 组合  

---

### 指定索引文件名称 匹配格式

根据索引文件名称来建立查询入口。即告诉elasticsearch 我需要从哪些文件中查找数据。<br>

如何添加？<br>

点击 management 菜单项 => 选择 Index Patterns  => 填写索引日志文件的正则规则  => save <br>

创建好的去哪里查询 ？ <br>

点击 discover => 在页面左侧会有创建好的 索引文件规则

---

### 创建可视化统计项目

整体步骤：<br>
点击 visualize => 选择数据展示形式 => 选择索引规则名称  => 组织数据筛选条件 => 保存设置 (点击save)

#### area chart
面积图
Y轴 是默认为数量 
X轴 可以选择时间， 统计区间

splist Area 可以有多个颜色的面积。 看项目例子  chart_splitarea_test

splist chart 把比较的指标分别作为一个图片显示。看项目例子 chart_splitchart_test
####table
split Row  分行  看项目例子  table_splitrows_test
split table 分表  看项目例子  table_splittable_test
#### heatmap  chart
可以分多列，每列多行 看项目例子  chart_hitmap_test

#### line chart
线图 可以查看例子 line_chart_test

#### pie chart 
饼图 看例子 host

----

### 组合显示

根据上边已经组织好的数据自己拖拉。然后点击save 进行面版的保存工作。




