#Install Hadoop on Docker single node
## Install Guide
>### Part_1 : @HDFS and YARN
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


