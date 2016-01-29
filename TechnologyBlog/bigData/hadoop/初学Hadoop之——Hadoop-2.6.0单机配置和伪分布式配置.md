# 初学Hadoop之一： Hadoop单机和伪分布式配置
#### 参考：http://blog.csdn.net/ggz631047367/article/details/42426391 <div align = right>2016/1/20 </div>
--------------

#### 机器说明
ThinkPad T450 笔记本4G内存，64位windows10
通过VirtualBox安装32位的ubuntu-12.04.5-desktop-i386 分配1G内存
hadoop-2.6.0，JDK_1.7
其中遇到了很多问题，重装了两次系统，现在将成功经验说来：

### 0. 安装VirtualBox和unbuntu虚拟环境
略
### 1. 在Ubuntu下创建hadoop用户组和用户

hadoop的管理员最好就是以后要登录桌面环境运行eclipse的用户，否则后面会有拒绝读写的问题出现。当然不是也有办法办法解决。

创建hadoop用户组，并创建hadoop用户后增加权限：

	sudo addgroup hadoop  
	sudo adduser -ingroup hadoop hadoop  
	sudo vim /etc/sudoers 

在root   ALL=(ALL:ALL)   ALL下添加hadoop   ALL=(ALL:ALL)  ALL.

### 2. 在Ubuntu下安装JDK1.7
- 下载：
百度JDK1.7官方版，然后进入官方链接进行下载，需要同意其条约，下载jdk-7u79-linux-i586.tar.gz，要注意，不能用wget通过网址直接下载。
- 解压：
```
sudo mkdir /usr/lib/jvm
sudo tar zxvf /home/adam/Downloads/jdk-7u79-linux-i586.tar.gz -C /usr/lib/jvm
```
- 修改环境变量：
```
sudo vim /etc/profile 
```
然后在后面添加
```
export JAVA_HOME=/usr/lib/jvm/jdk1.7.0_79
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH 
```
然后执行
```
source /etc/profile 
```

### 3. 安装ssh服务 

	sudo apt-get install ssh openssh-server

- 建立ssh无密码登录本机
切换到hadoop用户，执行以下命令：
```
su - hadoop
```  
ssh生成密钥有rsa和dsa两种生成方式，默认情况下采用rsa方式。

- 创建ssh-key，，这里我们采用rsa方式：
```
ssh-keygen -t rsa -P "" （注：回车后会在~/.ssh/下生成两个文件：id_rsa和id_rsa.pub这两个文件是成对出现的）  
```
- 进入~/.ssh/目录下，将id_rsa.pub追加到authorized_keys授权文件中，开始是没有authorized_keys文件的;
```
cd ~/.ssh  
cat id_rsa.pub >> authorized_keys （完成后就可以无密码登录本机了。）  
```
- 登录localhost验证：
```
ssh localhost  
```
4. 执行退出命令;
```
exit  
```

### 4.安装hadoop 
- 把hadoop解压到/usr/local下:

```
wget http://apache.fayea.com/hadoop/common/stable/hadoop-2.6.0.tar.gz
sudo tar -zxvf hadoop-2.6.0.tar.gz  
sudo mv hadoop-2.6.0 /usr/local/hadoop  
sudo chmod -R 775 /usr/local/hadoop  
sudo chown -R hadoop:hadoop /usr/local/hadoop  //否则ssh会拒绝访问  
```

- 配置

修改bashrc的配置：

	sudo vim ~/.bashrc  

在文件末尾添加：
```
#HADOOP VARIABLES START  
  
export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_25  
  
export HADOOP_INSTALL=/usr/local/hadoop  
  
export PATH=$PATH:$HADOOP_INSTALL/bin  
export PATH=$PATH:$JAVA_HOME/bin   
export PATH=$PATH:$HADOOP_INSTALL/sbin  
export HADOOP_MAPRED_HOME=$HADOOP_INSTALL  
export HADOOP_COMMON_HOME=$HADOOP_INSTALL  
export HADOOP_HDFS_HOME=$HADOOP_INSTALL  
export YARN_HOME=$HADOOP_INSTALL  
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_INSTALL/lib/native  
export HADOOP_OPTS="-Djava.library.path=$HADOOP_INSTALL/lib"  
#HADOOP VARIABLES END  
```

执行下面命令使改动生效： 

	source ~/.bashrc

修改hadoop-env.sh的配置：

	sudo vim /usr/local/hadoop/etc/hadoop/hadoop-env.sh  

找到JAVA_HOME改为上面的值。

- 测试

通过执行hadoop自带实例WordCount验证是否安装成功

	cd /usr/local/hadoop  
	mkdir input  
	cp README.txt input  

在hadoop目录下执行WordCount：

	hadoop jar share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.6.0-sources.jar org.apache.hadoop.examples.WordCount input output  


#### 5. Hadoop伪分布式配置

sudo vim /usr/local/hadoop/etc/hadoop/core-site.xml

	<configuration>
		<property>
		    <name>hadoop.tmp.dir</name>
		    <value>/usr/local/hadoop/tmp</value>
		    <description>Abase for other temporary directories.</description>
		</property>
		<property>
		    <name>fs.defaultFS</name>
		    <value>hdfs://localhost:9000</value>
		</property>
    </configuration>

