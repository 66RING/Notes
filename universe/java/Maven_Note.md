---
title: Maven 基本操作
date: 2019-9-15
tags: maven, java
---

### 基本操作
each project should have a pom.xml that is the setting of maven.  
Add dependency.
``` xml
<dependencies>
		<dependency>
			<groupId>groupId</groupId>
			<artifactId>artifactId</artifactId>
			<version>version</version>
		</dependency>
</dependencies>
```

This block of XML declares a list of dependencies for the project. Specifically, it declares a single dependency for the Joda Time library. Within the &lt;dependency&gt; element, the dependency coordinates are defined by three sub-elements:
- **&lt;groupId&gt;**-The group or organization that the dependency belongs to.
- **&lt;artifactId&gt;**- The library that is required.
- **&lt;version&gt;**-The specific version of the library that is required.

Creat a library package (such as JAR file),and install the library in the local Maven dependency repository.  
When it finished,you should find the file in target/classes  directory.

``` java
mvn compile
```
Package the code up in a JAR(setting in pom.xml) within the target directory.
The name of the JAR file will be baseed on the project's &lt;artifactId&gt; and &lt;version&gt;.
``` jave 
mvn package
```

To execute the JAR file run:
``` jave
java -jar target/gs-maven-0.1.0.jar
```

Maven also maintains a repository of dependencies on your local machine (usually in a *.m2/repository* directory in your home directory) for quick access to project dependencies. If you’d like to install your project’s JAR file to that local repository, then you should invoke the *install* goal:
``` java
mvn install
```
The install goal will compile, test, and package your project’s code and then copy it into the local dependency repository, ready for another project to reference it as a dependency.

Maven uses a plugin called "surefire" to run unit tests. The default configuration of this plugin compiles and runs all classes in src/test/java with a name matching \*Test. You can run the tests on the command line like this.
``` java
mvn test
```
or just use mvn install step as we already showed above (there is a lifecycle definition where "test" is included as a stage in "install").






