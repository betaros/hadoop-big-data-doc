# Big Data

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Big Data](#big-data)
	- [Aufbau des Clusters](#aufbau-des-clusters)
	- [Hardware](#hardware)
	- [Einrichtung virtueller Maschinen](#einrichtung-virtueller-maschinen)
	- [Netzwerkeinstellungen](#netzwerkeinstellungen)
		- [IPv6 ausschalten](#ipv6-ausschalten)
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
	- [Arbeitsspeicher](#arbeitsspeicher)
		- [etc/hadoop/yarn-site.xml](#etchadoopyarn-sitexml)
		- [etc/hadoop/mapred-site.xml](#etchadoopmapred-sitexml)
	- [Bedienung](#bedienung)
		- [Starten](#starten)
		- [Stoppen](#stoppen)
		- [Prozesse](#prozesse)
	- [HIPI](#hipi)
		- [Installation](#installation)
		- [Updates](#updates)
		- [Probleme](#probleme)
	- [Quellen](#quellen)

<!-- /TOC -->

## Aufbau des Clusters


## Hardware
Das Cluster besteht aus drei Nodes welche in drei eigenen virtuellen Maschinen laufen. Diese teilen sich auf in ein Master Node und zwei Slave Nodes. Das Hostsystem hat folgende Systemeigenschaften:
- Intel i7-7700 4-core 3.60 GHz
- 32 GB DDR4 RAM
- Windows 10 Pro
- Virtualbox 5.2

Die virtuellen Maschinen verfügen über die folgenden Resourcen:
- 1 vCPU
- 4 GB vRAM
- 15 GB Festplatte
- Ubuntu 18.04.1

## Einrichtung virtueller Maschinen
Für die Erstellung eines virtuellen Clusters wurde VirtualBox 5.2 von der Firma Oracle verwendet. Zusätzlich wurden das Erweiterungspaket installiert. Man findet beides unter

> [www.virtualbox.org](http://www.virtualbox.de)

Nachdem VirtualBox installiert wurde, wurde ein Betriebssystem für die virtuellen Maschinen heruntergeladen. Dabei haben wir uns für die aktuelle Version von [Ubuntu 18.04.1](http://releases.ubuntu.com/18.04/) entschieden. Um möglichst wenig Resourcen zu verwenden wurde ein Minimal-Image heruntergeladen und das Betriebssystem über Konsole installiert.

> [Minimal Image Ubuntu 18.04.1](http://archive.ubuntu.com/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/)

Zuerst wurde eine einzelne VM eingerichtet. Diese wurde wie im Kapitel [Single-Node](#single-node) beschrieben eingerichtet. Die eingerichtete VM ist der Masternode. Nach der Einrichtung wurde die VM zwei Mal geklont, welche dann als Slaves eingerichtet. Das System wurde dann von Single-Node auf Multi-Node wie im Kapitel [Cluster](#cluster) beschrieben konfiguriert.

## Netzwerkeinstellungen
Im Hadoop Cluster verfügen die einzelnen VMs über eigene statische IPs:

Maschine | IP
---------|-----------------
master   | 192.168.178.100
slave1   | 192.168.178.101
slave2   | 192.168.178.102

Die IP Adressen wurden über den Router festgelegt. Dabei wurde eine Fritzbox 7362 SL verwendet.

### IPv6 ausschalten
Um Hadoop nutzen zu können muss IPv6 deaktiviert werden. Dies hat unter Ubuntu 18.04 einen Bug, den man berücksichtigen muss.

1. /etc/sysctl.conf Zeilen zum Ende hinzufügen
```bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

2. Konfiguration neu laden
```bash
sudo sysctl -p
```

3. Bug beheben, dass die Konfiguration nach dem Neustart noch vorhanden ist. Dazu muss die Datei /etc/rc.local erstellt werden, mit folgenden Inhalt:
```bash
 #!/bin/bash
 # /etc/rc.local
 # Load kernel variables from /etc/sysctl.d
/etc/init.d/procps restart
exit 0
```

4. Rechte für Datei nutzen
```bash
sudo chmod 755 /etc/rc.local
```

## Installation HDFS
Das Hadoop Distributed File System (HDFS) ist ein verteiltes Dateisystem, welches auf normalen Rechnern installiert werden kann. Es steht für Anwender frei zur Verfügung.

Viele Teile von Hadoop wurden in Java geschrieben. Aufgrund dessen müssen auf allen Nodes folgende Programme installiert werden. Dabei kann, wie bereits in [Einrichtung virtueller Maschinen](#einrichtung-virtueller-maschinen) beschrieben, das folgende auf dem Master Node angewandt werden und die virtuelle Maschine dann geklont werden.

Zuerst muss das OpenJDK installiert werden. Dabei wurde auf das Default-JDK unter Ubuntu zurückgegriffen, welches das OpenJDK-11 zur Zeit der Einrichtung ist.
```bash
apt install default-jdk
```

Nachdem Java installiert wurde kann Hadoop 2.9.1 heruntergeladen und entpackt werden.
```bash
wget -c  https://archive.apache.org/dist/hadoop/core/hadoop-2.9.1/hadoop-2.9.1.tar.gz
tar -xvf hadoop-2.9.1.tar.gz
```

Das entpackte Paket kann nun in einen Ordner verschoben werden, in dem Hadoop installiert werden soll. Dabei haben wir uns für den Ordner /opt im Linux Dateisystem entschieden, da dieser Ordner für optionale Software verwendet wird.
```bash
sudo mv hadoop /opt/hadoop
```

Auf dem System müssen dann Pfade erstellt und angegeben werden, welche für den NameNode und DataNode benötigt werden. Dies wurde wie folgt erstellt:
```bash
sudo mkdir /opt/hadoop-data
sudo mkdir /opt/hadoop-data/name
sudo mkdir /opt/hadoop-data/data
```

Um Hadoop auch ohne Rootrechten nutzen zu können wurden die Eigentümer für die neuen Ordner auf für den User hadoop auf *hadoop* gesetzt:
```bash
sudo chown -R hadoop:hadoop /opt/hadoop
sudo chown -R hadoop:hadoop /opt/hadoop-data
```

Da der Aufruf von Hadoop über den Pfad sehr umständlich ist wurde zur Vereinfachung die Pfadvariable erweitert um die Pfade zu den ausführbaren Dateien in Hadoop. Diese befinden sich unter ```/opt/hadoop/bin``` und ```/opt/hadoop/sbin```.
```bash
echo "export PATH=$PATH:/opt/hadoop/bin:/opt/hadoop/sbin" >> ~/.bashrc
```

## Konfiguration
In den folgenden Kapiteln wird zuerst die Einrichtung eines einzelnen Nodes beschrieben. Dies wird dann erweitert um weitere Nodes, welche dann ein Cluster bilden.

### Single-Node
Die Einrichtung eines einzelnen Nodes wird auf der Apache Webseite zu Hadoop beschrieben. Wir haben uns bei der Einrichtung an dieser orientiert:  [Apache Hadoop](https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html).

#### SSH
Damit die Nodes untereinander kommunizieren können, ist es notwendig einen SSH Schlüssel auf jeden Node zu erstellen. Dieser wird dann auf jeden Node installiert, sodass alle Nodes untereinander mit Hadoop kommunizieren können.
```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
ssh-copy-id 'Name des Ziels'
```

Bei der Einrichtung eines einzelnen Nodes muss das Ziel der Master Node sein: *master*.

#### etc/hadoop/core-site.xml
In der Datei ```core-site.xml``` wird angegeben wo sich die NameNodes im Cluster befinden. Zudem werden Grundfunktionalitäten wie HDFS und MapReduce dort definiert. Hier wird der Zugriffspunkt aus dem Netzwerk definiert. In diesem Fall *localhost:9000*.
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

> Gegebenenfalls muss vor dem Ausführen von ```hdfs namenode -format``` der Hadoop-Data Ordner geleert werden (master und slaves)

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
        <value>master:8031</value>
    </property>

    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>master:8030</value>
    </property>

    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>master:8032</value>
    </property>
</configuration>
```

#### etc/hadoop/slaves
Damit der Hadoop Master Node mit den Slave Nodes kommunizieren kann, müssen die Hostnames der Slaves in der Datei *slaves* hinzugefügt werden:
```bash
slave1
slave2
```

## Arbeitsspeicher
Hadoop verwendet standardmäßig in für die Nodes 8 GB RAM. Da die eingerichteten Nodes jedoch nur über 4 GB vRAM verfügen, muss dies noch konfiguriert werden. Die folgende Tabelle zeigt dabei die eingestellten Werte.

Eigenschaften                        | Wert
-------------------------------------|------
yarn.nodemanager.resource.memory-mb  | 3072
yarn.scheduler.maximum-allocation-mb | 3072
yarn.scheduler.minimum-allocation-mb | 256
yarn.app.mapreduce.am.resource.mb    | 1024
mapreduce.map.memory.mb              | 512
mapreduce.reduce.memory.mb           | 512

### etc/hadoop/yarn-site.xml
```bash
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>3072</value>
</property>

<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>3072</value>
</property>

<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>256</value>
</property>

<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
</property>
```

### etc/hadoop/mapred-site.xml
```bash
<property>
        <name>yarn.app.mapreduce.am.resource.mb</name>
        <value>1024</value>
</property>

<property>
        <name>mapreduce.map.memory.mb</name>
        <value>512</value>
</property>

<property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>512</value>
</property>
```


## Bedienung

### Starten
- DFS Starten
```bash
start-dfs.sh
```

- YARN Starten
```bash
start-yarn.sh
```

### Stoppen
- YARN Stoppen
```bash
stop-yarn.sh
```

- DFS Stoppen
```bash
stop-dfs.sh
```

### Prozesse
Prozesse anzeigen
```bash
jps
```

Nach dem Start von DFS und YARN sollten die folgenden die folgenden Prozesse auf dem *Master* angezeigt werden:
```bash
NameNode
Jps
ResourceManager
SecondaryNameNode
NodeManager
```

Auf den *Slaves* sollten jeweils die folgenden Prozesse gestartet sein:
```bash
Jps
SecondaryNameNode
DataNode
```

Die Prozesse haben die folgende Bedeutung:
- NameNode

	Der NameNode kontrolliert und verwaltet alle Dateien, die im HDFS abgespeichert sind. Dabei beinhaltet es nur Metadaten von den Dateien. Es läuft nur auf dem Master Node

- jps

	Mit JPS werden die laufenden Prozesse im Hadoop Cluster angezeigt.

- ResourceManager

	Der ResourceManager verteilt die vorhanden Resourcen an die unterschiedlichen Nodes und sorgt damit für eine optimale Auslastung des Clusters.

- SecondaryNameNode

	Der SecondaryNameNode ist ein Hilfsprozess für den NameNode, welcher den Zugang zum HDFS auf den einzelnen Nodes darstellt.

- DataNode

	Die im Cluster vorhanden Daten werden im DataNode gespeichert. In unserer Konfiguration werden die DataNodes nur auf den Slaves ausgeführt.

- NodeManager

	Der NodeManager sorgt auf jedem Node dafür, dass die Auslastung des Nodes erfasst und an den ResourceManager weitergeleitet wird.

## HIPI
HIPI ist eine Bildverarbeitungsbibliothek für Hadoop, welche an der University of Virginia, USA entwickelt wurde. Für die Bildverarbeitung wird MapReduce verwendet. Zudem bietet es die Möglichkeit große Datenmengen zu verwalten und mit OpenCV auszuwerten.

### Installation
Bei der Installation haben wir uns auf [die offizielle Dokumentation](http://hipi.cs.virginia.edu/gettingstarted.html) der Entwickler bezogen.
- Gradle installieren
```bash
sudo apt install gradle
```

- HIPI clonen
```bash
git clone https://github.com/uvagfx/hipi.git
```

- tools/build.gradle anpassen
```bash
jar {
    manifest {
      attributes("Class-Path" : configurations.runtime.collect { it.getAbsolutePath() }.join(' '));
  //    attributes("Class-Path" : configurations.runtime.collect { it.toURI() }.join(' '));
    }
```

- Gradle nutzen
```bash
cd hipi
gradle
```

### Updates
Möchte man HIPI updaten, kann man dies mit Git machen. Jedoch wurden seit über 3 Jahren keine Änderungen am Quellcode durchgeführt.
```bash
git pull origin release
```

## Probleme
Nach der Installation von Hipi haben wir versucht das Beispielprogramm auszuführen.
- ClassNotFoundException
> https://stackoverflow.com/questions/53298672/hadoop-hipi-hibimport-noclassdeffounderror/53409716#53409716


## Quellen
- https://www.linode.com/docs/databases/hadoop/how-to-install-and-set-up-hadoop-cluster/
- https://hadoop.apache.org/docs/r2.9.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html
- https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/ClusterSetup.html
- https://www.admintome.com/blog/disable-ipv6-on-ubuntu-18-04/
