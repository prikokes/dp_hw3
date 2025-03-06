# ДЗ 2

Выполнил: Иванов Георгий Ярославович \
Группа: БПИ223

После выполнения предыдущих домашних заданий у нас должна уже быть среда с развернутым hadoop и yarn.

Сначала переключаемся на tmpl-nn и устанавливаем postgres командой. 

```
sudo apt install posgtresql
```

Затем переключаемся в пользователя postgres и в консоли postgresql создаем базу данных, пользователя PostgreSQL и выдаем этому пользователю  
все права на эта базу данных, а так же делаем его владельцем базы данных. 

```
psql
```

```
CREATE DATABASE metastore;
CREATE USER hive WITH PASSWORD 'hiveMegaPass';
GRANT ALL PRIVILEGES ON DATABASE "metastore" TO hive;
ALTER DATABASE metastore OWNER TO hive;
```

Далее в файле /etc/postgresql/16/main/postgresql.conf указываем postgresql слушать tmpl-nn (потому что мы развернули postgresql на этом узле).
```
sudo vim /etc/postgresql/16/main/postgresql.conf
```

Раскомментируем эту строчку: 
```
listen_addresses = '*' 
```

Далее в файле /etc/postgresql/16/main/pg_hba.conf
```
sudo vim /etc/postgresql/16/main/pg_hba.conf
```

Указываем, как можно можно подключиться к базе данных на tmpl-nn: 
``` 
host    metastore    hive    192.168.1.1/32    password
host    metastore    hive    192.168.1.18/32    password
```

Далее переходим на tmpl-jn и устанавливаем postgresql-client-16 
``` 
sudo apt install posgtresql-client-16
```

Затем мы переходим в пользователя hadoop 
``` 
sudo -i -u hadoop
```

Затем скачиваем hive и драйвер к postgresql
``` 
wget https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

Создаем файл hive-site.xml 
``` 
vim ../conf/hive-site.xml
```

Прописываем в нём конфиг для hive.
``` 
<configuration>
    <property>
        <name>hive.server2.authentication</name>
        <value>NONE</value>
    </property>
    
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
    
    <property>
        <name>hive.server2.thrift.port</name>
        <value>5433</value>
        <description>TCP port number to listen on, default 10000</description>
    </property>
    
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://tmpl-nn:5432/metastore</value>
    </property>
    
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
    </property>
    
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
    </property>
    
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hiveMegaPass</value>
    </property>
</configuration>
```

Теперь открываем файл 
``` 
vim ~/.profile
```

Добавляем пути: 
``` 
export HIVE_HOME=/home/hadoop/apache-hive-4.0.0-alpha-2-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
```

Применяем изменения 
``` 
source ~/.profile
```

Создаем директорию и 
``` 
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
```

Затем инициализируем схему
``` 
bin/schematool -dbType postgres -initSchema
```

Затем запускаем Hive 
``` 
hive --hiveconf hive.server2.enable.doAs=false --hiveconf hive.security.authorization.enabled=false --service hiveserver2 1>> /tmp/hs2.log 2>> /tmp/hs2.log &
```

Запускаем консоль beeline:
``` 
beeline -u jdbc:hive2://team-4-jn:5433
```