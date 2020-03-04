#Install Hadoop on Docker single node
## Install Guide
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
$ wget http://mirror.navercorp.com/apache/hadoop/common/hadoop-2.9.2/hadoop-2.9.2.tar.gz
$ wget http://mirror.navercorp.com/apache/hive/hive-2.3.6/apache-hive-2.3.6-bin.tar.gz
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

```







