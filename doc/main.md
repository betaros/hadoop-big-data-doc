# Big Data

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Big Data](#big-data)
	- [Hardware](#hardware)
	- [Installation HDFS](#installation-hdfs)
	- [Konfiguration](#konfiguration)
		- [Single-Node](#single-node)
			- [SSH](#ssh)
			- [etc/hadoop/core-site.xml](#etchadoopcore-sitexml)
			- [etc/hadoop/hdfs-site.xml](#etchadoophdfs-sitexml)
			- [etc/hadoop/hadoop-env.sh](#etchadoophadoop-envsh)
			- [etc/hadoop/mapred-site.xml](#etchadoopmapred-sitexml)
			- [etc/hadoop/yarn-env.sh](#etchadoopyarn-envsh)
			- [etc/hadoop/yarn-site.xml](#etchadoopyarn-sitexml)
			- [Namenode einrichten](#namenode-einrichten)
		- [Cluster](#cluster)
			- [/etc/hosts](#etchosts)
			- [etc/hadoop/core-site.xml](#etchadoopcore-sitexml)
			- [etc/hadoop/hdfs-site.xml](#etchadoophdfs-sitexml)
			- [etc/hadoop/yarn-site.xml](#etchadoopyarn-sitexml)
			- [etc/hadoop/slaves](#etchadoopslaves)
		- [Bedienung](#bedienung)
			- [Prozesse](#prozesse)
			- [Starten](#starten)
			- [Stoppen](#stoppen)
	- [Quellen](#quellen)

<!-- /TOC -->

## Hardware
Das Cluster besteht aus drei Nodes welche in drei eigenen virtuellen Maschinen laufen. Diese teilen sich auf in ein Master Node und zwei Slave Nodes. Das Hostsystem hat folgende Systemeigenschaften:
- Intel i7-7700 4-core 3.60 GHz
- 32 GB DDR4 RAM
- Windows 10 Pro

Die virtuellen Maschinen verfügen über die folgenden Resourcen:
- 1 vCPU
- 4 GB vRAM
- 15 GB Festplatte
- Ubuntu 18.04.1

## Installation HDFS
- Virtuelle Maschine mit Ubuntu 18.04 installieren
  - Username: hadoop
- JDK installieren
  - ```apt install default-jdk```
- Hadoop 2.9.1 runterladen und entpacken
  - ```wget -c  https://archive.apache.org/dist/hadoop/core/hadoop-2.9.1/hadoop-2.9.1.tar.gz```
  - ```tar -xvf hadoop-2.9.1.tar.gz```
- Hadoop nach /opt/hadoop entpacken
  - ```sudo mv hadoop /opt/hadoop```
- Ordner erstellen für Namenode und Datanode
  - ```sudo mkdir /opt/hadoop-data```
  - ```sudo mkdir /opt/hadoop-data/name```
  - ```sudo mkdir /opt/hadoop-data/data```
- Rechte setzen für User hadoop
  - ```sudo chown -R hadoop:hadoop /opt/hadoop```
  - ```sudo chown -R hadoop:hadoop /opt/hadoop-data```
- $PATH erweitern um /opt/hadoop/bin und /opt/hadoop/sbin
  - ```echo "export PATH=$PATH:/opt/hadoop/bin:/opt/hadoop/sbin" >> ~/.bashrc```

## Konfiguration
### Single-Node
- https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html

#### SSH
Wenn ```ssh localhost``` fehlschlägt:
```bash
$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ chmod 0600 ~/.ssh/authorized_keys
```

#### etc/hadoop/core-site.xml
```bash
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

#### etc/hadoop/hdfs-site.xml
- Namenode Dateisystem Pfad angeben
- Datanode Dateisystem Pfad angeben
- Replikationen von Blöcken einstellen

```bash
<configuration>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/opt/hadoop-data/name</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/opt/hadoop-data/data</value>
    </property>

    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

#### etc/hadoop/hadoop-env.sh
- Java 11 Pfad anpassen

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/
```

#### etc/hadoop/mapred-site.xml
```bash
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

#### etc/hadoop/yarn-env.sh
- Bugfix für JDK 9+
- siehe Bug [JIRA Hadoop](https://issues.apache.org/jira/browse/HADOOP-14978)

```bash
export YARN_RESOURCEMANAGER_OPTS="--add-modules java.activation"
export YARN_NODEMANAGER_OPTS="--add-modules java.activation"
```

#### etc/hadoop/yarn-site.xml
```bash
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

#### Namenode einrichten
- Namenode Dateisystem formatieren
  - ```hdfs namenode -format```
- DFS starten
  - ```start-dfs.sh```
- User Ordner in DFS einrichten
  - ```hdfs dfs -mkdir /user```
  - ```hdfs dfs -mkdir /user/hadoop```
- Input Dateien kopieren
  - ```hdfs dfs -put etc/hadoop input```

### Cluster
#### /etc/hosts
- Adressen für Master und Slaves angeben

```bash
192.168.178.100 master
192.168.178.101 slave1
192.168.178.102 slave2
```

#### etc/hadoop/core-site.xml
```bash
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
</configuration>
```

#### etc/hadoop/hdfs-site.xml
- Replikationen auf Anzahl der Slaves erhöhen

```bash
<configuration>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/opt/hadoop-data/name</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/opt/hadoop-data/data</value>
    </property>

    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
</configuration>
```

#### etc/hadoop/yarn-site.xml
- Adressen für Slaves anpassen

```bash
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <property>
        <name>yarn.resourcemanager.address</name>
        <value>master:port</value>
    </property>

    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>master:port</value>
    </property>

    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>master:port</value>
    </property>
</configuration>
```

#### etc/hadoop/slaves
```bash
slave1
slave2
```

### Bedienung

#### Prozesse
- Prozesse anzeigen
```bash
jps
```

- Gestartete Prozesse nach dem Start von DFS und YARN
```bash
3479 NameNode
5671 Jps
4297 ResourceManager
3851 SecondaryNameNode
3646 DataNode
4430 NodeManager
```

#### Starten
- DFS Starten
```bash
start-dfs.sh
```

- YARN Starten
```bash
start-yarn.sh
```

#### Stoppen
- YARN Stoppen
```bash
stop-yarn.sh
```

- DFS Stoppen
```bash
stop-dfs.sh
```

## Quellen
- https://www.linode.com/docs/databases/hadoop/how-to-install-and-set-up-hadoop-cluster/
- https://hadoop.apache.org/docs/r2.9.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html
- https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/ClusterSetup.html
