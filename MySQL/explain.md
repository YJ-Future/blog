# explain语句
1. MySQL使用嵌套循环（nested-loop）处理所有的连接（join）
2. 柱	含义
   id	SELECT标识符
   select_type	SELECT类型
   table	输出行的表
   partitions	匹配的分区
   type	连接类型
   possible_keys	可供选择的索引
   key	实际选择的指数
   key_len	所选键的长度
   ref	列与索引进行比较
   rows	估计要检查的行
   filtered	按表条件过滤的行的百分比
   Extra	附加信息

