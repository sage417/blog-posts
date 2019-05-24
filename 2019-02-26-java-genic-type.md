---
title: '[片段]使用TypeToken在运行期保存泛型信息'
date: '2019-02-26 10:00:00'
tags:
    - java
    - 代码
categories:
    - Java基础
---

一般来说可以使用getGenericSuperclass 获取子类范型信息，但是泛型有嵌套的话想获取完整信息还是有点复杂的。例如：Message<List<T>> 有两个泛型信息。

guava中有强大的TypeToken帮助你保存复杂泛型信息，可以参考：

```java
ParameterizedTypeReference<Message<T>> responseTypeRef = 
	ParameterizedTypeReferenceBuilder.fromTypeToken(
        new TypeToken<Message<T>>() {}.where(new TypeParameter<T>() {}, new TypeToken<List<OrgSugVOV1>>() {}));
```

如果需要在spring框架中使用，需要一个适配器：

```java
public class ParameterizedTypeReferenceBuilder {

    public static <T> ParameterizedTypeReference<T> fromTypeToken(TypeToken<T> typeToken) {
        return new TypeTokenParameterizedTypeReference<>(typeToken);
    }

    private static class TypeTokenParameterizedTypeReference<T> extends ParameterizedTypeReference<T> {

        private final Type type;

        private TypeTokenParameterizedTypeReference(TypeToken<T> typeToken) {
            this.type = typeToken.getType();
        }

        @Override
        public Type getType() {
            return type;
        }

        @Override
        public boolean equals(Object obj) {
            return (this == obj || (obj instanceof ParameterizedTypeReference &&
                    this.type.equals(((ParameterizedTypeReference<?>) obj).getType())));
        }

        @Override
        public int hashCode() {
            return this.type.hashCode();
        }

        @Override
        public String toString() {
            return "ParameterizedTypeReference<" + this.type + ">";
        }
    }
}
```

关于java的泛型我就不多做吐槽了。