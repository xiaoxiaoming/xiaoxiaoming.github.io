---
layout: post
title: "用Maven制作能够读取外部配置文件的可执行java工程"
date: 2013-10-20 16:22
comments: true
categories: 
---


需要的最终可执行工程目录结构如下：
<pre>
mavnconf
├─bin
├─conf
└─lib
</pre>

**步骤一**：创建一个maven工程，其名为mavnconf，并在main目录下，增加resources、scripts和assembly目录。
<pre>
mavnconf
├─src
│  ├─main
│  │  ├─java 
│  │  ├─assembly
│  │  ├─resources
│  │  └─scripts
</pre>
**步骤二**：添加启动脚本。

在scripts目录创建run.sh文件，内容如下：
<pre>
java -Djava.ext.dirs="../lib" -server -jar "../lib/mavnconf-1.0-SNAPSHOT.jar"</pre>
在scripts目录创建run.bat文件，内容如下
<pre>
java -Djava.ext.dirs="..\lib" -server -jar ..\lib\mavnconf-1.0-SNAPSHOT.jar
</pre>

**步骤三**：给pom.xml添加maven-jar-plugin，内容如下：
```xml
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>2.3.2</version>
            <configuration>
                <archive>
                    <manifest>
                        <addClasspath>true</addClasspath>
                        <classpathPrefix>lib/</classpathPrefix>
                        <mainClass>mavnconf.App</mainClass>
                    </manifest>
                    <manifestEntries>
                        <Class-Path>./../conf/</Class-Path>
                    </manifestEntries>
                </archive>
            </configuration>
        </plugin>
```
关键点说明:

* mainfiest.addClasspath：是否添加类路径
* mainfiest.classpathPrefix：库路径的前缀,通过配置**lib/**，jar包可以读app/lib/目录下的类库。
* mainfiest.mainClass:可执行jar包的入口类
* mainfiest.manifestEntries.Class-Path:自定义一个类路径，在本工程中设为./../conf/。.代表jar所在的目录，即为lib目录。./../代表jar包所在目录的父目录，即为app。./../conf/代表外部配置文件所在的目录,与lib目录同级。

**步骤四**，添加 maven-assembly-plugin，使得maven组装出需要的工程结构:

```xml
    		<plugin>
				<artifactId>maven-assembly-plugin</artifactId>
				<version>2.3</version>
				<configuration>
					<descriptors>
						<descriptor>
							src/main/assembly/assembly.xml
						</descriptor>
					</descriptors>
                    <finalName>mavnconf</finalName>
				</configuration>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>assembly</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
```
这里最关键的引入assembly.xml文件，在它的帮助下，可以打包需要的目录结构
assembly.xml的内容如下

```xml
	<id>release</id>
	<formats>
		<format>tar.gz</format>  <!-- 其他可选格式 gzip/zip/tar.gz/ -->
	</formats>
	<includeBaseDirectory>false</includeBaseDirectory>
```
此配置的含义是：将打包成名为mavnconf-release.tar.gz的压缩包。


```xml
	<dependencySets>
		<dependencySet>
            <!--最终打包至输出包内的lib路径下-->
			<outputDirectory>/lib</outputDirectory>
            <!--将项目本身生成的构件也包含在内-->
			<useProjectArtifact>true</useProjectArtifact>
			<unpack>false</unpack>
			<scope>runtime</scope>
		</dependencySet>
	</dependencySets>
```
此配置的含义是，将 工程文件打包成jar包，并和依赖的类库，复制到lib目录下

关键要点说明：

* dependencySet.outputDirectory为输出目录。

* dependencySet.useProjectArtifact：是否使用本工程代码和资源来构建构建，这里必须为true，否在在lib目录，无法找到mavnconf-1.0-SNAPSHOT构件。

```xml
<fileSets>
		<fileSet>
			<directory>src/main/scripts</directory>
			<outputDirectory>/bin</outputDirectory>
            <fileMode>0755</fileMode>
            <directoryMode>0755</directoryMode>

		</fileSet>
        <fileSet>
            <directory>target/classes/config/properties/</directory>
            <outputDirectory>/conf</outputDirectory>
            <includes>
                <include>**.properties</include>
            </includes>
        </fileSet>
    </fileSets>
``` 
第一个fileSet目录是将run.sh文件复制到bin目录

第二个fileSet的目录是将以来的properties文件复制到app/conf目录下

要点说明：

* run.sh是可执行文件，在linux中它必须是可执行的，所以权限为0755


**步骤五:**写个例子来验证效果

在resources目录下，创建文件夹properties,并加入文件app.properties，内容如下
<pre>
app.test="app.test"
</pre>

在src/main/mavnconf目录下创建App.class，内容如下:

```java
public class App 
{
    public static void main( String[] args )throws Exception
    {
        Properties p = new Properties();
        p.load(getInputStream());
        System.out.println(p.getProperty("app.test"));
    }

    public static InputStream getInputStream(){
      InputStream input= App.class.getClassLoader().getResourceAsStream("app.properties");
        if(input == null){
         input = App.class.getClassLoader().getResourceAsStream("./properties/app.properties");
      };
      return input;
    }
}
```

验证方式如下:

* 使用maven编译工程，在target目录下找到mavenconf-release.tar.gz压缩包,并解压文件。
* win+r，输入cmd回车，弹出shell窗口，进入对应的bin目录，执行run.bat
* 修改conf目录下app.properties内容，再次执行run.bat


#附件，稍等一段时间...