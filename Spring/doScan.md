# 配置类后置处理器解析配置类的过程

标签（空格分隔）： Spring

---

本文将继续跟着前文中的配置类判断进行，方法入口位于 `org.springframework.context.annotation.ConfigurationClassParser#parse(java.util.Set<org.springframework.beans.factory.config.BeanDefinitionHolder>)` ，在这里，已注册bean定义中的配置类bean定义已经被筛选到候选对象集合中，逐个进行遍历解析。

1. 如果bean实现自 `AnnotatedBeanDefinition` 接口，则进行注解bean定义的解析流程，详见下面章节中的各步骤说明。
2. 如果bean实现自 `AbstractBeanDefinition` 接口，且指定了beanClass 属性，则进行抽象bean定义的解析；
3. 否则作为常规配置类进行解析
4. 逐个遍历解析完毕配置类之后，判断是否需要延迟导入，如果存在需要延迟导入的处理器，直接进行处理。

## 1. 注解bean定义 `AnnotatedBeanDefinition` 的解析流程

实际调用到 `ConfigurationClassParser#processConfigurationClass` 方法进行配置类解析，这里的参数是将元数据和bean名称进行封装成为一个整体
1. 首先判断是否指定了 `@Conditional` 注解及子注解，需要跳过解析环节	
2. 判断给定的bean类是否在给定的类集合中
    1. 如果已经存在，则判断bean类是否是通过 `@Import` 导入的，且已存在的bean类也是通过 `@Import` 导入的，则直接将当前判断的bean类合并到已经存在的bean类中，直接返回。
    2. 如果已经存在，且当前判断的bean类不是通过 `@Import` 导入的，则从给定的类集合中删除。
3. 根据bean定义中的类，解析得到源类 `SourceClass`
4. 调用方法 `ConfigurationClassParser#doProcessConfigurationClass` 执行处理配置类的动作，并最终返回父类，递归进行处理，直到最顶级的父类
    1. 判断配置类是否添加了 `@Component` 注解，如果添加了，则还需要解析成员类，也就是嵌套类，这里我们先不讨论
    2. 如果配置类添加了 `@PropertySources` 注解，并且配置类解析器的环境变量实现了 `ConfigurableEnvironment` 接口，则调用 `ConfigurationClassParser#processPropertySource` 方法处理 `@PropertySources` 注解，详细解析过程见下面章节

## `ConfigurationClassParser#processPropertySource` 方法处理 `@PropertySources` 注解的过程

1. 获取 `@PropertySources` 注解中的 `name` 属性值，如果未指定，则参数值为 `null`
2. 获取 `@PropertySources` 注解中的 `encoding` 属性值，如果未指定，则参数值为 `null`
3. 获取 `@PropertySources` 注解中的 `value` 属性值，解析为位置数组，如果未指定，则直接判定为失败，必须要指定 `value` 属性
4. 获取 `@PropertySources` 注解中的 `ignoreResourceNotFound` 属性值，默认为false，判断是否忽略找不到资源的错误。
5. 获取 `@PropertySources` 注解中的 `factory` 属性值，作为属性源工厂 `PropertySourceFactory` 的类名，默认为 `PropertySourceFactory.class`。根据属性值创建属性源工厂实例对象，值为 `PropertySourceFactory.class` 时，工厂为 `DefaultPropertySourceFactory` 实例。
6. 遍历上面第3步得到的位置数组，将位置解析成为绝对地址，并加载可用资源，进行解码后，使用工厂实例创建属性源对象，添加到属性源中
7. 获取 `@ComponentScan` 注解的属性值，如果指定了该注解，且在注册bean `REGISTER_BEAN` 时不需要跳过，则遍历属性值
    1. 使用组件扫描解析器扫描给定的包名路径，如何完成扫描请查看下面的章节
    2. 遍历扫描出来的bean定义集合，如果bean定义存在源定义则使用源定义进行进一步判断，判断bean定义是否是配置类，如果是配置类，则继续调用解析配置类方法进行递归解析
13. 调用方法 `ConfigurationClassParser#processImports` 处理 `@Import` 注解（*详细的处理过程，将另开文章进行说明*）
14. 获取 `@ImportResource` 注解，读取 `location` ，替换 `location` 中的参数值，读取 `reader` 属性值，解析为bean定义读取器类，保存对应的映射关系到配置类的导入资源集合中
15. 解析配置类，得到配置类中使用 `@Bean` 注解的方法集合，将获取到的方法元数据创建为 `BeanMethod` ，保存到配置类的 `BeanMethod` 属性中（*具体的注解方法扫描过程，我们另开文章说明*）
16. 递归遍历处理接口上的默认 `@Bean` 方法
17. 递归处理超类，如果超类不为空，且不是 `java` 开头，则添加到已知超类集合中，返回超类，继续递归遍历处理
18. 返回null
    
