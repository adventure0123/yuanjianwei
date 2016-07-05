---
title: springIoc
date: 2016-07-05 22:56:22
tags: SpringIOC
---

Spring IOC是Spring框架的核心功能之一，本文主要介绍Spring IOC的主要概念和自己模拟实现IOC和DI的实现原理.<!--more-->IOC是inversion of control的缩写，中文翻译成控制反转。Java程序中的业务需要多个对象来协同完成，通常每个对象在使用其他对象的时候，自己要使用new object()这样的语法来生成合作的对象，这样你会发现对象之间的耦合程度变高了。而IOC的思想是，由Spring容器来实现这些依赖对象的创建和协调作用，对象只要实现业务逻辑本身就可以了。从这方面来说，对象如何获得协作对象的责任被反转了。
举个通俗的例子，我们找对象的过程是到处找哪里有长的漂亮或者气质好的妹子，然后再拿到他们的QQ，微信，电话号码等，想办法认识他们。这时候你要面对找对象的各个环节，而IOC就像一个婚姻介绍所，所有的妹子都在介绍所里注册，当你需要对象的时候，你只要告诉婚姻介绍所，你需要对象的要求就行了，比如身高，相貌等，婚姻介绍所就会自动返回符合要求的对象，我们只要负责如何和她恋爱就行了，找对象的过程都不再需要我们自己控制。Spring的思想就和这类似，所有的类都会在Spring容器中注册，你需要什么东西，spring都会在系统运行的适当时候将你需要的东西交给你，所有类的创建销毁都由spring来控制。
DI是Martin Fowler在2014年首次提出的，他总结：控制的什么被反转了？就是：获得依赖对象的方式反转了。DI是Dependency Injection的简称，中文叫依赖注入。就是在IOC容器运行期间，动态将某种依赖注入到对象之中。所以依赖注入和控制反转是从不同的角度描述同一件事，就是通过IOC容器，利用依赖注入的方式，实现对象之间的解耦。
下面我们来模拟IOC和DI的实现过程：
首先定义DAO和Service的接口和实现类
```java
package com.yuanjianwei.dao;

/**
 * Created by adventure on 16/7/4.
 */
public interface PersonDao {
    void save();
}


package com.yuanjianwei.dao.impl;

import com.yuanjianwei.dao.PersonDao;

/**
 * Created by adventure on 16/7/4.
 */
public class PersonDaoImpl implements PersonDao {
        public void save() {
            System.out.print("dao---->");
            System.out.println("保存");
        }

    }
```

```java
package com.yuanjianwei.service;

/**
 * Created by adventure on 16/7/4.
 */
public interface PersonService {
    void doSave();
}



package com.yuanjianwei.service.impl;

import com.yuanjianwei.dao.PersonDao;
import com.yuanjianwei.service.PersonService;

/**
 * Created by adventure on 16/7/4.
 */
public class PersonServiceImpl implements PersonService {
    PersonDao personDao;
    public void doSave() {
        System.out.print("service ---->");
        personDao.save();
    }

    public PersonDao getPersonDao() {
        return personDao;
    }

    public void setPersonDao(PersonDao personDao) {
        this.personDao = personDao;
    }
}

```

