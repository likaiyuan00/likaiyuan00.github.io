---
title: 使用maven打包
date: 2025-05-12 10:25:14
tags: maven
---
# 使用springboot
```config
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.9</version>
        <relativePath/>
    </parent>

    <groupId>org.ecs</groupId>
    <artifactId>springboot01</artifactId>
    <version>1.0-SNAPSHOT</version>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```
# 其他
```config
<groupId>org.example</groupId>
    <artifactId>CpuLoad</artifactId>
    <version>1.0-SNAPSHOT</version>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <!-- 指定入口函数 -->                             
			    <mainClass>org.example.CpuLoad</mainClass>
                            <!-- 是否添加依赖的jar路径配置 -->
                            <addClasspath>true</addClasspath>
                            <!-- 依赖的jar包存放未知，和生成的jar放在同一级目录下 -->
                            <classpathPrefix>lib/</classpathPrefix>
                        </manifest>
                    </archive>
                    <!-- 不打包com.yh.excludes下面的所有类 -->
                    <excludes>com/xx/excludes/*</excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
