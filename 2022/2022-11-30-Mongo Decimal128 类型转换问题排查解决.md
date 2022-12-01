---
title: 'Mongo Decimal128 类型转换问题排查解决'
date: '2022-11-30 00:00:00'
cover: https://i.loli.net/2021/03/25/1iIC9XgetrluNRQ.jpg
toc: true
categories:
    - 工作
    - 挑战
---


## 问题背景
java中对于精确小数，我们通常使用Bigdecimal进行存储，而mongo中是不存在Bigdecimal类型，对应的是Decimal128。
项目中使用mongo的地方，为了能够在插入mongo时将Bigdecimal转为Decimal128，查询时将Decimal128转回Bigdecimal，可以利用spring中的org.springframework.core.convert.converter.Converter。如下

```java
@WritingConverter
public class BigDecimalToDecimal128Converter implements Converter<BigDecimal, Decimal128> {
 
    @Override
    public Decimal128 convert(BigDecimal bigDecimal) {
        return new Decimal128(bigDecimal);
    }
}
```
```java
@ReadingConverter
public class Decimal128ToBigDecimalConverter implements Converter<Decimal128, BigDecimal> {
 
    @Override
    public BigDecimal convert(Decimal128 decimal128) {
        return decimal128.bigDecimalValue();
    }
}
```
在向容器中注入MongoTemplate时添加自定义的Converter后
我们就可以自由的对BigDecimal类型进行mongo存储和读取了。
## 真的没有问题了吗
当使用Map类型去接收Mongo查询结果时，上面的自定义Converter就会失效：
```java
return mongoTemplate.find(query, JSONObject.class, TABLE_NAME + year);
```
![企业微信截图_16535653001637.png](https://object-storage.mihoyo.com:9000/plat-knowledge-management/prod/92d2afc9c6cac8dd3ca5391f046577f0_1653616202505.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=plat-knowledge-management-admin%2F20221130%2F%2Fs3%2Faws4_request&X-Amz-Date=20221130T124433Z&X-Amz-Expires=21600&X-Amz-SignedHeaders=host&X-Amz-Signature=9df161b1d3a1811f4b65f2e064e1a2084aae5d5b340c2c82da580e165c5f2226 =100%x)
经过代码调试，定位到MappingMongoConverter负责类型转换，其核心代码为：

```java
 private Object getPotentiallyConvertedSimpleRead(@Nullable Object value, @Nullable Class<?> target) {
  // 使用Map<String,Object>接收结果，target=Object.class 而Decimal128 是 Object子类，因此不进行转换
  if (value == null || target == null || ClassUtils.isAssignableValue(target, value)) {
     return value;
  }
  
  if (conversions.hasCustomReadTarget(value.getClass(), target)) {
   return conversionService.convert(value, target);
  }

  if (Enum.class.isAssignableFrom(target)) {
   return Enum.valueOf((Class<Enum>) target, value.toString());
  }

  return conversionService.convert(value, target);
 }
```
由此发现使用Map<String,Object>接收结果时，target=Object.class ，而Decimal128属于Object子类，因此自定义Converter不生效
## 一波三折的解决方案-增加自定义Converter
由于Decimal128在下游系统中不存在，因此下游接收到数字被转换为Map类型且很难再次转换，因此准备从源头解决问题，修改类型转换逻辑，尝试将Decimal128转换为BigDecimal输出至下游，因此增加新的自定义Converter
```java
@ReadingConverter
public class Decimal128ToObjectConverter implements Converter<Decimal128, Object> {
    public Decimal128ToObjectConverter() {
    }

    @Override
    public Object convert(Decimal128 decimal128) {
        return decimal128.bigDecimalValue();
    }
}
```
结论是不行的，原因为子类判断优先自定义Converter转换：
```java
 private Object getPotentiallyConvertedSimpleRead(@Nullable Object value, @Nullable Class<?> target) {
  
  if (value == null || target == null || ClassUtils.isAssignableValue(target, value)) {
     return value;
  }
  // 优先判断是否为子类，所以即使存在自定义Converter，也不会走该逻辑
  if (conversions.hasCustomReadTarget(value.getClass(), target)) {
   return conversionService.convert(value, target);
  }

  if (Enum.class.isAssignableFrom(target)) {
   return Enum.valueOf((Class<Enum>) target, value.toString());
  }

  return conversionService.convert(value, target);
 }
```
## 第二折 自定义MappingMongoConverter
我们自定义转换逻辑，重写核心转换逻辑，使得自定义Converter优先转换，是否还有问题？
```java
 private Object getPotentiallyConvertedSimpleRead(@Nullable Object value, @Nullable Class<?> target) {
  
  if (value == null || target == null) {
     return value;
  }
  
  if (conversions.hasCustomReadTarget(value.getClass(), target)) {
   return conversionService.convert(value, target);
  }
// 将 子类判断移动至自定义Converter之后
 if (ClassUtils.isAssignableValue(target, value)) {
    return value;
  }

  if (Enum.class.isAssignableFrom(target)) {
   return Enum.valueOf((Class<Enum>) target, value.toString());
  }

  return conversionService.convert(value, target);
 }
```
可以看到，当target=Object.class时，已经可以成功将Decimal128转换为BigDecimal，但是target=null时还是无法转换
## 第三折 解决嵌套转换问题
当对象存在嵌套时，MappingMongoConverter默认使用Map类型进行类型转换，此时target=null，造成了嵌套对象中的Decimal128对象并没有走到自定义Converter转换逻辑，因此再次修改代码：
```java
 private Object getPotentiallyConvertedSimpleRead(@Nullable Object value, @Nullable Class<?> target) {
  
  if (value == null) {
       return value;
  }
  // target == null 时，尝试寻找Object.class类型转换器
  if (conversions.hasCustomReadTarget(value.getClass(), ObjectUtils.defaultIfNull(target, Object.class))) {
   	return conversionService.convert(value, ObjectUtils.defaultIfNull(target, Object.class));
  }
	// 如果没有匹配到 converter 则返回原始值
  if (target == null || ClassUtils.isAssignableValue(target, value)) {
    return value;
  }

  if (Enum.class.isAssignableFrom(target)) {
   return Enum.valueOf((Class<Enum>) target, value.toString());
  }

  return conversionService.convert(value, target);
 }
```
至此使用Map接收Decimal128将被正确的转换至BigDecimal类型
## 总结与收获

此方案需要改动两处代码

1. 增加Decimal128ToObjectConverter
2. 自定义MappingMongoConverter

排查后收获:

1. 除非上游改动代价太大，尽量在上游解决问题，不要抛给下游处理
2. mongo-spring对于Map等动态类型转换支持不完善，尽量定义准确类型数据结构接收结果，重写MappingMongoConverter不是上策
3. 固定位数小数也可转换为整数存储，来绕过数字精度和类型转换问题
