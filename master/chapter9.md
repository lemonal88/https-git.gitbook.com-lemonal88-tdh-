# Spark streaming

stream是一套专门处理流数据的框架，由于实际需求很大，例如营销系统，节点传感器监控，实时报警等，它也是TDH4.3版本中变化最大的一个功能

stream的每个行业的数据结构都不一样，每个流进来以后，数据会自动进表（和文件）
1、stream————kafka————spark（holodesk on ssd）
2、stream————kafka————HDFS中的表————HiveQL查询分析