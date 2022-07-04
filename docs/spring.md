# Spring

## IoC

## AOP

## Bean

### 生命周期

Spring 中通过`BeanDefinition`类描述 Bean 的信息。Spring 容器启动的时候，会扫描 XML、注解和配置中需要被 Spring 创建的 Bean 信息，然后将这些信息封装成`BeanDefinition`对象形式保存在`beanDefinitionMap`中，接着会遍它进行创建对象操作

Bean 的生命周期大致分为四个阶段：

1. 实例化
2. 注入属性
3. 初始化
    1. 如果 Bean 实现了`Aware`相关接口，则进行对应的逻辑处理，例如：`BeanNameAware`接口的`setBeanName`方法、`ApplicationContextAware`接口的`setApplicationContext`方法。
    2. 如果容器中存在实现了`BeanPostProcessor`接口的对象，则调用`postProcessBeforeInitialization`方法。
    3. 如果 Bean 中存在`@PostConstruct`注解修饰的方法，则调用该方法。（支持多个）
    4. 如果 Bean 实现了`InitializingBean`接口，则调用`afterPropertiesSet`方法。
    5. 如果声明了`init-method`方法，则调用该方法。（通过 XML 声明）
    6. 如果容器中存在实现了`BeanPostProcessor`接口的对象，则调用`postProcessAfterInitialization`方法。
4. 销毁
    1. 如果 Bean 实现了`DisposableBean`接口，则调用`destroy`方法。
    2. 如果声明了`destroy-method`方法，则调用该方法。（通过 XML 声明）

![spring_bean_lifecycle](images/spring_bean_lifecycle.png)

### 三级缓存

Spring 中使用了三级缓存解决了单例循环依赖和代理的问题。

```
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap(256);
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap(16);
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap(16);
}
```

| 名称                      | 级别    | 说明                            |
|:------------------------|:------|:------------------------------|
| `singletonObjects`      | 第一级缓存 | 保存已初始化的完整单例对象                 |
| `earlySingletonObjects` | 第二级缓存 | 保存未初始化的半成品单例对象                |
| `singletonFactories`    | 第三级缓存 | 存放完成品单例对象，用与生成半成品单例对象并放入到二级缓存 |

#### 二级缓存能否解决循环引用？

如果存在`对象A`和`对象B`互相依赖，通过二级缓存解决循环引用：

|     | 对象A                                        | 对象B                            |
|:----|:-------------------------------------------|:-------------------------------|
| 1   | 实例化`对象A`                                   |                                |
| 2   | 将`对象B`注入到`对象A`中，发现`对象B`不存在，先将`对象A`放入半成品缓存中 |                                |
| 3   |                                            | 实例化`对象B`                       |
| 4   |                                            | 将`对象A`注入到`对象B`中，从半成品缓存中获取`对象A` |
| 5   |                                            | 初始化完成，将`对象B`放入完成品缓存中           |
| 6   | 重试将`对象B`注入到`对象A`中，从完成品缓存中获取`对象B`           |                                |
| 7   | 初始化完成，将`对象A`从半成品缓存中移动至完成品缓存中               |                                |

如果`对象A`和`对象B`需要动态代理，可以在实例化时立即创建代理，并将代理对象放入缓存中。

为了验证上述流程，我们对`AbstractAutowireCapableBeanFactory`类的源码进行略微改造：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
  // ...

  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
          isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
          logger.trace("Eagerly caching bean '" + beanName +
                  "' to allow for resolving potential circular references");
      }
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
      
      // 新增一行代码，立即从三级缓存取出对象放入二级缓存（从三级缓存中取出的对象，可能是新创建的代理对象）
      getSingleton(beanName, true);
  }

  // ...
}
```

将实例化的对象在放入三级缓存之后，立刻将对象移动至二级缓存，如此以来，三级缓存的作用就被跳过了，就相当于只有二级缓存。
经过测试，结果是可行的，并且从源码上分析可以得出两种方式性能是一样的。二级缓存就可以解决循环引用的问题，为什么 Spring 选择使用三级缓存方式？

#### 为什么 Spring 使用三级缓存？

第三级缓存的作用就是：产生了一个工厂对象，但是并没有真实的利用这个工厂立即创建代理对象，也就是代理对象的创建是延迟的。

通常来讲，代理对象是在对象初始化之后返回的，如果要使用二级缓存解决循环依赖，意味着对象在实例化完后就创建代理对象，动态代理流程就被提前到了初始化之前，导致生命周期不明确，这样违背了 Spring 设计原则。

Spring 将 AOP 和 Bean 的生命周期结合，是在 Bean 创建完全之后通过`AnnotationAwareAspectJAutoProxyCreator`后置处理器来完成的，在后置处理器的`postProcessAfterInitialization`方法中对初始化后的 Bean 进行动态代理。
如果出现了循环依赖，才会让这个代理对象被提前创建。从设计模式的角度说，动态代理是一种设计模式，动态代理的主要工作是增强，而不是构造。

#### 三级缓存循环引用的代理单例示例

如果存在`对象A`和`对象B`互相依赖，且都需要代理的话，创建流程如下：

|     | 对象A                                   | 对象B                                                                                        |
|:----|:--------------------------------------|:-------------------------------------------------------------------------------------------|
| 1   | 实例化`对象A`                              |                                                                                            |
| 2   | 将`对象A`封装为`ObjectFactory`放入第三级缓存中      |                                                                                            |
| 3   | 准备将`对象B`注入到`对象A`中，发现`对象B`不存在，先处理`对象B` |                                                                                            |
| 4   |                                       | 实例化`对象B`                                                                                   |
| 5   |                                       | 将`对象B`封装为`ObjectFactory`放入第三级缓存中                                                           |
| 6   |                                       | 从第三级缓存中获取`对象A`的`ObjectFactory`，并创建`对象A代理`，将`对象A`的`ObjectFactory`从第三级缓存中移除，将`对象A代理`放入第二级缓存中 |
| 7   |                                       | 将`对象A代理`注入到`对象B`中                                                                          |
| 8   |                                       | 完成`对象B`初始化                                                                                 |
| 9   |                                       | 创建`对象B代理`，将`对象B`的`ObjectFactory`从第三级缓存中移除，将`对象B代理`放入第一级缓存中                                 |
| 10  | 从第一级缓存中获取`对象B代理`                      |                                                                                            |
| 11  | 将`对象B代理`注入到`对象A`中                     |                                                                                            |
| 12  | 完成`对象A`初始化                            |                                                                                            |
| 13  | 将`对象A代理`从第二级缓存中移除，将`对象A代理`放入第一级缓存中    |                                                                                            |

