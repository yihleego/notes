# Proxy

## 动态代理

### JDK 动态代理

JDK 动态代理是通过拦截器和反射生成的匿名类实现的，所以至少需要实现一个接口。

```java
public interface Adder {
    int add(int a, int b);
}

public class AdderInvocationHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("add")) {
            return (int) args[0] + (int) args[1];
        }
        throw new UnsupportedOperationException();
    }
}
```

```java
public static void main(String[] args) {
    Adder adder = (Adder) Proxy.newProxyInstance(Adder.class.getClassLoader(), new Class[]{Adder.class}, new AdderInvocationHandler());
    int sum = adder.add(1, 2);
}
```

### CGLIB 动态代理

CGLIB 动态代理是通过继承和修改字节码实现的，所以父类不能为`final`，否则会报错`java.lang.IllegalArgumentException: Cannot subclass final class`。

```java
public class Adder {
    public int add(int a, int b) {
        return a + b;
    }
}

public class AdderMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before");
        Object value = methodProxy.invokeSuper(o, objects);
        System.out.println("after");
        return value;
    }
}
```

```java
public static void main(String[] args) {
    Enhancer e = new Enhancer();
    e.setSuperclass(Adder.class);
    e.setCallback(new AdderMethodInterceptor());
    Adder adder = (Adder) e.create();
    int sum = adder.add(1, 2);
}
```