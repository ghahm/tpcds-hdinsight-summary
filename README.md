# tpcds-hdinsight-summary
tpcds-hdinsight 요약

원본 - https://github.com/hdinsight/tpcds-hdinsight

1. Clone this repo
   git clone https://github.com/hdinsight/tpcds-hdinsight/ && cd tpcds-hdinsight
2. Run TPCDSDataGen.hql with settings.hql file and set the required config variables.
   hive -i settings.hql -f TPCDSDataGen.hql -hiveconf SCALE=500 -hiveconf PARTS=500 -hiveconf LOCATION=/HiveTPCDS/ -hiveconf TPCHBIN=resources
3. create tables on the generated data
   hive -i settings.hql -f ddl/createAllExternalTables.hql -hiveconf LOCATION=/HiveTPCDS/ -hiveconf DBNAME=tpcds
4. generate Parquet data (사이즈에 따라 시간이 많이 소요됨)
   hive -i settings.hql -f ddl/createAllParquetTables.hql -hiveconf PARQUETDBNAME=tpcds_pqt -hiveconf SOURCE=tpcds
5. run the queries with Spark (10회 실행)
   for f in queries/*.sql; do for i in {1..10} ; do STARTTIME="`date +%s`";  beeline -u "jdbc:hive2://`hostname -f`:10002/tpcds_pqt;transportMode=http" -i sparksettings.hql -f $f  > $f.run_$i.out 2>&1 ; SUCCESS=$? ; ENDTIME="`date +%s`"; echo "$f,$i,$SUCCESS,$STARTTIME,$ENDTIME,$(($ENDTIME-$STARTTIME))" >> times_pqt.csv; done; done;