然后自定义MyClassPathXmlApplicationContext，首先读取spring.xml的配置信息，然后通过反射机制实现bean，最后注入所需的bean。
```java
package com.yuanjianwei.common;

import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import java.beans.Introspector;
import java.beans.PropertyDescriptor;
import java.lang.reflect.Method;
import java.net.URL;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Created by adventure on 16/7/4.
 */
public class MyClassPathXmlApplicationContext {

        // xml所有的属性
        private ArrayList<BeanDefinition> beanDefinitions = new ArrayList<BeanDefinition>();
        // xml中所有的bean
        private Map<String, Object> beanMap = new HashMap<String, Object>();

        public MyClassPathXmlApplicationContext(String file) {
            readXml(file);
            instanceBeans();
            instanceObject();
        }

        /**
         * 注入
         */
        private void instanceObject() {
            for (BeanDefinition beanDefinition : beanDefinitions) {
                //判断有没有注入属性
                if (beanDefinition.getProperty() != null) {
                    Object bean = beanMap.get(beanDefinition.getId());
                    if (bean != null) {
                        try {
                            //得到被注入bean的所有的属性
                            PropertyDescriptor[] ps = Introspector.getBeanInfo(bean.getClass()).getPropertyDescriptors();
                            //得到所有的注入bean属性
                            for(PropertyDefinition propertyDefinition:beanDefinition.getProperty()){
                                for(PropertyDescriptor propertyDescriptor:ps){
                                    if(propertyDescriptor.getName().equals(propertyDefinition.getName())){
                                        Method setter = propertyDescriptor.getWriteMethod();//获取set方法
                                        if(setter!=null){
                                            setter.setAccessible(true);//得到private权限
                                            //注入属性
                                            setter.invoke(bean, beanMap.get(propertyDefinition.getRef()));
                                        }
                                        break;
                                    }
                                }
                            }
                        } catch (Exception e) {
                            // TODO Auto-generated catch block
                            e.printStackTrace();
                        }
                    }
                }
            }
        }

        /**
         * 实例所有的bean
         */
        private void instanceBeans() {
            for (BeanDefinition bd:beanDefinitions) {
                try {
                    try {
                        if (bd.getClassName() != null
                                && !bd.getClassName().equals(""))
                            beanMap.put(bd.getId(), Class.forName(
                                    bd.getClassName()).newInstance());
                    } catch (InstantiationException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    } catch (IllegalAccessException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                } catch (ClassNotFoundException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }

        /**
         * 读xml
         *
         * @param file
         */
        private void readXml(String file) {
            try {
                SAXReader reader = new SAXReader(); // 使用SAX方式解析XML
                URL xmlPath = this.getClass().getClassLoader().getResource(file);
                Document doc = reader.read(xmlPath);
                Element root = doc.getRootElement(); // 取得根节点
                List<Element> beans = root.elements();
                for (Element element : beans) {
                    String id = element.attributeValue("id");// id;
                    String clazz = element.attributeValue("class");
                    BeanDefinition bd = new BeanDefinition(id, clazz);
                    // 读取子元素
                    if (element.hasContent()) {
                        List<Element> propertys = element.elements();
                        for (Element property : propertys) {
                            String name = property.attributeValue("name");
                            String ref = property.attributeValue("ref");
                            PropertyDefinition pd = new PropertyDefinition(name,
                                    ref);
                            bd.getProperty().add(pd);
                        }
                    }
                    beanDefinitions.add(bd);
                }
            } catch (Exception e) {
                // TODO: handle exception
            }
        }

        /**
         * 通过名字得到bean
         *
         * @param str
         * @return
         */
        public Object getBean(String str) {
            return beanMap.get(str);
        }



}


package com.yuanjianwei.common;

import java.util.ArrayList;
import java.util.List;

/**
 * Created by adventure on 16/7/4.
 */
public class BeanDefinition {
        private String id;
        private String className;
        private List<PropertyDefinition> property = new ArrayList<PropertyDefinition>();

        public BeanDefinition(String id, String className) {
            super();
            this.id = id;
            this.className = className;
        }

        public String getId() {
            return id;
        }

        public void setId(String id) {
            this.id = id;
        }

        public String getClassName() {
            return className;
        }

        public void setClassName(String className) {
            this.className = className;
        }

        public List<PropertyDefinition> getProperty() {
            return property;
        }

        public void setProperty(List<PropertyDefinition> property) {
            this.property = property;
        }



}

package com.yuanjianwei.common;

/**
 * Created by adventure on 16/7/4.
 */
public class PropertyDefinition {

        private String name;
        private String ref;

        public PropertyDefinition(String name, String ref) {
            this.name = name;
            this.ref = ref;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getRef() {
            return ref;
        }

        public void setRef(String ref) {
            this.ref = ref;
        }

    }

```

spring的配置如下：
```vim
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="personDao" class="com.yuanjianwei.dao.impl.PersonDaoImpl"></bean>
    <bean id="personService" class="com.yuanjianwei.service.impl.PersonServiceImpl">
        <property name="personDao" ref="personDao"></property>
    </bean>
</beans>

```
最后自己写测试类测试
```java
package com.yuan.test;

import com.yuanjianwei.common.MyClassPathXmlApplicationContext;
import com.yuanjianwei.dao.PersonDao;
import com.yuanjianwei.service.PersonService;
import org.junit.Test;

/**
 * Created by adventure on 16/7/4.
 */
public class PersonTest {
    @Test
    public void springIoc(){
        MyClassPathXmlApplicationContext applicationContext=new MyClassPathXmlApplicationContext("spring-config.xml");
        PersonDao personDao= (PersonDao) applicationContext.getBean("personDao");
        personDao.save();
        PersonService personService= (PersonService) applicationContext.getBean("personService");
        personService.doSave();
    }

}

```
实现结果如下
```vim
dao---->保存
service ---->dao---->保存
```
总结：Spring IOC的真正实现要比这复杂许多倍，但是基本的思想是一致的。