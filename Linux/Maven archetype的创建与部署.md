# Maven archetype的创建与部署

本文档介绍了根据Java多模块maven项目创建archetype项目并将其部署到私服的过程。

生成archetype后，可以通过 `mvn archetype:generate` 命令快速构建项目。

## 1. 生成archetype项目

通过maven-archetype-plugin能够生成archetype项目，在 `pom.xml` 文件中添加以下插件：

```xml
<!-- 配置maven-archetype -->
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-archetype-plugin</artifactId>
	<version>3.0.0</version>
</plugin>
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-resources-plugin</artifactId>
	<version>3.0.0</version>
	<configuration>
		<encoding>UTF-8</encoding>
	</configuration>
</plugin>
```

在 `pom.xml` 文件目录下，执行以下命令生成archetype项目：

```shell
> mvn archetype:create-from-project
```

执行成功以后，在项目的target文件夹中会出现以下目录与文件：

---generated-sources

   ---archetype

​      ---project files

​         ---src

​         ---target

​         ---pom.xml

进入到archetype目录：

```shell
> cd ./target/generated-sources/archetype/
```

修改其下 `pom.xml` ，在其 `<project>` 中配置私服路径，例如：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.sunveee</groupId>
  <artifactId>ssm-archetype</artifactId>
  <version>1.0.RELEASE</version>
  <packaging>maven-archetype</packaging>

  <name>ssm-archetype</name>

  <build>
    <extensions>
      <extension>
        <groupId>org.apache.maven.archetype</groupId>
        <artifactId>archetype-packaging</artifactId>
        <version>3.0.0</version>
      </extension>
    </extensions>

    <pluginManagement>
      <plugins>
        <plugin>
          <artifactId>maven-archetype-plugin</artifactId>
          <version>3.0.0</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>

  <distributionManagement>
    <repository>
      <id>nexus-releases</id> 
      <name>Nexus Release Repository</name> 
      <url>http://192.168.51.51:8081/nexus/content/repositories/releases/</url> 
    </repository> 
    <snapshotRepository> 
      <id>nexus-snapshots</id> 
      <name>Nexus Snapshot Repository</name> 
      <url>http://192.168.51.51:8081/nexus/content/repositories/snapshots/</url> 
    </snapshotRepository>
  </distributionManagement>

</project>
```

保存后在当前目录下执行：

```shell
> mvn install
```

执行成功以后，在当前目录下执行crawl命令，生成 `archetype-catalog.xml` ：

```shell
> mvn archetype:crawl
```

执行成功后，在本地仓库的根目录生成了 `archetype-catalog.xml` ，至此archetype项目生成完成。

## 2. 部署archetype项目到私服

检查本地maven配置文件的私服配置，在 `<servers>` 中需要配置私服相应的 `<server>` ，例如要发布到releases与snapshots需要具有如下配置：

```xml
<server>
  <id>nexus-releases</id>
  <username>admin</username>
  <password>123456</password>
</server>
<server>
  <id>nexus-snapshots</id>
  <username>admin</username>
  <password>123456</password>
</server>
```

注意：这里的server.id需要与archetype/ `pom.xml`配置中的repository.id保持一致。

配置完成后，在当前目录下执行以下命令将项目发布到私服：

```shell
> mvn deploy
```

提示BUILD SUCCESS即发布成功。

