---
title: 布隆过滤器
date: 2020-03-20 17:40:24
tags:
- Guava
- BloomFilter
categories: Java
---

### 布隆过滤器


#### Guava 实现

> 创建并将数据持久化到文件
``` java
bloomFilter = BloomFilter.create(Funnels.stringFunnel(StandardCharsets.UTF_8), 500_0000, 0.001);
        try {
            InputStream existsUniqueCode = Object.class.getResourceAsStream("/exists_unique_code.json");

            String jsonStr = CharStreams.toString(new InputStreamReader(existsUniqueCode));
            List<String> stringList = JSON.parseArray(jsonStr, String.class);
            stringList.forEach(bloomFilter::put);

            FileOutputStream fs = new FileOutputStream(serializePath);
            ObjectOutputStream os = new ObjectOutputStream(fs);
            os.writeObject(bloomFilter);

            log.info("初始化union_code_filter成功");
        } catch (Exception e) {
            log.error("初始化union_code_filter失败", e);
        }
```


#### Redis 实现分布式布隆过滤器

>Redisson 底层使用bitset保存

``` java
RBloomFilter<SomeObject> bloomFilter = redisson.getBloomFilter("sample");
// 初始化布隆过滤器，预计统计元素数量为55000000，期望误差率为0.03
bloomFilter.tryInit(55000000L, 0.03);
bloomFilter.add(new SomeObject("field1Value", "field2Value"));
bloomFilter.add(new SomeObject("field5Value", "field8Value"));
bloomFilter.contains(new SomeObject("field1Value", "field8Value"));
```

#### Redis 4.0之后可通过添加`redislabs/rebloom`模块实现



