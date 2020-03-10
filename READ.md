1. HDFS와 MapReduce를 이용해서 wordCount하기.
2. HDFS와 Hive 연동하기 + Stream 이용해보기

# 이론
## 1. Hive
- 정의 : 하이브는 하둡 에코시스템 중에서 데이터를 모델링하고 프로세싱하는 경우 가장 많이 사용하는 데이터 웨어하우징용 솔루션 이다.<br>
- 특징<br>
* File 기반 테이블 이다. 그래서 기본적으로 테이블의 모든 ROW 정보를 읽는다.
* Partition을 나눠서 데이터를 저장할 수 있다. 조회 속도를 높이기 위한 대처.

### What is Data WareHousing?
1. `What is Data WearHouse?`<br>
: 효율적으로 분석 가능한 형태로 정보들이 저장되어 있는 중앙 저장소이다. 데이터 웨어 하우스는 관계형 데이터베이스(RDB), 트랜잭션 시스템 등, 다양한 시스템으로부터 정기적으로 데이터를 수집하는 개념이며, 이와 같이 수집된 데이터를 비즈니스 인텔리전스 도구, SQL 클라이언트와 같이 데이터를 분석하는 목적으로 사용한다. 줄여서 *DW*라고 부른다.
<br>

2. `Data WareHousing Structure` :<br>
* Bottom Tier : 데이터 서버로서 데이터를 Load 및 Store 한다.
* Middle Tier : 분석 엔진이 있는 영역으로 데이터 Access 및 분석을 담당한다.
* Top Tier : 분석한 결과를 리포팅하는 등, 결과를 가시화 하는 단계이다.

### 1-1. Hive Service
- 정의 :  하이브는 편리한 작업을 위한 몇가지 서비스를 제공한다.<br>
* MetaStore<br> 
: 메타스토어 서비스는 *HDFS 데이터 구조를 저장*하는 실제 데이터베이스를 가지고 있다. 3가지 실행모드가 있다. 테스트로 동작할 때는 임베이디드 모드를 사용하고, 실제 운영에서는 리모트 모드 또는 로컬 모드를 사용한다.<br>

* Beline<br>
: SQLLine 기반의 하이브서버2(Hiveserver2)에 접속하여 Query를 실행하기 위한 도구이다. JDBC를 사용하여 하이브서버2에 접속한다.<br>
#### Beline 접속
```sh
$ beeline
beeline > !connect jdbc:hive2://localhost:10000/default
0: jdbc:hive2://localhost:10000/default > 
```

### 1-2. Hive Database
- 정의 : 하이브의 데이터베이스는 테이블의 이름을 구별하기 위한 네임 스페이스(Namespace) 역할을 한다. 그리고 테이블의 데이터의 기본 저장위치를 제공한다.<br>
- Database 기본 설정
```sh
# 기본 위치
hive.metastore.warehouse.dir=hdfs:///user/hive/
vi {HIVE_INSTALL_HOME}/conf/hive-site.xml
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
    <description>location of default database for the warehouse</description>
  </property>

# 데이터베이스의 기본 위치
hdfs:///user/hive/{데이터베이스명}.db

# 테이블의 기본 위치
hdfs:///user/hive/{데이터베이스명}.db/{테이블명}
```

- Database 생성
```sh
> CREATE (DATABASE|SCHEMA) [IF NOT EXISTS] database_name
   [COMMENT database_comment]
   [LOCATION hdfs_path]
   [WITH DBPROPERTIES (property_name=property_value, ...)];
```

- Database 수정
```sh
# PROPERTY 추가
> ALTER (DATABASE|SCHEMA) database_name SET DBPROPERTIES (property_name=property_value, ...);

# 권한 수정
> ALTER (DATABASE|SCHEMA) database_name SET OWNER [USER|ROLE] user_or_role;

# HDFS PATH 수정
> ALTER (DATABASE|SCHEMA) database_name SET LOCATION hdfs_path; 
```

- Database 삭제
```sh
> DROP (DATABASE|SCHEMA) [IF EXISTS] database_name [RESTRICT|CASCADE];
```

