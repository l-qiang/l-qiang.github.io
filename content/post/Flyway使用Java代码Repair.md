---
title: "Flyway使用Java代码Repair"
date: 2021-06-01T15:31:42+08:00
lastmod: 2021-06-01T15:31:42+08:00
categories: ["Spring Boot", "Flyway"]
tags: ["Spring Boot", "Flyway"]
keywords: ["Spring Boot", "Flyway", "Repair"]
toc: false
draft: false
---

<!--more-->

Repair的作用：

> - Remove failed migration entries (only for databases that do NOT support DDL transactions)
> - Realign the checksums, descriptions, and types of the applied migrations with the ones of the available migrations
> - Mark all missing migrations as **deleted**

1. Maven依赖

   ```xml
   <dependency>
   	<groupId>org.flywaydb</groupId>
   	<artifactId>flyway-core</artifactId>
   </dependency> 
   ```

   

2. Java配置

   ```java
   @Component
   public class FlywayConfig implements FlywayMigrationStrategy{
   
       @Override
       public void migrate(Flyway flyway) {
           flyway.repair();
           flyway.migrate();
       }
   
   }
   ```
