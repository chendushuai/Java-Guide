# Spring初始化时注册默认注解配置处理器

本文主要讲述默认bean后置处理器的注入时机，如用于处理配置类的配置类注解后置处理器

## 一、 默认测试代码提供

本账号所有Spring文章，如为特殊说明，均使用如下测试入口代码

```java
public class Test {
	public static void main(String[] args) {
		// C01 初始化注解配置上下文对象
		AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext();
		// C02 注册配置类
		annotationConfigApplicationContext.register(Appconfig.class);
		// C02_1 添加自定义bean工厂后置处理器
		annotationConfigApplicationContext.addBeanFactoryPostProcessor(new ChenssRegisterBeanFactoryPostProcessor());
		// C03 刷新上下文对象
		annotationConfigApplicationContext.refresh();
	}
}
```

## 二、 初始化注解bean定义读取器时注册默认的注解配置处理器

1. 在 `AnnotationConfigApplicationContext` 的无参构造方法中，会初始化用于读取注解bean定义的读取器 `AnnotatedBeanDefinitionReader` ，该读取器主要用于读取使用注解标注的bean定义
2. 在 `AnnotatedBeanDefinitionReader` 的构造方法中，会根据跟定的bean定义注册器生成一个读取器，此处给定的注册器实际为 `AnnotationConfigApplicationContext` ，这也表明上下文对象其实是 `BeanDefinitionRegistry` 的子类。
3. 在读取器的构造方法 `AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment)` 中会调用 `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);` 注册注解配置处理器。

## 三、 `registerAnnotationConfigProcessors` 方法说明

1. 获取默认bean工厂 `DefaultListableBeanFactory` ，如果给定注册器实现自 `DefaultListableBeanFactory` ，则直接返回自身；如果给定注册器实现自 `GenericApplicationContext` ，则返回对象的 `DefaultListableBeanFactory` 属性；否则返回空
2. 如果bean工厂不为空，判断是否指定依赖排序，如未指定则指定默认排序规则；判断bean工厂的自动装配候选解析器是否实现自上下文注解自动装配候选解析器，如果不是，则使用默认解析器 `ContextAnnotationAutowireCandidateResolver` 
3. 生成 `ConfigurationClassPostProcessor` 配置类后置处理器的 `RootBeanDefinition` ，添加到bean工厂的bean定义集合 `beanDefinitionMap` 中。
4. 生成 `AutowiredAnnotationBeanPostProcessor` 自动装配注解Bean后置处理器的 `RootBeanDefinition` ，添加到bean工厂的bean定义集合 `beanDefinitionMap` 中。
5. 如果支持JSR-250，则生成 `CommonAnnotationBeanPostProcessor` 常规注解Bean后置处理器的 `RootBeanDefinition` ，添加到bean工厂的bean定义集合 `beanDefinitionMap` 中。
6. 如果支持JPA，则生成 `org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor` 的 `RootBeanDefinition` ，添加到bean工厂的bean定义集合 `beanDefinitionMap` 中。
7. 生成 `EventListenerMethodProcessor` `@EventListener`注解处理器的后置处理器的 `RootBeanDefinition` ，添加到bean工厂的bean定义集合 `beanDefinitionMap` 中。
8. 生成 `EventListenerMethodProcessor` 添加内部管理的 `EventListenerFactory` 的bean的 `RootBeanDefinition` ，添加到bean工厂的bean定义集合 `beanDefinitionMap` 中。