### 1-3. Hive Table
- 정의 : 하이브에서 Table은 HDFS 상에 저장된 파일과 디렉토리 구조에 대한 메타 정보라 할 수 있다. 실제 저장된 파일의 구조에 대한 정보와 저장위치, 입력 포맷, 출력 포맷, 파티션 정보, 프로퍼티에 대한 정보 등 다양한 정보를 가지고 있다.

* 입력(Insert)과 조회(Select)
- 입력과 조회는 하이브 테이블의 메타 정보를 이용하여 데이터를 읽고, 쓰는 작업을 한다. 테이블의 메타 정보에 실제 파일의 로케이션과 데이터의 포맷이 저장되어 있다. 이 정보를 이용하여 파일을 읽고, 쓰게 된다.
- 입력(Insert)은 INSERT문으로 테이블에 데이터를 쓰는 방법과 테이블의 저장위치, 테이블을 생성할 때 지정한 위치에 있는 파일을 복사하는 방법이 있다.
- 조회(Select)은 테이블의 LOCATION의 위치에 있는 파일을 조회한다. 하이브는 LOCATION 아래의 모든 파일을 조회 하기 때문에 용량이 큰 파일을 저장한다면 파티션을 이용하여 데이터를 분산하여 주는 것이 좋다.
- 파일 To 테이블 : 파일을 읽어서 테이블의 LOCATION에 쓰는 방법이다.
```sh
# 1. LOAD 명령을 이용하는 방법
## Load 명령으로 파일을 읽어서 테이블에 쓰면 테이블의 LOCATION에 데이터가 저장된다. HDFS와 로컬 데이터를 테이블에 쓸 수 있고, 파티션 추가도 가능하다.
> LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE table_name [PARTITION(partcol1=val1, partcol2=val2 ...)];
## hdfs의 파일을 읽어서 tb1 테이블의 파티션 yyyymmdd='20180510'으로 입력
> LOAD DATA INPATH 'hdfs://127.0.0.1/user/data/sample.csv' INFO TABLE tb1 PARTITION(yymmdd='20180510');
## 로컬의 파일을 읽어서 tb1 테이블에 입력
> LOAD DATA LOCAL INPATH '/home/hduser/ml-m1/user.data' INTO TABLE tc1;

# 2. Table의 Location에 파일을 복사하는 방법
# - 테이블의 메타정보에는 물리적인 파일의 위치를 위한 LOCATION이 존재한다. 테이블을 조회할 때 해당 파일을 읽기 때문에 이 위치에 파일을 복사하면 데이터를 입력한 것과 동일한 역할을 한다. 로컬 위치는 접근이 불가능하고, HDFS나, S3 등 하둡에서 접근 가능한 파일 공유 시스템을 이용해야 한다.
## 테이블 생성 시 LOCATION 설정
> CREATE TABLE employee(
        id      STRING,
        name    STRING,
) LOCATION 'hdfs://127.0.0.1/user/data';

## 테이블 생성 후, ALTER 명령으로 LOCATION 설정
> CREATE TABLE employee(
        id      STRING,
        name    STRING);
> ALTER TABLE employee SET LOCATION 'hdfs://127.0.0.1/user/data';

# 3. 테이블 파티션을 추가/수정하여 LOCATION을 파일 위치로 주는 방법
# - 파티션 테이블은 파티션 별로 LOCATION을 가지고 있다.
> ALTER TABLE employee ADD PARTITION (yyyymmdd='20180510') LOCATION 'hdfs://127.0.0.1/user/';

# 4. 테이블 To 테이블
# - 테이블의 정보를 읽어서 다른 테이블에 입력하는 방법.
## 기본적인 데이터 입력 방식으로 테이블, 뷰의 데이터를 다른 테이블에 입력한다.
## 기본문법
> INSERT OVERWRITE TABLE tablename1 PARTITION
> ... [IF NOT EXISTS] select_statement1 FROM from_statement;

> INSERT OVERWRITE TABLE tablename1 PARTITION
> ... select_statement1 FROM from_statement;

> INSERT OVERWRITE TABLE users_2
>       SELECT
>               TRANSFORM(userid, gender, age, occupation, zipcode)
>               USING 'python occupation_mapper.py'
>               AS(userid, gender, age, occupation_str, zipcode)
>       FROM
>               users;

# 5. From INSERT 문
# - FROM INSERT문은 여러 테이블에 한번에 입력할 때 사용한다. FROM 절에 원천 데이터를 조회하여 뷰처럼 사용할 수 있다.
> FROM page_view_stg pvs
> INSERT OVERWRITE TABLE page_view PARTITION(dt='2008-06-08', country)
>       SELECT pvs.ip, pvs.country;

> FROM(
>       SELECT *
>       FROM source1
>       UNION
>       SELECT *
>       FROM source2
> ) R
> INSERT INTO TABLE target1
> SELECT R.name,
>        R.age
>
> INSERT INTO TABLE target2
> SELECT R.name,
>        R.age;
















```

