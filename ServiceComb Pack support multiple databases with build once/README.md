# ServiceComb Pack Alpha 编译一次支持多种数据库类型

在有些场合使用什么数据库不是软件发布者所能决定的（例如：xx运营商只让用mysql，xx运营商只让用postgresql，不做传统行业不知道的痛🙈）

alpha默认支持postgresql，如果要使用mysql需要重新使用`mvn clean install -Pmysql'重新编译发布，这有点不方便 :(

我们可以使用支持外部jar的方式重新编译alpha-server，这样我们可以只发布一次就可以支持多种数据类型。

1、打开alpha-server pom.xml

增加layout为ZIP

```
<plugin>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-maven-plugin</artifactId>
<configuration>
  <layout>ZIP</layout>
</configuration>
</plugin>
```

2、重新编译alpha-server

```
mvn clean package -DskipTests
```

3、重新编译后的alpha-server-0.4.0-SNAPSHOT-exec.jar就支持外部jar了

首先在alpha-server-0.4.0-SNAPSHOT-exec.jar同级目录建立一个libs目录

复制mysql-connector-java-8.0.15.jar到这个libs目录

在启动时通过命令行参数指定外部jar目录

```
java -Dloader.path=./libs -jar alpha-server-0.4.0-SNAPSHOT-exec.jar \
--spring.datasource.platform=mysql \
--spring.datasource.dataSourceClassName=com.mysql.jdbc.Driver \
--spring.datasource.url='jdbc:mysql://0.0.0.0:3306/saga?serverTimezone=GMT%2b8&useSSL=false' \
--spring.datasource.username=saga-user \
--spring.datasource.password=saga-password \
--spring.profiles.active=prd
```

**注意：**alpha默认提供了mysql、postgresql两种建库脚本，所以你使用的是其他数据库还需要在外部增加自动建库的sql，例如oracle，在alpha-server-0.4.0-SNAPSHOT-exec.jar同级目录下建立schema-oracle.sql文件并编写建表脚本，然后在按照上边的方法启动就可以了

**注意：**alpha默认并没有进行其他数据库版本支持的测试，你需要先自己测试一下，因为使用的是JPA我想问题不大，祝你好运！

