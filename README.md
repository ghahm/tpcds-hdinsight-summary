원본 - https://github.com/hdinsight/tpcds-hdinsight

을 참고하여 Orc 대신 Parquet 파일로 변환하여 수행하는 코드로 수정한 버전입니다.

테스트 환경 : Spark 2.1 버전의 클러스터

(주의) 
Spark 1.6 버전에서 테스트 쿼리의 에러가 발행하며 그 이유는 Spark에서 Hive Metastore 정보를 인식하지 못하기 때문임. 그 원인은 Spark 1.6 버전에서 Thrift server의 기본값이 multi-session mode인데 이로 인해 Hive에서 생성한 Temp table을 Spark에서 인식 못하는 것임.

# 1. Clone this repo
   git clone https://github.com/hdinsight/tpcds-hdinsight/ && cd tpcds-hdinsight
# 2. Run TPCDSDataGen.hql with settings.hql file and set the required config variables
   hive -i settings.hql -f TPCDSDataGen.hql -hiveconf SCALE=500 -hiveconf PARTS=500 -hiveconf LOCATION=/HiveTPCDS/ -hiveconf TPCHBIN=resources
# 3. Create tables on the generated data
   hive -i settings.hql -f ddl/createAllExternalTables.hql -hiveconf LOCATION=/HiveTPCDS/ -hiveconf DBNAME=tpcds
# 4. Generate Parquet data (사이즈에 따라 시간이 많이 소요됨)
   hive -i settings.hql -f ddl/createAllParquetTables.hql -hiveconf PARQUETDBNAME=tpcds_pqt -hiveconf SOURCE=tpcds
# 5. Run a test query
   beeline -u "jdbc:hive2://\`hostname -f\`:10002/tpcds_pqt;transportMode=http" -n "" -p "" -i settings.hql -f queries/query12.sql <br>
   
   (주의 : 원본에는 10001 포트로 되어 있으나 10002로 수정하여 실행할 것. 10001은 Hive용 포트이고 10002는 Spark용 포트인데 10001로 실행할 경우 SQL의 Parsing 에러 발생. 테스트 쿼리가 SparkQL Syntax라 Hive Syntax에 맞지 않는 것으로 보임)
# 5. run the queries with Spark (10회 실행)
   for f in queries/*.sql; do for i in {1..10} ; do STARTTIME="`date +%s`";  beeline -u "jdbc:hive2://\`hostname -f\`:10002/tpcds_pqt;transportMode=http" -i sparksettings.hql -f $f  > $f.run_$i.out 2>&1 ; SUCCESS=$? ; ENDTIME="`date +%s`"; echo "$f,$i,$SUCCESS,$STARTTIME,$ENDTIME,$(($ENDTIME-$STARTTIME))" >> times_pqt.csv; done; done; <br>

### 참고
생성된 csv를 기본 container의 /hive 밑으로 복사하기<br>
* hadoop fs -put *.csv /hive
