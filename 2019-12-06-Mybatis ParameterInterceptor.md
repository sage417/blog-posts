---
title: '[片段] Mybatis ParameterHandler 实践'
date: '2019-12-06 10:00:00'
thumbnail: https://i.loli.net/2019/12/04/RUNcmFlT5ajLCAo.jpg
tags:
    - mybatis
    - java
    - 代码
categories:
    - Java框架
    - Mybatis
---

用来批量加密用@Decrypted注解的String字段，可能还有一些坑。


```java

import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import com.ke.zhaopin.manage.server.config.mybatis.interceptor.anno.Decrypted;
import com.lianjia.ctt.kinko.spi.CipherSpi;
import com.sun.istack.internal.NotNull;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.apache.ibatis.binding.MapperMethod;
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.scripting.defaults.DefaultParameterHandler;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.defaults.DefaultSqlSession;
import org.joor.Reflect;
import reactor.core.publisher.Flux;

import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.sql.PreparedStatement;
import java.util.*;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import java.util.stream.Collectors;


@Intercepts({
        @Signature(type = ParameterHandler.class, method = "setParameters", args = {PreparedStatement.class}),
})
@Slf4j
public class EncryptInterceptor implements Interceptor {

    private static final String COLLECTION_KEY = "collection";
    private static final String ARRAY_KEY = "array";

    private final LoadingCache<Class, List<String>> decryptFieldCaches = CacheBuilder.newBuilder()
            .maximumSize(200)
            .expireAfterAccess(10L, TimeUnit.MINUTES)
            .build(new CacheLoader<Class, List<String>>() {
                       @Override
                       public List<String> load(Class key) {
                           return Arrays.stream(key.getDeclaredFields())
                                   .filter(f -> f.getAnnotation(Decrypted.class) != null)
                                   .filter(f -> {
                                       boolean isString = f.getType() == String.class;
                                       if (!isString) {
                                           log.warn(f.getName() + "is not String, actual type is " + f.getType().getSimpleName() + " ignored");
                                       }
                                       return isString;
                                   })
                                   .map(Field::getName)
                                   .collect(Collectors.toList());
                       }
                   }
            );

    private CipherSpi cipherSpi;

    public EncryptInterceptor(CipherSpi cipherSpi) {
        this.cipherSpi = cipherSpi;
    }

    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        Flux<CryptContext> contextFlux = Flux.empty();

        do {
            if (!(invocation.getTarget() instanceof DefaultParameterHandler)) break;

            final Reflect parameterHandler = Reflect.on(invocation.getTarget());
            final Object parameterObject = parameterHandler.get("parameterObject");
            final Configuration configuration = parameterHandler.get("configuration");


            if (parameterObject instanceof DefaultSqlSession.StrictMap) {
                // 单个Collection/Map/Array参数
                DefaultSqlSession.StrictMap<?> paramMap = (DefaultSqlSession.StrictMap<?>) parameterObject;

                Collection<?> collection = null;
                Class<?> componentType = null;
                if (paramMap.containsKey(COLLECTION_KEY)) {
                    collection = (Collection<?>) paramMap.get(COLLECTION_KEY);
                    componentType = collection.iterator().next().getClass();
                } else if (paramMap.containsKey(ARRAY_KEY)) {
                    Object[] array = (Object[]) paramMap.get(ARRAY_KEY);
                    componentType = array.getClass().getComponentType();
                    collection = Arrays.asList(array);
                }

                if (!isUserDefinedClass(componentType)) break;

                final Flux<CryptContext> collectionFlux = collection(configuration, collection, parameterObject.getClass());

                contextFlux = collectionFlux;

            } else if (parameterObject instanceof MapperMethod.ParamMap) {
                // 多个参数
                MapperMethod.ParamMap<?> paramMap = (MapperMethod.ParamMap<?>) parameterObject;

                final List<?> params = paramMap.values().stream().filter(Objects::nonNull).distinct().collect(Collectors.toList());

                for (Object parameter : params) {
                    if (parameter instanceof Collection) {
                        Collection<?> collection = (Collection<?>) parameter;
                        if (collection.isEmpty()) {
                            continue;
                        }

                        Class<?> componentType = collection.iterator().next().getClass();
                        if (!isUserDefinedClass(componentType)) {
                            continue;
                        }
                        final Flux<CryptContext> collectionFlux = collection(configuration, collection, componentType);
                        contextFlux = contextFlux.concatWith(collectionFlux);

                    } else if (parameter.getClass().isArray()) {
                        if (Array.getLength(parameter) == 0) continue;
                        final Class<?> componentType = parameter.getClass().getComponentType();
                        if (!isUserDefinedClass(componentType)) {
                            continue;
                        }
                        Collection<?> collection = Arrays.asList((Object[]) parameter);

                        final Flux<CryptContext> collectionFlux = collection(configuration, collection, componentType);
                        contextFlux = contextFlux.concatWith(collectionFlux);

                    } else if (isUserDefinedClass(parameter.getClass())) {
                        final Flux<CryptContext> singleFlux = collection(configuration, Collections.singletonList(parameter), parameter.getClass());
                        contextFlux = contextFlux.concatWith(singleFlux);
                    }
                }

            } else if (isUserDefinedClass(parameterObject.getClass())) {
                // 单个非Collection/Map/Array参数
                final Flux<CryptContext> singleFlux = collection(configuration, Collections.singletonList(parameterObject), parameterObject.getClass());
                contextFlux = singleFlux;
            } else {
                // 不是用interface的情况
            }


        } while (false);

        final List<CryptContext> cryptContexts = encrypt(contextFlux);

        invocation.proceed();

        restore(cryptContexts);

        return null;
    }

    private void restore(List<CryptContext> cryptContexts) {
        for (CryptContext cryptContext : cryptContexts) {
            cryptContext.metaObject.setValue(cryptContext.fieldName, cryptContext.value);
        }
    }

    private Flux<CryptContext> collection(Configuration configuration, Collection<?> collection, Class<?> componentType) throws ExecutionException {
        final List<String> fieldNames = this.getDecryptFields(componentType);

        return Flux.fromIterable(collection)
                .map(configuration::newMetaObject)
                .flatMapIterable(metaObject -> fieldNames.stream().map(fieldName -> new CryptContext(metaObject, fieldName)).collect(Collectors.toList()));
    }

    private List<CryptContext> encrypt(Flux<CryptContext> contextFlux) {
        final Flux<CryptContext> publish = contextFlux
                .filter(context -> StringUtils.isNotBlank(context.value));

        return publish.buffer(1000)
                .doOnNext(contexts -> {
                    Map<String, String> secretMap = Collections.emptyMap();
                    try {
                        secretMap = cipherSpi.batchEncrypt(contexts.stream().map(CryptContext::getValue).distinct().collect(Collectors.toList()));
                    } catch (Exception e) {

                    }
                    for (CryptContext context : contexts) {
                        context.secret = secretMap.get(context.value);
                    }
                })
                .flatMapIterable(Function.identity())
                .doOnNext(context -> context.metaObject.setValue(context.fieldName, context.secret))
                .collectList()
                .block();
    }

    @NotNull
    private List<String> getDecryptFields(Class<?> modelClazz) throws ExecutionException {
        return this.decryptFieldCaches.get(modelClazz);
    }

    private boolean isUserDefinedClass(Class<?> clazz) {
        return !clazz.isPrimitive() && !clazz.getPackage().getName().startsWith("java");
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
class CryptContext {

    CryptContext(MetaObject metaObject, String fieldName) {
        this.metaObject = metaObject;
        this.fieldName = fieldName;
        this.value = (String) metaObject.getValue(fieldName);
        if (StringUtils.isBlank(value)) {
            this.secret = StringUtils.EMPTY;
        }
    }

    final MetaObject metaObject;

    final String fieldName;

    final String value;

    String secret;
}

```