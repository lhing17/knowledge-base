## knife4j使用及相关问题

### 0. 什么是knife4j?

### 1. 引入knife4j不同版本
#### 1.1 Swagger2规范和OpenAPI3规范
- Swagger2规范：依赖Springfox项目，该项目目前几乎处于停更状态，但很多老项目依然使用的是该规范，所以Knife4j在更新前端Ui的同时也继续保持了兼容
- OpenAPI3规范：依赖Springdoc项目，更新发版频率非常快，建议开发者尽快迁移过来使用OpenAPI3规范,Knife4j后面的重心也会在这里。

#### 1.2 Spring Boot不同版本引入knife4j
Spring Boot 版本 | 使用规范 | 引入依赖
----- | ---- | ----
2.x | Swagger2 | knife4j-spring-boot-starter (knif4j版本<4.0)
2.x | Swagger2 | knife4j-openapi2-spring-boot-starter (knif4j版本>=4.0)
2.x | OpenAPI3 | knife4j-openapi3-spring-boot-starter
3.x | OpenAPI3 | knife4j-openapi3-jakarta-spring-boot-starter

### 2. knife4j开启增强模式
注意只有基于OpenAPI3规范的knife4j才支持增强模式。

要开启增强模式，需要在application.yml或application.properties文件中添加如下配置：

```yaml
knife4j:
  enable: true
```

也可以在Java配置类中使用@EnableKnife4j注解开启增强模式：

```java
@Configuration
@EnableKnife4j
public class Knife4jConfig {
}
```

### 3. knife4j接口分组和排序
``` java
/**
     * 创建API
     */
    public GroupedOpenApi createRestApi(String groupName, String basePackage) {
        return GroupedOpenApi.builder()
                .group(groupName)
                .packagesToScan(basePackage) // 指定包
                .addOperationCustomizer((operation, handlerMethod) -> {
                    // 只保留有 @Operation 标注的方法
                    if (!handlerMethod.hasMethodAnnotation(io.swagger.v3.oas.annotations.Operation.class)) {
                        return null; // 返回 null = 不生成文档
                    }
                    return operation;
                })
                .addOpenApiCustomiser(globalSecurityScheme())
                .build();
    }

    /**
     * 分组A
     */
    @Bean
    public GroupedOpenApi projectApi() {
        return createRestApi("分组A", "com.xlwx.foo.controller");
    }

    /**
     * 分组B
     */
    @Bean
    public GroupedOpenApi deviceApi() {
        return createRestApi("分组B", "com.xlwx.bar.controller");
    }
```

如上图所示，分组A和分组B分别对应了不同的Controller包，可以创建不同的Bean用来管理不同的接口分组。如果想要对接口排序，最简单的方式是在分组名前加入数字前缀，例如：01-分组A、02-分组B。

### 4. knife4j自定义页面UI

#### 4.1 隐藏Swagger Models菜单

要隐藏Swagger Models菜单，需要在application.yml或application.properties文件中添加如下配置：

```yaml
knife4j:
  setting:
    enable-swagger-models: false
```

#### 4.2 隐藏文档管理菜单

要隐藏文档管理菜单，需要在application.yml或application.properties文件中添加如下配置：

```yaml
knife4j:
  setting:
    enable-document-manage: false
```

#### 4.3 自定义主页

要自定义主页，需要在application.yml或application.properties文件中添加如下配置：

```yaml
knife4j:
  setting:
    enable-home-custom: true
    home-custom-path: classpath:/knife4j/index.md
```

其中，/knife4j/index.md是自定义的主页路径，开发者可以根据实际情况修改。
