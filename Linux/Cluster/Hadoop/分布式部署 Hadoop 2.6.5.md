#### 环境说明
```txt
因虚拟机资源限制，将SN,NN,YARN仍放在1个节点 (Node1) , 需注意Hadoop集群中各节点间ntp同步及各个节点的主机名调整

Node1(192.168.0.3)作为Master：   NN，SNN，YARN(RsourceManager) 
Node2-4(192.168.0.4/7/8)作为：   DN(NodeManager)


    [Node1] ----- [Node2]
           \ ---- [Node3]
            \ --- [Node4]
```
#### 在部署Hadoop集群前先在所有节点执行如下
```bash
[root@localhost ~]# systemctl stop firewalld && setenforce 0    #生产环境要写入配置
[root@localhost ~]# date -s "yyyy-mm-dd HH:MM"      #生产环境要使用ntpdate，若不进行同步在执行YARN任务时会报错
[root@localhost ~]# yum -y install java-1.7.0-openjdk.x86_64 java-1.7.0-openjdk-devel.x86_64
[root@localhost ~]# echo "export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.161-2.6.12.0.el7_4.x86_64" \
> /etc/profile.d/java.sh && . /etc/profile.d/java.sh
[root@localhost ~]# tar -zxf hadoop-2.6.5.tar.gz -C /
[root@localhost ~]# chown -R root.root /hadoop-2.6.5/
[root@localhost ~]# ln -sv /hadoop-2.6.5/ /hadoop
[root@localhost ~]# cat > /etc/profile.d/hadoop.sh <<'eof'
export HADOOP_PREFIX="/hadoop"                          
export PATH=$PATH:$HADOOP_PREFIX/bin:$HADOOP_PREFIX/sbin
export HADOOP_COMMON_HOME=${HADOOP_PREFIX}              
export HADOOP_HDFS_HOME=${HADOOP_PREFIX}                
export HADOOP_MAPRED_HOME=${HADOOP_PREFIX}              
export HADOOP_YARN_HOME=${HADOOP_PREFIX}   
eof
[root@localhost ~]# . /etc/profile
[root@localhost ~]# cat >> /etc/hosts <<eof
192.168.0.3   node1 master
192.168.0.4   node2
192.168.0.7   node3
192.168.0.8   node4
eof
[root@localhost ~]# groupadd hadoop                                  
[root@localhost ~]# useradd hadoop -g hadoop        #此处仅使用hadoop用户
[root@localhost ~]# echo "123456" | passwd --stdin hadoop            

#使所有Hadoop集群中的节点能够以"hadoop"用户的身份进行免密钥互通（Master启动时将通过SSH的方式启动各节点的daemon进程）
[root@localhost ~]# su - hadoop                                                                       
[hadoop@localhost ~]$ ssh-keygen -t rsa -P ''                                                         
[hadoop@localhost ~]$ for ip in 3 4 7 8;do ssh-copy-id -i .ssh/id_rsa.pub hadoop@192.168.0.${ip};done 
[hadoop@localhost ~]$ for nd in 1 2 3 4;do ssh node${nd} "echo test" ;done
[hadoop@localhost ~]$ exit
[root@localhost ~]# mkdir -p  /data/hadoop/hdfs/{nn,snn,dn}         #创建HDFS各角色使用的元数据及块数据等路径
[root@localhost ~]# chown -R hadoop.hadoop /data/hadoop/hdfs/       #修改属主属组是必须的...
[root@localhost ~]# cd /hadoop
[root@localhost hadoop]# mkdir logs
[root@localhost hadoop]# chmod -R g+w /hadoop/logs                  #日志路径...
[root@localhost hadoop]# chown -R hadoop.hadoop /hadoop/            #
[root@localhost hadoop]# chown -R hadoop.hadoop /hadoop             #软连接
[root@localhost hadoop]# ll /hadoop
lrwxrwxrwx. 1 hadoop hadoop 14 1月  12 07:00 /hadoop -> /hadoop-2.6.5/
```
#### 在Hadoop集群中的Master节点，即本文环境中的NN,SNN,YARN节点（node1）执行如下
```bash
[root@node1 hadoop]# vim etc/hadoop/core-site.xml
<configuration>
    <!--指定NameNode地址，即Hadoop集群中HDFS的RPC服务端口-->
    <property>
        <name>fs.defaultFS</name>                                   #
        <value>hdfs://node1:8020/</value>                           #指明HDFS的Master节点访问接口（监听的RPC端口）
        <final>true</final>
    </property>
 </configuration>
 
[root@node1 hadoop]# vim etc/hadoop/yarn-site.xml  #用于配置YARN进程及其相关属性，本文件中的node1指的是yarn节点地址
<configuration>
    <!--Hadoop集群中YARN的ResourceManager守护所在的主机和监听的端口，对于伪分布式模型来讲其主机为localhost-->
    <property>    
        <name>yarn.resourcemanager.address</name> 
        <value>node1:8032</value>                           
    </property>
    <!--Hadoop集群中YARN的ResourceManager使用的scheduler（作业任务的调度器）-->
    <property>    
        <name>yarn.resourcemanager.scheduler.address</name> 
        <value>node1:8030</value>
    </property>
    <!--资源追踪器的地址-->
    <property>    
        <name>yarn.resourcemanager.resource-tracker.address</name> 
        <value>node1:8031</value>
    </property>
    <!--YARN的管理地址-->
    <property>    
        <name>yarn.resourcemanager.admin.address</name> 
        <value>node1:8033</value>
    </property>
    <!--YARN的内置WEB管理地址提供服务的地址及端口-->
    <property>    
        <name>yarn.resourcemanager.webapp.address</name> 
        <value>node1:8088</value>
    </property>
    <!--nomenodeManager获取数据的方式是shuffle(辅助服务)-->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <!--使用的shuffleHandler类-->
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <!--使用的调度器的类-->
    <property>
        <name>yarn.resourcemanager.scheduler.class</name> 
        <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
    </property>
<configuration>

[root@node1 hadoop]# vim etc/hadoop/hdfs-site.xml
#主要用于配置HDFS相关属性，如复制因子（数据块副本数），NN和DN用于存储数据的路径等...
<configuration>
    <!--指定hdfs保存数据的副本数量,即Hadoop集群中HDFS的DN下的数据冗余份数，对于伪分布式的Hadoop应设为1-->
    <property>
        <name>dfs.replication</name>
        <value>2</value>                                                        #
    </property>
    <!--指定hdfs中namenode的存储位置，数据的目录为前面的步骤中专门为其创建的路径-->
    <property>
        <name>dfs.namenode.name.dir</name> 
        <value>file:///data/hadoop/hdfs/nn</value>
    </property>
    <!--指定hdfs中datanode的存储位置，数据的目录为前面的步骤中专门为其创建的路径-->
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///data/hadoop/hdfs/dn</value>
    </property>
    <!--设置hdfs中checkpoint即SNN的追加日志文件路径，数据的目录为前面的步骤中专门为其创建的路径-->
    <property>
        <name>fs.checkpoint.dir</name>
        <value>file:///data/hadoop/hdfs/snn</value>
    </property>
    <!--设置hdfs中checkpoint即的编辑目录-->
    <property>
        <name>fs.checkpoint.edits.dir</name>
        <value>file:///data/hadoop/hdfs/snn</value>
    </property>
    <!--若需要其他用户对HDFS有写入权限，还需要再添加如下属性的定义-->
    <property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>

#mapred-site.xml默认不存在，但有模块文件mapred-site.xml.template，只需要将其复制为mapred-site.xml即可
[root@node1 hadoop]# cp etc/hadoop/mapred-site.xml.template etc/hadoop/mapred-site.xml
[root@node1 hadoop]# vim etc/hadoop/mapred-site.xml
<!-- 用于配置集群的MapReduce framework，此处应该指定使用yarn，另外的可用值还有：local/classic -->
<configuration>
    <!--告诉hadoop的MR(Map/Reduce)运行在YARN上(version 2.0+)，而不是自己直接运行-->
    <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
    </property>
</configuration>

[root@localhost ~]# cat > etc/hadoop/masters <<eof
node1
eof

[root@node1 hadoop]# cat > etc/hadoop/slaves <<eof
node2
node3
node4
eof
```
#### 在Hadoop集群中的Master节点将配置文件使用hadoop用户推送到集群中的各节点
```bash
[root@node1 hadoop]# su - hadoop 
[hadoop@node1 ~]$ cd /hadoop/etc/hadoop
[hadoop@node1 ~]$ for n in {1..4};do scp core-site.xml  hadoop@node${nd}::/hadoop/etc/hadoop/core-site.xml ;done   
[hadoop@node1 ~]$ for n in {1..4};do scp yarn-site.xml  hadoop@node${nd}::/hadoop/etc/hadoop/yarn-site.xml ;done 
[hadoop@node1 ~]$ for n in {1..4};do scp mapred-site.xml  hadoop@node${nd}::/hadoop/etc/hadoop/mapred-site.xml ;done 
[hadoop@node1 ~]$ for n in {1..4};do scp hdfs-site.xml  hadoop@node${nd}::/hadoop/etc/hadoop/hdfs-site.xml ;done 
[hadoop@node1 ~]$ for n in {1..4};do scp slaves  hadoop@node${nd}::/hadoop/etc/hadoop/slaves ;done 
[hadoop@node1 ~]$ for n in {1..4};do scp masters  hadoop@node${nd}::/hadoop/etc/hadoop/masters ;done
#core-site.xml，mapred-site.xml，hdfs-site.xml，master，slave等配置文件在各节点中都是一样的
```
#### 启动 Hadoop Cluster
```bash
[root@node1 hadoop]# su - hadoop                        #先在Master节点格式化NN
[hadoop@node1 ~]$ hdfs namenode -format

# 有2种启动方式：
#     1. 在各节点分别启动要启动的服务
#     2. 在Master节点使用Apache官方提供的脚本启动整个集群

[root@node1 ~]# su - hadoop
[hadoop@node1 ~]$ start-dfs.sh                          #自动通过配置信息到所有NN/DN启动（关闭使用：stop-dfs.sh）
Starting namenodes on [node1]
node1: starting namenode, logging to /hadoop/logs/hadoop-hadoop-namenode-node1.out
node4: starting datanode, logging to /hadoop/logs/hadoop-hadoop-datanode-node4.out
node2: starting datanode, logging to /hadoop/logs/hadoop-hadoop-datanode-node2.out
node3: starting datanode, logging to /hadoop/logs/hadoop-hadoop-datanode-node3.out
Starting secondary namenodes [0.0.0.0]                  #因为不存在snn，它直接找0.0.0.0去了 (实际上找的是本机)
hadoop@0.0.0.0\'s password: 
0.0.0.0: starting secondarynamenode, logging to /hadoop/logs/hadoop-hadoop-secondarynamenode-node1.out  

[root@node3 hadoop]# su - hadoop -c "jps"               #此时在集群的其他节点执行jps可以看到dn已经启动    
46321 Jps
46248 DataNode

[hadoop@node1 ~]$ start-yarn.sh                         #在Master节点启动YARN，此操作也会将所有DN节点的NM启动
starting yarn daemons
starting resourcemanager, logging to /hadoop/logs/yarn-hadoop-resourcemanager-node1.out
node2: starting nodemanager, logging to /hadoop/logs/yarn-hadoop-nodemanager-node2.out
node3: starting nodemanager, logging to /hadoop/logs/yarn-hadoop-nodemanager-node3.out
node4: starting nodemanager, logging to /hadoop/logs/yarn-hadoop-nodemanager-node4.out

[root@node4 hadoop]# su - hadoop -c "jps"               #在其他节点验证nodemanager是否启动
46004 DataNode
46120 NodeManager
46265 Jps
```
#### 在Master节点通过YARN执行软件包内Apache官方提供的MapReduce的 "wordcount"（单词统计）jar包测试运行状态
```bash
[hadoop@node1 ~]$ hdfs dfs -mkdir /test
[hadoop@node1 ~]$ hdfs dfs -ls -R /
drwxr-xr-x   - hadoop supergroup          0 2017-01-13 21:36 /test
[hadoop@node1 ~]$ hdfs dfs -put /etc/fstab /test/fstab
```
###### 通过HDFS内建的WEB页面查看
![img](资料/Hadoop分布式WEB.gif)
```bash
[hadoop@node1 mapreduce]$ yarn jar /hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.5.jar \
> wordcount /test/fstab /test/fstab.analyze.out
18/01/13 21:52:36 INFO client.RMProxy: Connecting to ResourceManager at node1/192.168.0.3:8032
18/01/13 21:52:37 INFO input.FileInputFormat: Total input paths to process : 1
18/01/13 21:52:37 INFO mapreduce.JobSubmitter: number of splits:1
18/01/13 21:52:37 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1484314634480_0002
18/01/13 21:52:37 INFO impl.YarnClientImpl: Submitted application application_1484314634480_0002
18/01/13 21:52:37 INFO mapreduce.Job: The url to track the job: http://node1:8088/proxy/application_1484314634480_0002/
18/01/13 21:52:37 INFO mapreduce.Job: Running job: job_1484314634480_0002
18/01/13 21:52:46 INFO mapreduce.Job: Job job_1484314634480_0002 running in uber mode : false
18/01/13 21:52:46 INFO mapreduce.Job:  map 0% reduce 0%
18/01/13 21:52:52 INFO mapreduce.Job:  map 100% reduce 0%
18/01/13 21:53:02 INFO mapreduce.Job:  map 100% reduce 100%
18/01/13 21:53:02 INFO mapreduce.Job: Job job_1484314634480_0002 completed successfully
18/01/13 21:53:02 INFO mapreduce.Job: Counters: 49
        File System Counters
                FILE: Number of bytes read=555
                FILE: Number of bytes written=215267
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=558
                HDFS: Number of bytes written=397
                HDFS: Number of read operations=6
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=2
        Job Counters 
                Launched map tasks=1
                Launched reduce tasks=1
                Other local map tasks=1
                Total time spent by all maps in occupied slots (ms)=4836
                Total time spent by all reduces in occupied slots (ms)=5752
                Total time spent by all map tasks (ms)=4836
                Total time spent by all reduce tasks (ms)=5752
                Total vcore-milliseconds taken by all map tasks=4836
                Total vcore-milliseconds taken by all reduce tasks=5752
                Total megabyte-milliseconds taken by all map tasks=4952064
                Total megabyte-milliseconds taken by all reduce tasks=5890048
        Map-Reduce Framework
                Map input records=11
                Map output records=54
                Map output bytes=589
                Map output materialized bytes=555
                Input split bytes=93
                Combine input records=54
                Combine output records=38
                Reduce input groups=38
                Reduce shuffle bytes=555
                Reduce input records=38
                Reduce output records=38
                Spilled Records=76
                Shuffled Maps =1
                Failed Shuffles=0
                Merged Map outputs=1
                GC time elapsed (ms)=496
                CPU time spent (ms)=2610
                Physical memory (bytes) snapshot=441872384
                Virtual memory (bytes) snapshot=2112487424
                Total committed heap usage (bytes)=277348352
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters 
                Bytes Read=465
        File Output Format Counters 
                Bytes Written=397
```
###### 通过YARN内建的WEB页面查看
![img](资料/MP-result.gif)
```bash
[hadoop@node1 mapreduce]$ hdfs dfs -ls -R /test
-rw-r--r--   2 hadoop supergroup        465 2017-01-13 21:37 /test/fstab
drwxr-xr-x   - hadoop supergroup          0 2018-01-13 21:53 /test/fstab.analyze.out
-rw-r--r--   2 hadoop supergroup          0 2018-01-13 21:53 /test/fstab.analyze.out/_SUCCESS
-rw-r--r--   2 hadoop supergroup        397 2018-01-13 21:53 /test/fstab.analyze.out/part-r-00000
[hadoop@node1 mapreduce]$ hdfs dfs -cat /test/fstab.analyze.out/part-r-00000
#       7
'/dev/disk'     1
/       1
/boot   1
/dev/mapper/centos-root 1
/dev/mapper/centos-swap 1
/etc/fstab      1
0       6
06:32:32        1
20      1
2017    1
Accessible      1
Created 1
Mon     1
Nov     1
See     1
UUID=2ad33313-a113-4cf8-863a-da3c9e79e4f0       1
anaconda        1
and/or  1
are     1
blkid(8)        1
by      2
defaults        3
filesystems,    1
findfs(8),      1
for     1
fstab(5),       1
info    1
maintained      1
man     1
more    1
mount(8)        1
on      1
pages   1
reference,      1
swap    2
under   1
xfs     2
```
#### 附 YARN 命令相关参数
```txt
[root@node4 hadoop]# yarn
Usage: yarn [--config confdir] COMMAND
where COMMAND is one of:
  resourcemanager -format-state-store   deletes the RMStateStore        #删除RM的状态存储
  resourcemanager                       run the ResourceManager
  nodemanager                           run a nodemanager on each slave
  timelineserver                        run the timeline server
  rmadmin                               admin tools
  version                               print the version
  jar <jar>                             run a jar file
  application                           prints application(s)
                                        report/kill application
  applicationattempt                    prints applicationattempt(s)
                                        report
  container                             prints container(s) report
  node                                  prints node report(s)
  queue                                 prints queue information
  logs                                  dump container logs
  classpath                             prints the class path needed to
                                        get the Hadoop jar and the
                                        required libraries
  daemonlog                             get/set the log level for each
                                        daemon
 or
  CLASSNAME                             run the class named CLASSNAME
Most commands print help when invoked w/o parameters.
```