> Source `occupation_mapper.py`
```python
import sys
occupation_dict = {
    0:  "other or not specified",
    1:  "academic/educator",
    2:  "artist",
    3:  "clerical/admin",
    4:  "college/grad student",
    5:  "customer service",
    6:  "doctor/health care",
    7:  "executive/man sagerial",
    8:  "farmer",
    9:  "homemaker",
    10:  "K-12 student",
    11:  "lawyer",
    12:  "programmer",
    13:  "retired",
    14:  "sales/marketing",
    15:  "scientist",
    16:  "self-employed",
    17:  "technician/engineer",
    18:  "tradesman/craftsman",
    19:  "unemployed",
    20:  "writer"
}

for line in sys.stdin:
    line = line.strip()
    userid, gender, age, occupation, zipcode = line.split('#')
    occupation = int(occupation)
    occupation_str = occupation_dict[occupation]
    print '#'.join([userid, gender, age, occupation_str, zipcode])
```


#Install Hadoop on Docker single node
>`4개의 프로세스` : Name node, Data node, Job tracker, Test tracker
## 1. Hello, Hadoop!
- Hadoop : HDFS(Hadoop 분산 파일 시스템)과 mapReduce(맵리듀스)가 합쳐진 용어 <br>
|-- HDFS :  분산 파일 시스템(데이터 저장소)<br>
|-- mapReduce : Raw 데이터(key-value 방식)의 데이터를 map에 저장 후, 데이터를 reduce(=데이터 filtering) 한다.<br>
- Hive : Hadoop에서 HiveQL(SQL)를 사용하여 *실시간 데이터 조회 및 Raw데이터 집계*에 사용되는 라이브러리<br>
*HiveQ로 정의한 내용을 Hive가 MapReduce Job으로 변환해서 실행
        
## 2. Install Guide
>### Part_1 : @HDFS and YARN
## 1. Hello Hadoop~

## 2. Install Guide
1. Install docker
2. Install ubuntu:latest
```sh
//if not exist ubuntu image, auto download image
$ docker run -d -it --name hadoop_hive ubuntu:latest /bin/bash
```

3. Access Console on docker ubuntu
```sh
$ docker exec -it hadoop_hive /bin/bash
```

4. Update apt-get and Download needed module 
```sh
// Update and Download
$ apt-get update
$ apt-get install -y software-properties-common wget sudo vim ssh rsync git
// Add repository
$ add-apt-repository ppa:webupd8team/java
```

5. Install java
- Oracle의 라이센스 정책으로 인해서 'apt install oracle-java8-installer'가 아닌 다른 방식으로 install.
```sh
// Java
$ apt-get install openjdk-8-jdk
// Check complete install
$ java -version
openjdk version "1.8.0_242"
OpenJDK Runtime Environment (build 1.8.0_242-8u242-b08-0ubuntu3~18.04-b08)
OpenJDK 64-Bit Server VM (build 25.242-b08, mixed mode)
```

6. Start SSH service
```sh
$ service ssh start
```

