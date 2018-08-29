# python 性能分析

##1、 vprof  可以显示出执行时间火焰图、内存使用情况、代码块显示执行时间，  (换行 两个空格然后回车)
		github:https://github.com/nvdv/vprof

* 1、安装
	
	pip install vprof

* 2、使用
	vprof -c cmh 'filename *args '
	
	c - CPU flame graph  
	p - profiler  
      m - memory graph  
      h - code heatmap  


## 2、cProfile  

		import cProfile  
		cProfie.run('func()')



