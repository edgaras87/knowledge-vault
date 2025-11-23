

## `yaml` Example - Pre-setup for Spring Slice  

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/appdb
    username: appuser
    password: apppass
  jpa:
    hibernate:
      ddl-auto: validate # or validate | update | create | create-drop as needed
    #show-sql: false
```


## dependency (Maven)


### MapStruct needs only **two dependencies** to work:

```xml
<!-- API dependency: -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.6.2</version>
</dependency>


<!-- Annotation processor: -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.6.2</version>
    <scope>provided</scope>
</dependency>
```



 