7. Add Config JAVA Environment Varivable
```sh
$ vim /etc/profile
//Add config
...
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export PATH=$PATH:$JAVA_HOME/bin
export CLASS_PATH="."
//wq

source /etc/profile
```

8. Download package Hadoop and Hive
```sh
cd /tmp
$ wget http://mirror.navercorp.com/apache/hadoop/common/hadoop-2.9.2/hadoop-2.9.2
$ tar -xvf hadoop-2.9.2.tar.gz
$ tar -xvf apache-hive-2.3.6-bin.tar.gz
$ mkdir /usr/local/hadoop
$ mkdir /usr/local/hive
$ pwd
/tmp
$ ls -al
drwxrwxr-x 2 hadoop hadoop      4096 Mar  4 08:45 apache-hive-2.3.6-bin
-rw-rw-r-- 1 hadoop hadoop 232225538 Aug 22  2019 apache-hive-2.3.6-bin.tar.gz
drwxr-xr-x 2 hadoop hadoop      4096 Mar  4 08:45 hadoop-2.9.2
-rw-rw-r-- 1 hadoop hadoop 366447449 Nov 20  2018 hadoop-2.9.2.tar.gz
$ mv hadoop-2.9.2/* /usr/local/hadoop
$ mv apache-hive-2.3.6-bin/* /usr/local/hive
```

9. Modify Configuration Hadoop for Pseudo-Distributed Operation
* hadoop-env.sh : 환경변수 설정
* core-site.xml : HDFS와 MapReduce에서 공통적으로 사용할 환경정보 설정
* hdfs-site.xml : HDFS에서 사용할 환경정보 설정
* mapred-site.xml : MapReduce에서 사용할 환경정보 

```sh
$ cd /usr/local/hadoop
$ pwd
/usr/local/hadoop
$ vim .etc/hadoop/hadoop-env.sh
# 25Line
JAVA_HOME=JAVA_INSTALL_FULL_PATH
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://localhost:9000</value>
        </property>
        <property>
                <name>hadoop.proxyuser.hduser.groups</name>
                <value>*</value>
        </property>
        <property>
                <name>hadoop.proxyuser.hduser.hosts</name>
                <value>*</value>
        </property>
</configuration>
//:wq
$ vim etc/hadoop/hdfs-site.xml
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
</configuration>
//:wq

$ vim etc/hadoop/yarn-site.xml
<configuration>
<!-- Site specific YARN configuration properties -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
//:wq
$ cp etc/hadoop/mapred-site.xml.template etc/hadoop/mapred-site.xml
$ vim etc/hadoop/mapred-site.xml
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
//:wq

$ adduser hduser
$ adduser hduser sudo
$ tail /etc/passwd
$ tail /etc/group

$ su hduser
$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ ssh localhost
hduser@localhost $ exit
```

10. namenode format setting
```sh
$ cd /usr/local/hadoop
$ bin/hdfs namenode -format
$ sbin/start-dfs.sh
$ sbin/start-yarn.sh
```

11. Start service HDFS and YARN
```sh
$ pwd
/usr/local/hadoop
# Service Start cmd
$ sbin/start-dfs.sh
$ sbin/start-yarn.sh

# Check Service Status 
$ jps
9218 SecondaryNameNode
9394 ResourceManager
9831 Jps
9036 DataNode
8895 NameNode
9503 NodeManager

# Test Create directory at HDFS(Hadoop Distribute File System)
# 1. Make Directory 
$ bin/hdfs dfs -mkdir -p /user/hduser
# 2. Check Maked Dir
$ bin/hdfs dfs -ls /user
ound 1 items
drwxr-xr-x   - root supergroup          0 2020-03-05 04:32 /user/hduser
```
<br>
<br>
<br>
<br>

