---
title: Spring Dependency Setup
date: 2025-11-19
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A practical cheatsheet on setting up MapStruct in a Spring Boot application, detailing dependencies, configuration, and common pitfalls for Java developers.
aliases:
  - Spring Boot MapStruct Setup Cheatsheet

---



# Spring Boot + MapStruct – Setup Cheatsheet

> Goal: **Use MapStruct mappers as Spring beans** in a Spring Boot app (Java 17+).

---

## Contents

0. [Quick checklist](#0-quick-checklist)  
1. [Dependencies](#1-dependencies)  
2. [Compiler / annotation processing](#2-compiler--annotation-processing)  
3. [Component model: how MapStruct becomes a Spring bean](#3-component-model-how-mapstruct-becomes-a-spring-bean)  
4. [Minimal end-to-end example](#4-minimal-end-to-end-example)  
5. [Where MapStruct puts generated code](#5-where-mapstruct-puts-generated-code)  
6. [IDE settings (IntelliJ)](#6-ide-settings-intellij)  
7. [Optional: Global `MapperConfig`](#7-optional-global-mapperconfig)  
8. [Common problems & fixes](#8-common-problems--fixes)  

---

## 0. Quick checklist

You’re done when:

- `mapstruct` + `mapstruct-processor` added  
- Annotation processing enabled (IDE + build)  
- `componentModel = "spring"` (global or per mapper)  
- A mapper interface exists and compiles  
- Spring can inject it (`@Autowired` / constructor)

---

## 1. Dependencies

### Maven

```xml
<dependencies>
    <!-- Spring Boot starters as usual -->

    <!-- MapStruct API -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.6.2</version>
    </dependency>

    <!-- MapStruct annotation processor -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>1.6.2</version>
        <scope>provided</scope>
    </dependency>

    <!-- Optional: Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.34</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

> Minimal requirement: `mapstruct` + `mapstruct-processor`.

---

## 2. Compiler / annotation processing {#2-compiler--annotation-processing}

### Maven – `maven-compiler-plugin`

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>1.6.2</version>
                    </path>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>1.18.34</version>
                    </path>
                </annotationProcessorPaths>

                <!-- OPTIONAL: global component model -->
                <compilerArgs>
                    <arg>-Amapstruct.defaultComponentModel=spring</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Gradle (Groovy) – `build.gradle`

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'

    implementation 'org.mapstruct:mapstruct:1.6.2'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.6.2'

    compileOnly 'org.projectlombok:lombok:1.18.34'
    annotationProcessor 'org.projectlombok:lombok:1.18.34'
}

tasks.withType(JavaCompile).configureEach {
    options.compilerArgs += [
        '-Amapstruct.defaultComponentModel=spring'
    ]
}
```

### Gradle (Kotlin) – `build.gradle.kts`

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")

    implementation("org.mapstruct:mapstruct:1.6.2")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.6.2")

    compileOnly("org.projectlombok:lombok:1.18.34")
    annotationProcessor("org.projectlombok:lombok:1.18.34")
}

tasks.withType<JavaCompile>().configureEach {
    options.compilerArgs.addAll(
        listOf("-Amapstruct.defaultComponentModel=spring")
    )
}
```

---

## 3. Component model: how MapStruct becomes a Spring bean

Two patterns — pick one.

### A) Global default (via compiler arg)

* Use `-Amapstruct.defaultComponentModel=spring`
* Mapper:

```java
import org.mapstruct.Mapper;

@Mapper
public interface CategoryWebMapper {
    CategoryResponse toResponse(Category category);
}
```

No `componentModel` on the mapper itself: generator will add `@Component`.

### B) Per-mapper config

```java
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface CategoryWebMapper {
    CategoryResponse toResponse(Category category);
}
```

Good when you don’t want **all** mappers to be Spring beans.

---

## 4. Minimal end-to-end example

### Domain model

```java
package com.example.shop.domain.category;

public record Category(Long id, String name) {}
```

### Web DTO

```java
package com.example.shop.web.category.dto;

public record CategoryResponse(Long id, String name) {}
```

### Mapper

```java
package com.example.shop.web.category;

import com.example.shop.domain.category.Category;
import com.example.shop.web.category.dto.CategoryResponse;
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring") // or rely on global default
public interface CategoryWebMapper {

    CategoryResponse toResponse(Category category);
}
```

### Use in controller

```java
package com.example.shop.web.category;

import com.example.shop.domain.category.Category;
import com.example.shop.web.category.dto.CategoryResponse;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class CategoryController {

    private final CategoryWebMapper mapper;

    public CategoryController(CategoryWebMapper mapper) {
        this.mapper = mapper;
    }

    @GetMapping("/api/categories/sample")
    public CategoryResponse sample() {
        Category domain = new Category(1L, "Books");
        return mapper.toResponse(domain);
    }
}
```

If this starts, compiles, and `/api/categories/sample` returns JSON → setup is correct.

---

## 5. Where MapStruct puts generated code

* Maven: `target/generated-sources/annotations/...`
* Gradle: `build/generated/sources/annotationProcessor/...`

If `CategoryWebMapperImpl` is **missing**:

* annotation processing not running
* compilation error in mapper
* `componentModel` typo (`"spring"` spelled wrong, etc.)

---

## 6. IDE settings (IntelliJ)

You can have everything correct and still get red squiggles if IntelliJ doesn’t run annotation processing.

* Settings → *Build, Execution, Deployment* → *Compiler* → *Annotation Processors*
* Check **“Enable annotation processing”** for the project/module.

Then do a **Rebuild**.

---

## 7. Optional: Global `MapperConfig`

For shared config (naming, null handling, policies).

```java
package com.example.shop.mapping;

import org.mapstruct.MapperConfig;
import org.mapstruct.ReportingPolicy;

@MapperConfig(
    componentModel = "spring",                  // default for all mappers
    unmappedTargetPolicy = ReportingPolicy.ERROR
)
public interface GlobalMapperConfig {
}
```

Use it:

```java
package com.example.shop.web.category;

import com.example.shop.mapping.GlobalMapperConfig;
import org.mapstruct.Mapper;

@Mapper(config = GlobalMapperConfig.class)
public interface CategoryWebMapper {
    // methods...
}
```

Nice place: `com.example.shop.mapping` or `com.example.shop.infra.mapping`.

---

## 8. Common problems & fixes {#8-common-problems--fixes}

**Problem:** `NoSuchBeanDefinitionException: No qualifying bean of type 'CategoryWebMapper'`
**Check:**

* Mapper has `@Mapper(componentModel = "spring")` or global arg is set.
* MapStruct processor is on classpath as **annotationProcessor** / `provided`.
* Annotation processing enabled in IDE.
* Project compiles without errors.

**Problem:** Generated class not visible in IDE
**Check:**

* Rebuild project
* Mark generated sources directory as "Generated sources root" (IntelliJ usually does this automatically)

