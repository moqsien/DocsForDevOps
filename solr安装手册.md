# solr安装手册

## 环境准备及安装

软件下载地址

- jdk-1.8[下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- apache-tomcat-8.5.24[下载地址](http://mirror.bit.edu.cn/apache/tomcat/tomcat-8/v8.0.48/bin/apache-tomcat-8.0.48.tar.gz)
- solr-6.6.2[下载地址](http://mirror.bit.edu.cn/apache/lucene/solr/6.6.2/solr-6.6.2.tgz)
- ikanalyzer-6.6.1[下载地址](https://github.com/zxiaofan/ik-analyzer-solr6/releases/download/6.6.1/ikanalyzer-6.6.1.jar)和ikanalyzer-6.6.0[下载地址](https://github.com/zxiaofan/ik-analyzer-solr6/releases/download/6.6.0/ikanalyzer-6.6.0.jar)
- pinyin4j_IKconfig[下载地址](https://github.com/zxiaofan/ik-analyzer-solr6/releases/download/6.6.0/pinyin4j_IKconfig.zip)
- mysql-connector-java-5.1.45[下载地址](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.45.tar.gz)

### 环境安装（软件要求）
  **软件要求**
| 软件名称                 | 软件版本   | 其他说明 |
| -------------------- | ------ | ---- |
| rng-tools            | 2.13   | 熵服务  |
| lsof                 | 4.82   |      |
| jdk                  | 1.8    |      |
| apache-tomcat        | 8.5.24 |      |
| solr                 | 6.6.2  |      |
| ikanalyzer           | 6.6.0  |      |
| mysql-connector-java | 5.1.45 |      |

### 安装jdk

1. 安装rpm

```shell
rpm -ivh jdk-8u151-linux-x64.rpm
```

2. 配置环境变量

* 编辑/etc/profile文件

```shell
vim /etc/profile
```

*  在文件末尾加入代码

```shell
JAVA_HOME=/usr/java/jdk1.8.0_151
JRE_HOME=$JAVA_HOME/jre
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
export PATH JAVA_HOME CLASSPATH
```

* 生效配置，并检验结果

```shell
source /etc/profile
java -version
```

### 安装solr

* 安装前检测rng-tools及lsof是否正确安装

```shell
rpm -qa |grep rng-tools
rpm -qa |grep lsof
```

若未安装，设备能够联网的情况下，可使用如下命令安装，否则自行下载安装包安装

```shell
yum install rng-tools
yum install lsof
```

配置rng-tools

```shell
echo 'EXTRAOPTIONS="--rng-device /dev/urandom"' >/etc/sysconfig/rngd
service rngd start
chkconfig rngd on
```

* 在非root用户下，创建一个solr目录，并将apache-tomcat及solr压缩包拷贝进去，并解压缩

```shell
tar -zxvf apache-tomcat-8.5.24.tar.gz
tar -zxvf solr-6.6.2.tgz
```

### 配置solr

* 拷贝dataimporthandler的jar包

```shell
cp dist/solr-dataimporthandler-* server/solr-webapp/webapp/WEB-INF/lib/
```

* 拷贝mysql-connector-java

```shell
tar -zxvf mysql-connector-java-5.1.45.tar.gz
cp mysql-connector-java-5.1.45/mysql-connector-java-5.1.45-bin.jar solr-6.6.2/server/solr-webapp/webapp/WEB-INF/lib/
```

* 拷贝ikanalyzer

```shell
cp ikanalyzer-6.6.* solr-6.6.2/server/solr-webapp/webapp/WEB-INF/lib/
```

* 配置pinyin4j

**复制文件**

```shell
unzip pinyin4j_IKconfig.zip -d pinyin4j
cp pinyin4j/pinyin*.jar solr-6.6.2/server/solr-webapp/webapp/WEB-INF/lib/
mkdir solr-6.6.2/server/solr-webapp/webapp/WEB-INF/classes
cp pinyin4j/ext.dic solr-6.6.2/server/solr-webapp/webapp/WEB-INF/classes/
cp pinyin4j/IKAnalyzer.cfg.xml solr-6.6.2/server/solr-webapp/webapp/WEB-INF/classes/
cp pinyin4j/stopword.dic solr-6.6.2/server/solr-webapp/webapp/WEB-INF/classes/
```

**修改配置**solr-6.6.2/server/solr-webapp/webapp/WEB-INF/classes/IKAnalyzer.cfg.xml
加入新的内容

```shell
<!--词典动态更新时间间隔[首次延时,时间间隔]（格式：正整数，单位：分钟）-->
<entry key="dic_updateMin">1,1</entry>

<!--禁用内置主词典main2012.dic（默认false）-->
<!--<entry key="dicInner_disable">true</entry> -->
```

* 修改时区，修改bin/solr.in.sh中的SOLR_TIMEZONE="UTC+8"

## 创建core并配置

1. 进入solr-6.6.2目录，执行创建名称为goods的core（需要先启动）

```shell
bin/solr create -c goods
bin/solr create_core -c goods -p 8983
```

2. 修改配置

分别拷贝solrconfig.xml,data-config.xml,managed-schema到server/solr/goods/conf下

3. 详细修改

* solrconfig.xml添加内容（在</config>之前）

```shell
<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
        <str name="config">data-config.xml</str>
    </lst>
</requestHandler>
```

* data-config.xml添加内容

```shell
<?xml version="1.0" encoding="UTF-8"?>
<dataConfig>
    <dataSource 
        name="dbSource"
        type="JdbcDataSource"  
        driver="com.mysql.jdbc.Driver"  
        url="jdbc:mysql://db.hz.backustech.com:3311/mintai"
        batchSize="-1"
        user="dev"  
        password="xxxx"
        readOnly="true"
        autoCommit="true"
        netTimeoutForStreamingResults="0"
        />
    <document>
        <entity name="goods" dataSource="dbSource" onError="skip" pk="id" query="SELECT g.id,g.category_id,g.sales_sum,g.title,g.pics,g.price,g.class,g.sub_title,g.brand,g.dimension,g.update_time FROM lab_model_goods g LEFT JOIN lab_model_store s ON g.`store_quote-content_id` = s.id WHERE g.delete_time = 0 AND g.goods_status = 2 AND g.category_id IN (498) AND s.store_status=1"
            deltaImportQuery="select id,category_id,title,sub_title,pics,class,price,brand,sales_sum,dimension,update_time from lab_model_goods where id = '${dih.delta.id}'"
            deltaQuery="select g.id from lab_model_goods g LEFT JOIN lab_model_store s ON g.`store_quote-content_id` = s.id where g.update_time > unix_timestamp('${dataimporter.last_index_time}') AND g.delete_time=0 AND g.goods_status=2 AND g.category_id in (498) AND s.store_status=1">
            <field column="id" name="id" />
            <field column="category_id" name="category_id" />
            <field column="update_time" name="update_time" />
            <field column="title" name="title" />
            <field column="pics" name="pics" />
            <field column="price" name="price" />
            <field column="class" name="class" />
            <field column="sub_title" name="sub_title" />
            <field column="brand" name="brand" />
            <field column="dimension" name="dimension" />
            <field column="sales_sum" name="sales_sum" />
        </entity>
    </document>
</dataConfig>
```

* managed-schema添加内容（在<field name="id" ... />之后）

```shell
	<field name="category_id" type="string" indexed="true" stored="true"/>
    <field name="pics" type="string" indexed="true" stored="true"/>
    <field name="price" type="float" indexed="true" stored="true"/>
    <field name="class" type="string" indexed="true" stored="true"/>
    <field name="sub_title" type="text_ik" indexed="true" required="true" stored="true"/>
    <field name="brand" type="string" indexed="true" stored="true"/>
    <field name="dimension" type="string" indexed="true" stored="true"/>
    <field name="sales_sum" type="int" indexed="true" stored="true"/>
    <fieldType name="text_pinyin" class="solr.TextField" positionIncrementGap="0">
        <analyzer type="index">
            <tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory"/>
            <filter class="solr.SynonymGraphFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true" />
            <filter class="com.shentong.search.analyzers.PinyinTransformTokenFilterFactory" minTermLenght="2" />
            <filter class="com.shentong.search.analyzers.PinyinNGramTokenFilterFactory" minGram="1" maxGram="20" />
        </analyzer>
        <analyzer type="query">
            <tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory"/>
            <filter class="solr.SynonymGraphFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true" />
            <filter class="solr.LowerCaseFilterFactory" />
        </analyzer>
    </fieldType>
    <fieldType name="text_ik" class="solr.TextField">
        <analyzer type="index" useSmart="false" isMaxWordLength="false" >
            <tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory"/>
            <filter class="solr.SynonymGraphFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
        </analyzer>
        <analyzer type="query" useSmart="true" isMaxWordLength="true" >
            <tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory"/>
            <filter class="solr.SynonymGraphFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
        </analyzer>
    </fieldType>
    <field name="title_ik" type="text_ik" indexed="true" required="true" stored="true"/>
    <field name="title" type="text_general" indexed="true" required="true" stored="true" multiValued="false"/>
    <copyField source="title" dest="title_ik"/>
    <field name="update_time" type="int" indexed="true" stored="true"/>
    <field name="pinyin" type="text_pinyin" indexed="true" stored="true"/>
    <copyField source="title" dest="pinyin"/>
```

## 启动solr并导入数据

### 常用启停命令

* 进入目录solr-6.6.2，执行如下命令启动

```shell
bin/solr start
```

* 停止solr命令

```shell
bin/solr stop -all
```

* 重启solr

```shell
bin/solr stop -all; bin/solr start
```

### 导入数据

* 浏览器中访问http://IP:8983/, 查看CoreAdmin中是否存在创建的core:goods

* Core Selector选择新建的core（如goods），选择Dataimport，Command选择full-import，Start, Rows选择合理值，点击Excute执行数据导入

### 验证数据是否导入

* 接上一步中选择Query，直接点击Execute Query查看结果

### 验证分词是否可用

* 接上一步中选择Analysis，输入值，类型选择text_ik，查看分词结果（需要分词的数据类型在managed-schema中field的type为text_ik类型。

## 添加批处理任务

* 将apache-solr-dataimporthandler-.jar放到solr-6.6.2/server/solr-webapp/webapp/WEB-INF/lib/
* 在solr-6.6.2/server/solr-webapp/webapp/WEB-INF/web.xml中的</web-app>之前加入下面代码

```shell
<listener>
  <listener-class>org.apache.solr.handler.dataimport.scheduler.ApplicationListener</listener-class>
</listener>
```

* 在solr/conf中新建dataimport.properties，文件夹不存在时新建

```shell
#################################################
#                                               #
#       dataimport scheduler properties         #
#                                               #
#################################################

#  to sync or not to sync
#  1 - active; anything else - inactive
syncEnabled=1

#  which cores to schedule
#  in a multi-core environment you can decide which cores you want syncronized
#  leave empty or comment it out if using single-core deployment
syncCores=goods,goods-test

#  solr server name or IP address
#  [defaults to localhost if empty]
server=localhost

#  solr server port
#  [defaults to 80 if empty]
port=8983

#  application name/context
#  [defaults to current ServletContextListener's context (app) name]
webapp=solr

#  URL params [mandatory]
#  remainder of URL
#增量
params=/dataimport?command=delta-import&clean=false&commit=true&optimize=false&wt=json&indent=true&entity=goods&verbose=false&debug=false

#  schedule interval
#  number of minutes between two runs
#  [defaults to 30 if empty]
interval=20

#  重做索引的时间间隔，单位分钟，默认7200，即1天;
#  为空,为0,或者注释掉:表示永不重做索引
reBuildIndexInterval=7200

#  重做索引的参数
reBuildIndexParams=/dataimport?command=full-import&clean=true&commit=true&optimize=true&wt=json&indent=true&entity=goods&verbose=false&debug=false

#  重做索引时间间隔的计时开始时间，第一次真正执行的时间=reBuildIndexBeginTime+reBuildIndexInterval*60*1000；
#  两种格式：2012-04-11 03:10:00 或者  03:10:00，后一种会自动补全日期部分为服务启动时的日期
reBuildIndexBeginTime=09:00:00
```

* 重启solr