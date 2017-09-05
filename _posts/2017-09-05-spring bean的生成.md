---
layout: post
title: springԴ��-spring��ʼ��ʱbean������
date: 2017-09-05 16:21:00
categories:
- java
- spring
tags:
- spring
- java
- Դ��
---

��spring�У����ǿ���ͨ��xml��ע��ķ�ʽȥ����һ��bean�����úú���spring������ʱ��spring��������õ�����ȥʵ�������bean���п����ǵ����ģ�Ҳ�п����Ƕ����ġ�
���ﲻ���spring����ռ�bean�����������˵����ǰ����һƪ�����Ѿ���xml��bean���Խ�������˵���ˣ�������Ҫ��Ҫ�ĵû���spring��������ʱ�����������һ������bean����ġ�

### getBean����

ͨ������spring��Դ�룬�ᷢ��bean������ͨ��```AbstractBeanFactory.getBean(����)```���������ɵġ�
```AbstractBeanFactory.getBean(����)```����ʵ���ϻ�ȥ����```doGetBean(����)```

### doGetBean����

```java

    protected <T> T doGetBean(String name, Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
	 //��ȡʵ�ʵ�beanName��ȥ��beanFactory��&������alias��ȡʵ�ʵ�beanId
         final String beanName = this.transformedBeanName(name);
	 //��ȡ��������
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
	    //��ǰ����factoryBean������ͨ�������ɶ���
	    //�������ͨ������ֱ�ӷ���
            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
	    //�ж϶������͵Ķ����Ƿ����ڴ�������Ȼ���Կ������ǵ��������ѭ�������Ļ�����ֱ���׳��쳣
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }
	    //��ȡ����beanFactory��������ڣ����ҵ�ǰû��beanName�Ķ��壬��Ӹ���beanFactory�л�ȡ����
            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                String nameToLookup = this.originalBeanName(name);
                if (args != null) {
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
	    
	    //���bean�Ĵ���
            if (!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
	        //��ȡbeanDefinition
                final RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
		//��鵱ǰҪ���������Ƿ���abstract
                this.checkMergedBeanDefinition(mbd, beanName, args);
		//��ȡdepend-on����
                String[] dependsOn = mbd.getDependsOn();
                String[] var11;
		//Ϊdepend-on�����������󣬵ݹ����getBean
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
		//�����ķ�ʽ��������
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
		//�����ķ�ʽ��������
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
		    //������ʽ��������
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

1. ���ݴ����beanName��ȡʵ�ʵ�name
	* factoryBean�������&��ͷ����ȥ����
	* ����������һ��alias����᷵����ʵ�ʵ�beanId
2. ����beanName��ȡ�������󣨴˶����п��������籩¶�����Ķ��󣩣����������ڣ��򷵻�����
3. ���жϵ�ǰ�����Ƿ����Էǵ����ķ�ʽ����������˵�������ѭ���������ڣ���ѭ�������Ķ����Ƕ����ģ�����׳��쳣
4. ��ȡ��beanFactory
5. ��Ƕ������ڴ���
6. ���ݶ���scope���ж����ʵ�ʴ�������

#### transformedBeanName(name)