sudo vim /usr/local/hadoop/etc/hadoop/yarn-site.xml

	<property>
	<name>yarn.resourcemanager.hostname</name>
	<value>master</value>
	</property>
	<property>
	    <description>The address of the applications manager interface in the RM.</description>
	    <name>yarn.resourcemanager.address</name>
	    <value>${yarn.resourcemanager.hostname}:8032</value>
	</property>
	<property>
	    <description>The address of the scheduler interface.</description>
	    <name>yarn.resourcemanager.scheduler.address</name>
	    <value>${yarn.resourcemanager.hostname}:8030</value>
	  </property>
	<property>
	    <description>The http address of the RM web application.</description>
	    <name>yarn.resourcemanager.webapp.address</name>
	    <value>${yarn.resourcemanager.hostname}:8088</value>
	</property>
	<property>
	    <description>The https adddress of the RM web application.</description>
	    <name>yarn.resourcemanager.webapp.https.address</name>
	    <value>${yarn.resourcemanager.hostname}:8090</value>
	</property>
	<property>
	    <name>yarn.resourcemanager.resource-tracker.address</name>
	    <value>${yarn.resourcemanager.hostname}:8031</value>
	</property>
	<property>
	    <description>The address of the RM admin interface.</description>
	    <name>yarn.resourcemanager.admin.address</name>
	    <value>${yarn.resourcemanager.hostname}:8033</value>
	</property>
	<property>
	   <name>yarn.nodemanager.aux-services</name>
	   <value>mapreduce_shuffle</value>
	</property>

cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template /usr/local/hadoop/etc/hadoop/mapred-site.xml

sudo vim /usr/local/Hadoop/etc/hadoop/mapred-site.xml  //伪分布式不用配

	<property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
	</property>
	<property>
	  <name>mapreduce.jobhistory.address</name>
	  <value>master:10020</value>
	  <description>MapReduce JobHistory Server IPC host:port</description>
	</property>
	<property>
	  <name>mapreduce.jobhistory.webapp.address</name>
	  <value>master:19888</value>
	  <description>MapReduce JobHistory Server Web UI host:port</description>
	</property>

sudo vim /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```
<configuration>
<property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/dfs/data</value>
    </property>
    <property>                 //这个属性节点是为了防止后面eclopse存在拒绝读写设置的
            <name>dfs.permissions</name>
            <value>false</value>
     </property>
 </configuration>
```

sudo gedit /usr/local/hadoop/etc/hadoop/masters 添加：localhost

sudo gedit /usr/local/hadoop/etc/hadoop/slaves  添加：localhost

关于配置的一点说明：上面只要配置 fs.defaultFS 和 dfs.replication 就可以运行，不过有个说法是如没有配置 hadoop.tmp.dir 参数，此时 Hadoop 默认的使用的临时目录为 /tmp/hadoo-hadoop，而这个目录在每次重启后都会被干掉，必须重新执行 format 才行（未验证），所以伪分布式配置中最好还是设置一下。

配置完成后，首先在 Hadoop 目录下创建所需的临时目录：

	cd /usr/local/hadoop
	mkdir tmp dfs dfs/name dfs/data

修改pid目录位置（可以不做）

vim etc/hadoop/hadoop-env.sh

	export HADOOP_PID_DIR=/usr/local/hadoop-2.6.0/pid

vim etc/hadoop/hadoop-env.sh

	export HADOOP_MAPRED_PID_DIR=/usr/local/hadoop-2.6.0/pid

vim etc/hadoop/hadoop-env.sh

	export YARN_PID_DIR=/usr/local/hadoop-2.6.0/pid  


#### 6. 接着初始化文件系统HDFS。
	bin/hdfs namenode -format //每次执行此命令要把dfs/data/文件清空  

成功的话，最后的提示如下，Exitting with status 0 表示成功，Exitting with status 1: 则是出错。

	sbin/start-dfs.sh  
	sbin/start-yarn.sh  

开启Jobhistory

[html] view plain copy 在CODE上查看代码片派生到我的代码片
sbin/mr-jobhistory-daemon.sh  start historyserver  
<a target=_blank href="http://master:19888/">http://master:19888/</a>  


export HADOOP_OPTS="-Djava.library.path=$HADOOP_PREFIX/lib:$HADOOP_PREFIX/lib/native"
export HADOOP_OPTS="-Djava.library.path=$HADOOP_PREFIX/lib:$HADOOP_PREFIX/lib/native"


16/01/20 16:13:32 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Starting namenodes on [Java HotSpot(TM) Client VM warning: You have loaded library /usr/local/hadoop/lib/native/libhadoop.so.1.0.0 which might have disabled stack guard. The VM will try to fix the stack guard now.
It's highly recommended that you fix the library with 'execstack -c <libfile>', or link it with '-z noexecstack'.
localhost]
sed: -e 表达式 #1, 字符 6: “s”的未知选项↵
warning:: ssh: Could not resolve hostname warning:: Name or service not known
-c: Unknown cipher type 'cd'
Java: ssh: Could not resolve hostname Java: Name or service not known
it: ssh: Could not resolve hostname it: Name or service not known
It's: ssh: Could not resolve hostname It's: Name or service not known
HotSpot(TM): ssh: Could not resolve hostname HotSpot(TM): Name or service not known
root@localhost's password: link: ssh: Could not resolve hostname link: Name or service not known
<libfile>',: ssh: Could not resolve hostname <libfile>',: Name or service not known
'execstack: ssh: Could not resolve hostname 'execstack: Name or service not known
noexecstack'.: ssh: Could not resolve hostname noexecstack'.: Name or service not known
to: ssh: connect to host to port 22: Connection refused
'-z: ssh: Could not resolve hostname '-z: Name or service not known

修改etc/profile配置文件，添加：

export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib"

export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP_PREFIX}/lib/native

