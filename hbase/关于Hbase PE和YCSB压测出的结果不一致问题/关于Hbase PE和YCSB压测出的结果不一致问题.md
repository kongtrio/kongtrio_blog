./bin/ycsb run hbase20 -P workloads/hbase_workload -p table=usertable -p columnfamily=cf -p operationcount=1600000 -threads 16 -s
YSCB:
```
[OVERALL], RunTime(ms), 62102
[OVERALL], Throughput(ops/sec), 25764.065569546874
[TOTAL_GCS_PS_Scavenge], Count, 30
[TOTAL_GC_TIME_PS_Scavenge], Time(ms), 75
[TOTAL_GC_TIME_%_PS_Scavenge], Time(%), 0.12076905735725098
[TOTAL_GCS_PS_MarkSweep], Count, 1
[TOTAL_GC_TIME_PS_MarkSweep], Time(ms), 20
[TOTAL_GC_TIME_%_PS_MarkSweep], Time(%), 0.03220508196193359
[TOTAL_GCs], Count, 31
[TOTAL_GC_TIME], Time(ms), 95
[TOTAL_GC_TIME_%], Time(%), 0.15297413931918458
[CLEANUP], Operations, 32
[CLEANUP], AverageLatency(us), 35.65625
[CLEANUP], MinLatency(us), 0
[CLEANUP], MaxLatency(us), 814
[CLEANUP], 95thPercentileLatency(us), 39
[CLEANUP], 99thPercentileLatency(us), 814
[CLEANUP], 99.9PercentileLatency(us), 814
[CLEANUP], 99.99PercentileLatency(us), 814
[CLEANUP], 99.999PercentileLatency(us), 814
[UPDATE], Operations, 1600000
[UPDATE], AverageLatency(us), 612.815261875
[UPDATE], MinLatency(us), 349
[UPDATE], MaxLatency(us), 82943
[UPDATE], 95thPercentileLatency(us), 887
[UPDATE], 99thPercentileLatency(us), 1344
[UPDATE], 99.9PercentileLatency(us), 6055
[UPDATE], 99.99PercentileLatency(us), 40063
[UPDATE], 99.999PercentileLatency(us), 55039
[UPDATE], Return=OK, 1600000
```
hbase pe --valueSize=1000 --nomapred --rows=100000 randomWrite 16
2020-12-12 16:22:06,339 INFO  [main] hbase.PerformanceEvaluation: [RandomWriteTest duration ]	Min: 6007ms	Max: 6669ms	Avg: 6404ms
2020-12-12 16:22:06,339 INFO  [main] hbase.PerformanceEvaluation: [ Avg latency (us)]	63
2020-12-12 16:22:06,339 INFO  [main] hbase.PerformanceEvaluation: [ Avg TPS/QPS]	249985	 row per second