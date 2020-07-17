---
title: Spring Boot Note
date: 2019-9-14
tags: spring, spring boot, java
---

Project that build with Maven
## Quick buile a Spring Boot project 
Use your IDE and new a project with Spring Initializer
     
---

## Some basic operations
### Creat an excutable jar

To create an excutable jar we neen to add the `spring-boot-maven-plugin` to our pom.xml.  
  
Insert the following lines just below the `dependencis` section
``` xml
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>
```  
  
and than run
```
mvn package
```    
If you look in the `target` directory,you should see a JAR file.

To run the JAR use the `java -jar`command.

### Run with Maven Plugin
```
mvn spring-boot:run
```

---

## About POM file
###1. &lt;parent&gt;
``` xml
<!-- example -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.8.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```
Which is like the arbitration center of Spring Boot  

###2. &lt;dependencis&gt;
This part is to import dependencis
#### starter
Starters are a set of convenient dependency descriptors that you can include in your application.  
<https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/html/using-boot-build-systems.html#using-boot-starter>
``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
	<dependency>
		...       
    </dependency>
</dependencies>
```
--- 

## Main procedure class,main entrance class

### @SpringBootApplication
The `@SpringBootApplication` annotation is often placed on your main class, and it implicitly defines a base “search package” for certain items.  
  
For example, if you are writing a JPA application, the package of the `@SpringBootApplication` annotated class is used to search for `@Entity` items.  
  
 Using a root package also allows component scan to apply only on your project. 
``` xml
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

below is the detail of @SpringBootApplication,which is like a gather.
``` xml
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
```

### @SpringBootConfiguration
- @SpringBootConfiguration annotated class is an option class of Spring Boot.


### @EnableAutoConfiguration
Spring Boot auto-configuration attempts to automatically configure your Spring application based on the jar dependencies that you have added.    
  
You need to opt-in to auto-configuration by adding the `@EnableAutoConfiguration` or `@SpringBootApplication` annotations to one of your @Configuration classes.  
(Because the `@SpringBootConfiguration` have included @EnableAutoConfiguration,the auto-configuration wil automatically configure all component that in the same package with the `@SpringBootConfiguration`)
  

---

## Configuration file
  
Spring Boot default configuration in *src/main/resources/*.  
`application.yml` or `application.properties`

### 1.YAML basic
format like Python  
key:(space)value 
``` yml
server:
	port: 8081
	path: /halo
```
do not ignore case

### 2. key: value  
#### Object,Map  
key: value
``` yml
friends:
	name:ring
	age:20
```  

inline	
``` yml
friends:{name:ring,age:20}
```

#### Array  
-value
``` yml
pets:
	-cat
	-dog
```

inline  
``` yml
pet: [dog,cat]
```

### @ConfigurationProperties(prefix = "person")  
Bind all properties in this class with the properties in configuration file  
`prefix = "person"` means bind class properties with person properties.  
  
Only the component props can use the component function`@ConfigurationProperties`.  
``` java 
@Component
@ConfigurationProperties
public class XXX{
```
  
### @PropertySource(value = {"classpath:application2.yml'})   
Like @ConfigurationProperties ,but this can select another configuration file.  

### @ImportResource  
Import Spring configuration file,and   
........

 





---
