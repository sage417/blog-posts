---
title: '[片段] Mybatis ResultSetHandler实践-续'
date: '2019-04-04 10:00:00'
tags:
    - mybatis
    - java
    - 代码
categories:
    - Java框架
    - Mybatis
---

这次拦截的方法是handleResultSets(Statement stmt)，用来批量解密用@Encrypted注解的String字段。

上次的局限是只能批量解密一个对象的所有加密字段，对批量数据来说稍显不足，这个主要改进了这一点。



```java
  @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
```





```java
package app.pooi.common.encrypt;


import app.pooi.common.encrypt.anno.CipherSpi;
import app.pooi.common.encrypt.anno.Encrypted;
import lombok.Getter;
import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.SystemMetaObject;

import java.lang.reflect.Field;
import java.sql.Statement;
import java.util.*;
import java.util.function.Function;
import java.util.logging.Logger;
import java.util.stream.Collectors;


@Intercepts({
        @Signature(type = ResultSetHandler.class, method = "handleResultSets", args = {Statement.class}),
})
public class DecryptInterceptor implements Interceptor {

    private static final Logger logger = Logger.getLogger(DecryptInterceptor.class.getName());

    private CipherSpi cipherSpi;

    public DecryptInterceptor(CipherSpi cipherSpi) {
        this.cipherSpi = cipherSpi;
    }

    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        final Object proceed = invocation.proceed();

        if (proceed == null) {
            return proceed;
        }

        List<?> results = (List<?>) proceed;

        if (results.isEmpty()) {
            return proceed;
        }

        final Object first = results.iterator().next();

        final Class<?> modelClazz = first.getClass();

        final List<String> decryptFields = getDecryptFields(modelClazz);

        if (decryptFields.isEmpty()) {
            return proceed;
        }

        final List<List<String>> secret = Flux.fromIterable(results)
                .map(SystemMetaObject::forObject)
                .flatMapIterable(mo -> decryptFields.stream().map(mo::getValue).collect(Collectors.toList()))
                .cast(String.class)
                .buffer(1000)
                .collectList()
                .block();

        final Map<String, String> secretMap = secret.stream()
                .map(secrets -> {
                    try {
                        return cipherSpi.batchDecrypt(secrets);
                    } catch (Exception e) {
                        e.printStackTrace();
                        return Maps.<String, String>newHashMap();
                    }
                }).reduce(Maps.newHashMap(), (m1, m2) -> {
                    m1.putAll(m2);
                    return m1;
                });

        secretMap.put("", "0");

        for (Object r : results) {
            final MetaObject metaObject = SystemMetaObject.forObject(r);
            decryptFields.forEach(f -> metaObject.setValue(f, secretMap.get(metaObject.getValue(f))));
        }

        return results;
    }

    @NotNull
    private List<String> getDecryptFields(Class<?> modelClazz) {
        return Arrays.stream(modelClazz.getDeclaredFields())
                .filter(f -> f.getAnnotation(Decrypted.class) != null)
                .filter(f -> {
                    boolean isString = f.getType() == String.class;
                    if (!isString) {
                        logger.warning(f.getName() + "is not String, actual type is " + f.getType().getSimpleName() + " ignored");
                    }
                    return isString;
                })
                .map(Field::getName)
                .collect(Collectors.toList());
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}

@Getter
class Tuple2<T1, T2> {

    private final T1 t1;

    private final T2 t2;

    Tuple2(T1 t1, T2 t2) {
        this.t1 = t1;
        this.t2 = t2;
    }

    static <T1, T2> Tuple2<T1, T2> of(T1 t1, T2 t2) {
        return new Tuple2<>(t1, t2);
    }
}
```

