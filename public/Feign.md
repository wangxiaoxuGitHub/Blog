---
title: Spring Cloud OpenFeign上传文件
date: 2020-05-08
tags:
- Spring Cloud
- Feign
categories: Java
---

![20200508112621](https://img.yjll.art/img/20200508112621.png)

[Feign](https://github.com/OpenFeign/feign)是一个声明式的伪Http客户端，它使得写Http客户端变得更简单。

`Spring Cloud OpenFeign`是`Spring Could`服务间调用的基础组件,将Feign进行了封装，更便于使用，本文使用`Spring Cloud OpenFeign`进行文件上传。


<!-- more -->


>`spring-cloud-openfeign`版本`2.1.1.RELEASE`

声明使用Form表单形式提交
``` java
@Configuration
public class MultipartSupportConfig {
    @Bean
    public Encoder feignFormEncoder() {
        return new SpringFormEncoder();
    }
}

```

`consumes`指定`MediaType`为`multipart/form-data`

``` java
@FeignClient(name = "test", url = "http://localhost:8141", path = "p/c", configuration = MultipartSupportConfig.class)
public interface UploadFileClient {

    @RequestMapping(path = "uploadPrintedCodesFile", consumes = MULTIPART_FORM_DATA_VALUE)
    Object uploadPrintedCodesFile(UploadData uploadData);

}
```





