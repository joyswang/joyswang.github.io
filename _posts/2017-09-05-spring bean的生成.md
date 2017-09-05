---
layout: post
title: spring源码-spring初始化时bean的生成
date: 2017-09-05 16:21:00
categories:
- java
- spring
tags:
- spring
- java
- 源码
---

在spring中，我们可以通过xml或注解的方式去配置一个bean，配置好后，在spring启动的时候，spring会根据配置的属性去实例化这个bean，有可能是单例的，也有可能是多例的。
这里不会对spring如何收集bean属性做过多的说明，前面有一篇博文已经对xml的bean属性解析做过说明了，这里主要想要聊得还是spring在启动的时候，是如何生成一个个的bean对象的。

### getBean方法

通过跟踪spring的源码，会发现bean对象都是通过```AbstractBeanFactory.getBean(参数)```方法来生成的。
```AbstractBeanFactory.getBean(参数)```方法实际上会去调用```doGetBean(参数)```

### doGetBean方法

```java

    protected <T> T doGetBean(String name, Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
	 //获取实际的beanName，去除beanFactory的&，根据alias获取实际的beanId
         final String beanName = this.transformedBeanName(name);
	 //获取单例对象
         Object sharedInstance = this.getSingleton(beanName);
         Object bean;
         if (sharedInstance != null && args == null) {
             if (this.logger.isDebugEnabled()) {
                 if (this.isSingletonCurrentlyInCreation(beanName)) {
                     this.logger.debug("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
                 } else {
                     this.logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                 }
             }
	    //当前类是factoryBean对象，则通过其生成对象
	    //如果是普通对象，则直接返回
            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
	    //判断多例类型的对象是否正在创建，显然可以看出，非单例如果有循环依赖的话，会直接抛出异常
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }
	    //获取父的beanFactory，如果存在，而且当前没有beanName的定义，则从父的beanFactory中获取对象
            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                String nameToLookup = this.originalBeanName(name);
                if (args != null) {
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
	    
	    //标记bean的创建
            if (!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
	        //获取beanDefinition
                final RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
		//检查当前要创建对象是否是abstract
                this.checkMergedBeanDefinition(mbd, beanName, args);
		//获取depend-on依赖
                String[] dependsOn = mbd.getDependsOn();
                String[] var11;
		//为depend-on依赖创建对象，递归调用getBean
                if (dependsOn != null) {
                    var11 = dependsOn;
                    int var12 = dependsOn.length;

                    for(int var13 = 0; var13 < var12; ++var13) {
                        String dep = var11[var13];
                        if (this.isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }

                        this.registerDependentBean(dep, beanName);
                        this.getBean(dep);
                    }
                }
		//单例的方式创建对象
                if (mbd.isSingleton()) {
                    sharedInstance = this.getSingleton(beanName, new ObjectFactory<Object>() {
                        public Object getObject() throws BeansException {
                            try {
                                return AbstractBeanFactory.this.createBean(beanName, mbd, args);
                            } catch (BeansException var2) {
                                AbstractBeanFactory.this.destroySingleton(beanName);
                                throw var2;
                            }
                        }
                    });
                    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                } else if (mbd.isPrototype()) {
		//多例的方式创建对象
                    var11 = null;

                    Object prototypeInstance;
                    try {
                        this.beforePrototypeCreation(beanName);
                        prototypeInstance = this.createBean(beanName, mbd, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                } else {
		    //其他方式创建对象
                    String scopeName = mbd.getScope();
                    Scope scope = (Scope)this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }

                    try {
                        Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                            public Object getObject() throws BeansException {
                                AbstractBeanFactory.this.beforePrototypeCreation(beanName);

                                Object var1;
                                try {
                                    var1 = AbstractBeanFactory.this.createBean(beanName, mbd, args);
                                } finally {
                                    AbstractBeanFactory.this.afterPrototypeCreation(beanName);
                                }

                                return var1;
                            }
                        });
                        bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    } catch (IllegalStateException var21) {
                        throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var21);
                    }
                }
            } catch (BeansException var23) {
                this.cleanupAfterBeanCreationFailure(beanName);
                throw var23;
            }
        }

        if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
            try {
                return this.getTypeConverter().convertIfNecessary(bean, requiredType);
            } catch (TypeMismatchException var22) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var22);
                }

                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        } else {
            return bean;
        }
    }

```

1. 根据传入的beanName获取实际的name
	* factoryBean如果是以&开头，则去除它
	* 如果传入的是一个alias，则会返回其实际的beanId
2. 根据beanName获取单例对象（此对象有可能是提早暴露出来的对象），如果对象存在，则返回它。
3. 会判断当前对象是否正以非单例的方式创建，这里说明如果有循环依赖存在，且循环依赖的对象还是多例的，则会抛出异常
4. 获取父beanFactory
5. 标记对象正在创建
6. 根据对象scope进行对象的实际创建工作

#### transformedBeanName(name)

