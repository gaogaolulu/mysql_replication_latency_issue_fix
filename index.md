 操作系统： redhat 7  64bit
硬件： 主机 从机相同, 8 核  72 GB 内存 
网络： 10 GB 
mysql： 从 8.0.11 升级到  8.0.22  
cluster： 一主 一从， streaming replication
问题描述： 8.0.11  主从工作正常， 升级后， 从机 慢， 延迟 不断增加， 没有减少， 当将主机停机后， 从机延迟消失 。 
主机配置： 使用 binlog group commit ,  相关参数 已经增大 
从机配置： 有 128 个thread 来 应用 日志

binlog_transaction_dependency_history_size=100000 
binlog_group_commit_sync_delay=100000 
binlog_group_commit_sync_no_delay_count=256
slave_parallel_workers=128 
slave_pending_jobs_size_max = 1073741824
slave_preserve_commit_order=1

问题分享：
1. 网络延迟： 没有， 监控 从机的 relay log 生成， 速度很快， 从机还没有应用到的 日志 ，积累很多。 网络带宽 为 120 MB/ 秒， 
2. 主机 CPU / 内存：  cpu 占有率 8 核的  3 核 满负载运行。， 内存 50% - 70% .
3. 数据库表主键：  不缺失 。都有。 
4. 数据库sql 性能：  有问题， 有很多 运行很久的 超过 10 秒的 sql ，有的长达 3 分钟。 优化脚本后， 延迟无明显改善 
5. 从机 CPU / 内存： cpu 占有率 极低 ， 内存 10%  占用率 以下  ， 
6. 检查 从机等待事件：  
 event_name                                   
 
 wait/synch/cond/sql/Worker_info::jobs_cond   
 wait/synch/cond/sql/MYSQL_BIN_LOG::COND_done 
 

7 从机显示 waiting for handler commit,  执行过的 position 非常慢， 每秒大概执行  40 左右， 而写入 速度每秒大概 200 左右，这导致 延迟非常严重。
8. 从机统计 :  负载平摊在  大约 10 - 16 个 thread ， 

select sum( COUNT_STAR ) from
  ( select performance_schema.events_transactions_summary_by_thread_by_event_name.THREAD_ID AS THREAD_ID,
performance_schema.events_transactions_summary_by_thread_by_event_name.COUNT_STAR AS COUNT_STAR
from performance_schema.events_transactions_summary_by_thread_by_event_name
where performance_schema.events_transactions_summary_by_thread_by_event_name.THREAD_ID
in (select performance_schema.replication_applier_status_by_worker.THREAD_ID
from performance_schema.replication_applier_status_by_worker)   ) tablea  ;
 

到此， 分析完所有可能后，  走入绝境， 查不到问题根源。 
根据  MYSQL_BIN_LOG::COND_done  等待事件 ， 应该是 binary log 写入延迟， 但是这个是在从机上， 难道这个有问题？ 查找 mysql 8 文档，  log_slave_updates 缺省值是  OFF ，即关闭 从机 binary log 写入。 
从机上 检查 后  
show variables like 'log_slave_updates';  发现是 打开的， 修改 /etc/my.cnf  设置为:

log_slave_updates=OFF 

重启后， 发现执行的 速度非常快， 每秒大概 500 到 1000 ， 查看 CPU 占用率 ， 上升到 50%， 内存占用率 上升到 40%。　
等待事件只有：　
 event_name                                   
 
 wait/synch/cond/sql/Worker_info::jobs_cond   

binary log 等待事件 基本消失： 

 wait/synch/cond/sql/MYSQL_BIN_LOG::COND_done 
 




