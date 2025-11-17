---
title: "利用 Docker 分层，构建高效的 Spring Boot 镜像"
date: 2024-12-03T15:49:51+08:00
draft: true
tags:
- docker
- spring-boot
- 性能优化
- 运维部署
---

在实际项目中，我们偶尔会遇到由于客户部署环境的网络限制问题，导致 Docker 镜像的推送与拉取速度很慢，影响部署的效率。
最近接手的一个项目中，我就遇到了类似的情况，我们在客户的内网部署了 Docker 的私有仓库，Jenkins 打包完后需要推送到 Docker 仓库中。
由于客户基于安全要求对于内部网络之间的传输做了网速的限制，通过传统的 Spring Boot 的 FatJar 生成的 Docker 镜像过大，在每次推送镜像和拉取镜像都需要花不少时间。
我们在私有仓库部署的服务较多，对私有仓库的存储容量要求较高。

为了解决这个问题，我通过优化 Docker Layers 的方式大幅减少了镜像的大小变化，提高了部署效率。下面是完整的实战过程，希望对你有所帮助。

# FatJar 介绍
默认情况下，Spring Boot 的应用会采用 FatJar 的模式来将我们的 Spring Boot 应用打包成一个可执行的文件。通过如下的 Dockerfile 可以构建出该服务的 Docker 镜像用来启动服务
```sh
FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
java -jar app.jar -Dspring.profile.active=dev
```

FatJar 是一种将所有依赖打包到单个可执行 JAR 文件中的格式，采用这种设计方便了开发和运行，但也带来了一个明显的缺点：

- JAR 文件过大：常规的 Spring Boot 应用可能生成 100MB 到 200MB 的 JAR 文件，甚至更大。
- 依赖关系难以拆分：每次应用代码修改后，整个 JAR 文件都需要重新打包并更新。

我们可以使用 unzip -v 来确认这一点

在 Docker 镜像中，FatJar 通常被直接复制到镜像中，导致镜像每次更新时即使只有很小的代码变动，也需要重新推送整个镜像。这对于网络受限的环境来说，是不可接受的。

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<executable>true</executable>
	</configuration>
</plugin>
```

# Docker 镜像
Docker 构建镜像时，每一层都会生成一个可缓存的 Layer。如果某一层没有变化，Docker 就会复用缓存，而无需重新上传或下载。基于这一点，我们可以将 Spring Boot 应用分层打包，将不常变动的依赖和经常变动的代码分开，优化镜像的传输效率。

# 解决方案
基于 Docker Layers 优化镜像构建

```xml
<project>
    <build>
      <plugins>
        <plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId> 
          <configuration>
            <layers>
              <enabled>true</enabled>
            </layers>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </project>
```

# 优化的效果

```dockerfile
# Perform the extraction in a separate builder container
FROM bellsoft/liberica-openjre-debian:17-cds AS builder
WORKDIR /builder
# This points to the built jar file in the target folder
# Adjust this to 'build/libs/*.jar' if you're using Gradle
ARG JAR_FILE=target/*.jar
# Copy the jar file to the working directory and rename it to application.jar
COPY ${JAR_FILE} application.jar
# Extract the jar file using an efficient layout
RUN java -Djarmode=tools -jar application.jar extract --layers --destination extracted

# This is the runtime container
FROM bellsoft/liberica-openjre-debian:17-cds
WORKDIR /application
# Copy the extracted jar contents from the builder container into the working directory in the runtime container
# Every copy step creates a new docker layer
# This allows docker to only pull the changes it really needs
COPY --from=builder /builder/extracted/dependencies/ ./
COPY --from=builder /builder/extracted/spring-boot-loader/ ./
COPY --from=builder /builder/extracted/snapshot-dependencies/ ./
COPY --from=builder /builder/extracted/application/ ./
# Start the application jar - this is not the uber jar used by the builder
# This jar only contains application code and references to the extracted jar files
# This layout is efficient to start up and CDS friendly
ENTRYPOINT ["java", "-jar", "application.jar"]
```

# 总结

通过优化 Docker 镜像分层，我们成功解决了网络受限环境下的部署效率问题：

1. **减少镜像传输量**：只有应用代码层需要更新，依赖层可以复用缓存
2. **提升部署速度**：镜像推送和拉取时间大幅缩短
3. **降低存储压力**：减少了私有仓库的存储占用

这种优化方式特别适合：
- 网络带宽受限的环境
- 需要频繁部署更新的项目
- 对部署速度有较高要求的场景

希望这个实战经验对你有帮助！如果你也遇到过类似的部署问题，欢迎在评论区分享你的解决方案。
