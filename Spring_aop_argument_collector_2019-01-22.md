---
title: '[片段] Java收集方法参数+Spring DataBinder'
date: '2019-01-22 0:00:01'
tags:
    - java
    - spring
    - aop
    - 代码
---

## 【片段】 Java收集方法参数+Spring DataBinder

#### 收集参数

目前是使用了spring aop 来拦截方法调用，把方法参数包装成Map形式

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CollectArguments {
}

@Aspect
public class ArgumentsCollector {

    private static final ThreadLocal<Map<String, Object>> ARGUMENTS = ThreadLocal.withInitial(ImmutableMap::of);

    static Map<String, Object> getArgs() {
        return ARGUMENTS.get();
    }

    private Object[] args(Object[] args, int exceptLength) {
        if (exceptLength == args.length) {
            return args;
        }

        return Arrays.copyOf(args, exceptLength);
    }

    @Pointcut("@annotation(CollectArguments)")
    void collectArgumentsAnnotationPointCut() {
    }

    @Before("collectArgumentsAnnotationPointCut()")
    public void doAccessCheck(JoinPoint joinPoint) {
        final String[] parameterNames = ((MethodSignature) joinPoint.getSignature()).getParameterNames();
        final Object[] args = args(joinPoint.getArgs(), parameterNames.length);

        ARGUMENTS.set(Collections.unmodifiableMap((IntStream.range(0, parameterNames.length - 1)
                .mapToObj(idx -> Tuple2.of(parameterNames[idx], args[idx]))
                .collect(HashMap::new, (m, t) -> m.put(t.getT1(), t.getT2()), HashMap::putAll))));
    }

    @After("collectArgumentsAnnotationPointCut()")
    public void remove() {
        ARGUMENTS.remove();
    }

    @Data
    private static class Tuple2<T1, T2> {

        private T1 t1;
        private T2 t2;

        Tuple2(T1 t1, T2 t2) {
            this.t1 = t1;
            this.t2 = t2;
        }

        public static <T1, T2> Tuple2<T1, T2> of(T1 t1, T2 t2) {
            return new Tuple2<>(t1, t2);
        }
    }
}
```

#### 通过Map构造对象

```java
public class BinderUtil {

    BinderUtil() {
    }

    @SuppressWarnings("unchecked")
    public static <T> T getTarget(Class<T> beanClazz) {
        final DataBinder binder = new DataBinder(BeanUtils.instantiate(beanClazz));
        binder.bind(new MutablePropertyValues(ArgumentsCollector.getArgs()));
        return (T) binder.getTarget();
    }
}
```