## `ComponentScanAnnotationParser#parse` 解析配置对象，并扫描指定包下bean对象的流程

1. 创建扫描器 `ClassPathBeanDefinitionScanner`，这也可以证明在声明上下文时声明的扫描器是给程序员用的，Spring自己使用的时候会重新创建一个新的扫描器，这个扫描器在创建的时候会判断是否使用默认的过滤器。
2. 获取指定的Bean名称生成器，如果给定的生成器类型为 `BeanNameGenerator` ，认为使用的是内部默认的名称生成器，否则使用给定的名称生成器类，创建对应生成器实例，并绑定到扫描器上。
3. 判断指定的范围代理选项 `scopedProxy` ，指定是使用JDK代理还是CGLIB代理，或是不使用代理，并保存使用代理的选项。如果不使用范围代理，需要获取范围解析选项 `scopeResolver` ，并实例化解析器对象绑定到扫描器上。
4. 将设置的资源模式 `resourcePattern` 绑定到扫描器上
5. 将设置的包含类型 `includeFilters` 和排除类型 `excludeFilters` 绑定到扫描器上
6. 将设置的延迟加载选项 `lazyInit` 设置在扫描器中的bean定义的默认值上
7. 获取设置的扫描包名属性 `basePackages` ，替换其中的环境变量值，并将值转换为数组，多个包名之间可以使用 `,; \t\n` 进行分隔，将获得的包名集合保存到包地址集合中
8. 获取指定的包类名属性值 `basePackageClasses` ，获取其包名保存在包地址集合中
9. 如果最终包地址集合仍然没有值，则默认添加当前配置类所在的包名
10. 添加排除过滤器，用于排除扫描结果中类名跟配置类类名一直的结果，避免配置类重复加载
11. 调用 `ClassPathBeanDefinitionScanner#doScan` 执行实际包扫描的动作。（*实际扫描包的流程，见下文说明*）
12. 返回扫描出来的bean定义集合。

## `ClassPathBeanDefinitionScanner#doScan` 执行实际包扫描的流程

1. 声明bean定义集合
2. 遍历包名数组，扫描包下符合条件的候选Bean定义集合，遍历候选Bean定义集合（*具体的扫描过程我们另开文章再来说明*）
    1. 使用范围源数据解析器判断是否添加了 `@Scope` 注解，如果设置了该注解，以该bean定义中设置的范围和代理类型为准，否则使用默认的单例范围。并将解析出来的范围信息保存在bean定义中
    2. 使用bean名称生成器生成Bean名称，常规做法是直接根据类名，将首字母小写后作为bean名称，内部类名称中会产生 `.`。
    3. 如果bean定义实现了 `AbstractBeanDefinition` 接口，需要进一步处理bean定义：
        - 设置bean定义默认值，是否延迟加载、自动装配模式、依赖检查、初始化方法名称、销毁方法名称为默认值，且说明指定的初始化方法和销毁方法均不是默认的
        - 如果指定了自动装配候选规则，则使用规则同名称进行匹配，如果匹配成功，则认为该bean作为自动装配候选，否则不可以作为自动装配候选。
    4. 如果bean定义实现了 `AnnotatedBeanDefinition` 接口，则处理bean定义中的基本注解，设置到bean定义中。具体有：
        - `@Lazy` 注解，指定是否需要延迟初始化
        - `@Primary` 注解，指定当多个bean满足自动装配的条件时，当前bean是否作为优先选择对象
        - `@DependsOn` 注解，指定当前bean的依赖对象，需要在依赖bean初始化之后初始化，销毁之后销毁
        - `@Role` 注解，指定角色类型
        - `@Description` 注解，指定说明内容
    5. 检查扫描出来的bean定义同现有的已经扫描出来的bean定义是否冲突，如果返回false，则直接跳过该扫描出来的bean，否则创建一个新的bean定义并添加到bean定义集合中。（*添加方法为 `BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry)` ，具体的执行逻辑将另开文章说明*）
        - 如果bean定义集合中不存在给定名称的bean定义，则直接认为不冲突
        - 得到已经存在的bean定义，判断是否存在源bean定义，如果存在源，则使用源来进行比较
        - 比较已经存在bean定义跟新扫描出来的bean定义是否兼容，比较的内容有：原有bean定义没有实现 `ScannedGenericBeanDefinition` 接口，也就说原有bean不是扫描得到的；扫描文件中的文件源不为空，且文件源同原有文件的文件源一致，也就是说同一个源文件被扫描出来两次；原有bean定义和扫描出来的bean定义对象一致，也就是等价类被扫描两次。满足上面条件的认为是兼容的，直接返回false，不继续处理扫描出来的这个对象
        - 否则抛出 `ConflictingBeanDefinitionException` 异常，存在冲突的bean定义。
3. 返回处理完毕的bean定义集合