>### Part_2 : @Hive
##1. Hello Hive?
* Hive는
##2. Install Guide
1. HDFS에 필요한 파일 생성 및 권한 수정
```sh
# Change Dir Owner and Group
$ sudo chown -R hduser:hadoop /usr/local/hive

# HDFS내에서 Hive가 사용할 데이터저장소를 생성
$ bin/hdfs dfs -mkdir /tmp
$ bin/hdfs dfs -mkdir -p /user/hive/warehouse

# 파일권한 수정
$ bin/hdfs dfs -chmod g+w /tmp
$ bin/hdfs dfs -chmod g+w /user/hive/warehouse

# 파일 생성 확인
$ bin/hdfs dfs -ls /user/hive
Found 1 items
drwxrwxr-x   - root supergroup          0 2020-03-05 05:18 /user/hive/warehouse
```

2. Edit Hive configure 
```sh
$ pwd
/usr/local/hive/conf

# Copy:/ Template
$ cp hive-env.sh.template hive-env.sh
$ cp hive-default.xml.template hive-site.xml
$ hive-exec-log4j2.properties.template hive-exec-log4j2.properties
$ mv hive-log4j2.properties.template hive-log4j2.properties

# 1. Edit hive-env.sh
$ vim hive-env.sh
export HADOOP_HOME=/usr/local/hadoop
export HIVE_CONF_DIR=/usr/local/hive/conf
//:wq

# 2. Change param "value" in hive-site.xml
$ vim hive-site.xml
<property>
    <name>hive.exec.local.scratchdir</name>
    *<value>/tmp/hduser</value>*
    <description>Local scratch space for Hive jobs</description>
</property>
<property>
    <name>hive.downloaded.resources.dir</name>
    *<value>/tmp/hduser/hive/${hive.session.id}_resources</value>*
    <description>Temporary local directory for added resources in the remote file system.</description>
</property>
//:wq
```

3. 메타데이터를 보관할 저장소 설정(옵션: RDBMS or derby 데이터베이스)
- Repository URL
mysql-connector :
https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.9/mysql-connector-java-5.1.9.jar
```sh
$ cd ../
$ pwd
/usr/local/hive
$ bin/schematool -dbType derby -initSchema
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hive/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Metastore connection URL:	 jdbc:derby:;databaseName=metastore_db;create=true
Metastore Connection Driver :	 org.apache.derby.jdbc.EmbeddedDriver
Metastore connection User:	 APP
Starting metastore schema initialization to 2.3.0
Initialization script hive-schema-2.3.0.derby.sql
Initialization script completed
schemaTool completed
```

4. Execute Hiveserver2
```sh
# 1. Start hiveserver2
$ bin/hiveserver2

# 2. Open new cmd
$ docker exec -it hadoop_hive /bin/bash

# 3. Exec beeline at Ubuntu on Docker Container
$ su hduser
$ cd /usr/local/hive
$ bin/beeline
Beeline > !connect jdbc:hive2://localhost:10000/default
Connecting to jdbc:hive2://localhost:10000/default
Enter username for jdbc:hive2://localhost:10000/default: APP
Enter password for jdbc:hive2://localhost:10000/default: ***
Connected to: Apache Hive (version 2.3.6)
Driver: Hive JDBC (version 2.3.6)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://localhost:10000/default>
# -- end --- hive2에 접속 완료

# hive 정상 설치 확인
jdbc:hive2://localhost:10000/default> show tables;
+-----------+
| tab_name  |
+-----------+
+-----------+
No rows selected (1.793 seconds)
```
<br>
<br>
<br>
<br>

>### Part_3 : @Data Injection
1. Download 공공 데이터셋(부동산 실거래가.csv)
2. Copy Host에 저장된 공공 데이터셋 --> Docker Container
```sh
$ docker cp ./DATA_SET_FILE.csv dockerContainerName:/home/hduser
```
3. 원본 데이터(Row Data)를 HDFS에 복사 
```sh
$ su - root
$ cd /usr/local/hadoop
$ pwd
/usr/local/hadoop
$ bin/hdfs dfs -mkdir /input
$ bin/hdfs dfs -chown hduser:hadoop /user/hduser
$ bin/hdfs dfs -ls /user
$ su - hduser
$ bin/hdfs dfs -put CSV_DATA_PATH
```
4. Create Table in Hive 데이터셋에 맞게 Table 생성
```sh
```

5. 분석(집계)

6. API를 이용한 데이터 조회



























