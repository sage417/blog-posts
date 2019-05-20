---
title: '[片段] @CreatedBy / @ModifiedBy 拦截器实现'
date: '2019-02-11 10:00:00'
tags:
    - mybatis
    - java
    - 代码
---


拦截器实现：

```java
package app.pooi.common.entity;

import app.pooi.common.entity.anno.CreatedBy;
import app.pooi.common.entity.anno.ModifiedBy;
import lombok.Data;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;

import java.util.Arrays;
import java.util.Properties;
import java.util.function.Supplier;

@Data
@Intercepts({
        @Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class}),
})
public class EntityInterceptor implements org.apache.ibatis.plugin.Interceptor {

    private Supplier<Long> auditorAware;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        Executor executor = (Executor) invocation.getTarget();

        MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
        Object o = invocation.getArgs()[1];

        Arrays.stream(o.getClass().getDeclaredFields())
                .forEach(field -> {
                    final CreatedBy createdBy = field.getAnnotation(CreatedBy.class);
                    final ModifiedBy modifiedBy = field.getAnnotation(ModifiedBy.class);

                    if (createdBy != null || modifiedBy != null) {
                        field.setAccessible(true);
                        try {
                            field.set(o, auditorAware.get());
                        } catch (IllegalAccessException ignore) {
                        }
                    }
                });

        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```



配置：

```java
    @Configuration
    static class MybatisInterceptorConfig {

        @Bean
        public Interceptor[] configurationCustomizer(CipherSpi cipherSpi) {
            final EntityInterceptor entityInterceptor = new EntityInterceptor();

            entityInterceptor.setAuditorAware(() -> {
                final String header = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest().getHeader(XHeaders.LOGIN_USER_ID);
                return Long.valueOf(header);
            });
            return new Interceptor[]{new DecryptInterceptor(cipherSpi), entityInterceptor};
        }
    }   
```

