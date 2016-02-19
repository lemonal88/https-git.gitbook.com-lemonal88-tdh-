# 第十章：Spark streaming

　　stream是一套专门处理流数据的框架，由于实际需求很大，例如营销系统，节点传感器监控，实时报警等，它也是TDH4.3版本中变化最大的一个功能

　　stream对于每个行业的数据结构都不一样，每个流进来以后，数据会自动进表（和文件）

　　（1）、stream————kafka————spark（holodesk on ssd）

　　（2）、stream————kafka————HDFS中的表————HiveQL查询分析


###spark streaming和storm的区别
　　根本区别时在于他们的处理模型，storm处理的是每次传入的一个事件，而spark streaming是处理某个时间段窗口内的事件流，storm处理一个事件可以达到秒级延迟，而spark streaming则有几秒钟的延迟