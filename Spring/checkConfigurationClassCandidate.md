# ConfigurationClassPostProcessor 中如何判断一个bean定义是否为配置类

标签（空格分隔）： Spring

---

上文中我们已经介绍了 `ConfigurationClassPostProcessor` 配置类后置处理器的 `#processConfigBeanDefinitions` 方法中对于配置类的解析逻辑，本文中我们将介绍如何判断一个bean定义是一个配置类的方法，也就是 `ConfigurationClassUtils#checkConfigurationClassCandidate`

该方法的参数有单个bean的bean定义，元数据读取工厂，此处的元数据读取工厂是在初始化配置类后置处理器的类加载器时指定的。具体判断步骤如下：

1. 获取bean定义中的bean的类名，如果类名为空，且bean定义中的工厂方法名为空（工厂方法FactoryMethodName指用于生成bean实例对象的方法名称），则直接返回false，不是配置类。
2. 声明注解元数据对象 `AnnotationMetadata`
2. 如果bean定义类型继承或实现自 `AnnotatedBeanDefinition` 接口，并且bean定义中的类名和注解bean元数据中的类名一致，则获取bean的元数据
3. 如果bean定义类型继承或实现自 `AbstractBeanDefinition` 接口，且bean指定了 `beanClass` 属性，则加载 `beanClass` 类，判断是否实现了如下接口 `BeanFactoryPostProcessor` 、 `BeanPostProcessor` 、 `AopInfrastructureBean` 、 `EventListenerFactory` 之一，如果是，则返回false，不是配置类；如果没有实现指定接口，则根据 `beanClass` 类重新解析注解元数据
4. 否则直接使用元数据读取器从类名中加载注解元数据
5. 从获得的注解元数据中加载 `@Configuration` 注解的属性值，如果添加了该注解，这里会有两个属性值，分别为 `value` 和 `proxyBeanMethods`
6. 如果指定了 `@Configuration` 注解，且指定了 `proxyBeanMethods = true` ，设置bean定义的 `configurationClass` 属性值为 `full`；
7. 如果指定了 `@Configuration` 注解，或者指定了配置类候选注解，则设置bean定义的 `configurationClass` 属性值为 `full`，这里的配置类候选注解的判断方法指的是
    1. 非接口，并且声明了如下注解之一： `@Component` 、`@ComponentScan` 、 `@Import` 、 `ImportResource`，直接返回true，指定了配置类候选注解
    3. 非接口，并且包含使用 `@Bean` 注解的方法，则认为配置类候选注解
4. 如果第6条和第7条均不满足，则认为不是配置类，直接返回false
5. 如果配置类添加了 `@Order` 注解，将该注解的是放入元数据的 `order` 属性中。
6. 返回校验结果为true，该类为配置类