# Big Data

## Installation HDFS
- Virtuelle Maschine mit Ubuntu 18.04 installieren
- Hadoop 3.1.1 runterladen
- Hadoop nach /opt/hadoop entpacken
- Rechte setzen für User hadoop
- Ordner erstellen /opt/hadoop-data
- $PATH erweitern um /opt/hadoop/bin und /opt/hadoop/sbin

## Konfiguration
### Single-Node - Wordcount Beispiel
- https://hadoop.apache.org/docs/r3.1.1/hadoop-project-dist/hadoop-common/SingleCluster.html

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
```bash
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

#### etc/hadoop/yarn-env.sh
Bugfix für JDK 9+
```bash
export YARN_RESOURCEMANAGER_OPTS="--add-modules java.activation"
export YARN_NODEMANAGER_OPTS="--add-modules java.activation"
```
