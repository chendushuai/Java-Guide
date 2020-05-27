# 配置类后置处理器解析配置类进行bean定义注册的postProcessBeanDefinitionRegistry方法说明

标签（空格分隔）： Spring

---

在上文中，我们已经说明 `ConfigurationClassPostProcessor` 配置类后置处理器是在初始化注解bean定义读取器 `AnnotatedBeanDefinitionReader` 时添加到bean定义 `beanDefinitionMap` 中的。

在上下文对象的刷新 `#refresh()` 方法中，会调用到 `#invokeBeanFactoryPostProcessors` 方法，继续调用 `PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());` 方法执行bean工厂的后置处理器，具体执行顺序为：

1. 先使用 `getBeanFactoryPostProcessors()` 获取注册在上下文对象中的程序员自己通过 `org.springframework.context.support.AbstractApplicationContext#addBeanFactoryPostProcessor` 方法注册的bean工厂后置处理器 `BeanFactoryPostProcessor` 列表
2. 如果bean工厂继承或实现自 `BeanDefinitionRegistry`
    1. 判断bean工厂后置处理器是否继承或实现自 `BeanDefinitionRegistryPostProcessor` 接口，如果是，调用其 `#postProcessBeanDefinitionRegistry()` 方法
    2. 继续在bean工厂的bean定义集合中查找继承或实现自 `BeanDefinitionRegistryPostProcessor` 接口的bean定义，这里符合条件的只有 `ConfigurationClassPostProcessor` 配置类后置处理器（**注意，这里会实例化查找出来的后置处理器bean，也就是 `ConfigurationClassPostProcessor`**），调用其 `#postProcessBeanDefinitionRegistry()` 方法
    3. 继续在bean工厂的bean定义集合中查找继承或实现自 `BeanDefinitionRegistryPostProcessor` 接口的bean定义，**重新查找的原因是因为 `ConfigurationClassPostProcessor` 配置类后置处理器会解析配置类，在解析的过程中，可能会扫描添加指定包下使用注解添加的后置处理器Bean**，然后筛选出没有进行过处理的后置处理器，调用其 `#postProcessBeanDefinitionRegistry()` 方法
    4. 循环再次对于bean工厂里的bean定义集合中的继承或实现自 `BeanDefinitionRegistryPostProcessor` 接口的bean定义进行扫描处理，直至所有的符合条件的bean定义注册后置处理器全部调用 `#postProcessBeanDefinitionRegistry()` 方法执行完毕。
    5. 调用截止到当前所有被筛选出来的Bean定义注册后置处理器的 `#postProcessBeanFactory()` 方法
    6. 调用截止到当前所有被筛选出来的常规后置处理器（仅指程序员调用 `#addBeanFactoryPostProcessor` 方法注册到上下文对象中的，且非继承和实现自 `BeanDefinitionRegistryPostProcessor` 接口的bean定义）的 `#postProcessBeanFactory()` 方法。
3. 直接调用程序员自己注入的bean定义注册后置处理器的 `#postProcessBeanFactory()` 方法。
4. 在bean工厂的bean定义集合中查找继承或实现自 `BeanFactoryPostProcessor` 的bean工厂后置处理器集合（这里通过注解方式添加的bean工厂后置处理器bean就可以扫描出来了），排除掉已经处理的处理器后，调用其 `#postProcessBeanFactory()` 方法。

下面我们着重来介绍一下 `ConfigurationClassPostProcessor` 配置类后置处理器的 `#postProcessBeanDefinitionRegistry`方法

1. 根据给定的bean定义注册器（此处给定的参数值是我们的bean工厂，类型为 `DefaultListableBeanFactory`，实现了 `BeanDefinitionRegistry` 接口）生成 `registryId`，判断是否被重复调用了 `postProcessBeanDefinitionRegistry` 方法或 `postProcessBeanFactory` 方法，如果已经调用，则抛出异常
2. 记录该ID已经调用过 `postProcessBeanDefinitionRegistry` 方法了
3. 调用其 `#processConfigBeanDefinitions` 方法处理配置bean定义
    1. 循环遍历并判断bean定义中是否已经设置了 `configurationClass` 属性，如果设置了，则认为已经将该bean定义判断为配置类了；否则调用 `ConfigurationClassUtils.checkConfigurationClassCandidate` 方法判断bean定义是否是一个配置类bean定义，如果是，则将其添加到候选集合中。（*关于如何判断bean定义是否是一个配置类bean定义，我们另开文章来说明*）
2. 如果候选集合不为空，且配置类指定了加载顺序，则进行排序。
3. 判断是否指定了bean名称生成器 `BeanNameGenerator` ，指定的方式是直接将该对象添加到bean工厂的单例对象集合中。
4. 创建配置类解析器 `ConfigurationClassParser` ，用于解析配置类
5. 循环调用执行下面部分，循环的目的在于配置类中可能继续 `@Import` 配置类，或指定的包中扫描出配置类，或直接给定了 `@Bean` 配置类，需要解析完成所有的配置类
    1. 调用解析器的 `#parse` 方法解析配置类bean定义。（*关于如何进行配置类bean定义解析，我们另开文章来说明。*）
    2. 校验解析完成的bean，校验方法为：获取 `@Configuration` 属性值，如果属性不为空，且指定了 `proxyBeanMethods=true`，则判断类是否是final的，如果是，则直接抛出 `BeanDefinitionParsingException` 异常；然后逐个判断配置类中的@Bean方法，也就是判断方法不能使用 `static`, `final` 或 `private` 进行修饰。
    3. 从解析完毕的Bean定义中移除已经解析完成的配置类。这个步骤的含义在于配置类加载的对象中可能对于已解析的配置类进行了重新定义扫描。
    4. 创建配置类bean定义读取器 `ConfigurationClassBeanDefinitionReader`
    5. 使用读取器加载扫描出来的Bean定义，该步骤会完成判断bean是不是通过@Import导入进来的；遍历Bean中的@Bean注解过的方法；从Bean的引用资源中加载Bean定义；从Bean的导入bean定义注册中加载Bean定义（*关于如何进行扫描的过程，我们另开文章来说明*）
    6. 将解析完毕的Bean定义保存到已经完成解析的对象集合中
    7. 清除候选对象集合
    8. 如果bean工厂中已注册的bean定义的数量大于候选bean名称集合的数量，此处的候选bean名称集合来源于方法开始时，从bean工厂的bean定义名称集合中克隆出来的，所以仍然是扫描之前的数据。通过重新遍历新扫描出来的类，判断是不是配置类并且是不是已经被处理完毕，来决定是否将其加入到候选配置类对象集合中，继续进行解析和处理。
9. 将 `ImportRegistry` 注册为bean，以便支持支持 `ImportAware` `@Configuration` 类，来源于解析器的 `ImportRegistry` 属性
10. 清除外部提供的 `MetadataReaderFactory` 中的缓存；对于共享缓存，这里不会操作，因为它将被 `ApplicationContext` 清除。