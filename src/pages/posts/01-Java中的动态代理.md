---
date: 2025/01/02
---

<img src="https://web-ghw-demo.oss-cn-hangzhou.aliyuncs.com/1.jpg" width="800" />

<small>八阵图名成卧龙，六韬书功在飞熊。

—— 蟾宫曲·怀古</small>

## JDK动态代理
实现jdk代理有以下条件

1. 要代理的类必须实现接口
2. jdk代理是基于对象生成代理

我们通过Proxy类的newProxyInstance()方法生成代理对象，这个方法需要三个参数，分别是

1. 要代理的对象的类加载器

```java
target.getClass().getClassLoader()
```

2. 要代理的对象的类实现的接口

```java
target.getClass().getInterface()
```

3. 要代理的对象的包装类对象，这个包装类要实现InvocationHandler

```java
target的代理类
```

这个方法返回值就是生成的代理类，这个方法拿到了这个对象实现的接口信息，可以声明一个新的类型来实现这个接口中的所有方法，但这些方法中都是调用别的对象的方法实现的，这个对象不能是target本身，不然就是自己调用自己，没有实现方法的加强。他调用的是target的包装类，这个包装类必须实现了InvocationHandler接口，这个接口内部有个invoke()方法，他是调用所有target中的所有方法的入口，在这个方法中我们可以实现对方法的加强。

代码实现

```java
package annotation;

public interface Actor {

    String sing();

    String dance();
}
```

```java
package Impl;

import annotation.Actor;

public class Star implements Actor {
    @Override
    public String sing() {
        System.out.println("is Singing");
        return "Jack is Singing";
    }

    @Override
    public String dance() {
        System.out.println("is Dancing");
        return "Jack is Dancing";
    }
}
```

```java
package Proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class StarProxy implements InvocationHandler {

    private Object object;

    public StarProxy(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("记录日志");
        Object result = method.invoke(object, args);
        return result;
    }
}
```

```java
package Test;

import Impl.Star;
import annotation.Actor;
import Proxy.StarProxy;

import java.lang.reflect.Proxy;

public class JDKProxyTest {

    public static void main(String[] args) {
        Star actor = new Star();
        StarProxy starProxy = new StarProxy(actor);
        Object object = Proxy.newProxyInstance(actor.getClass().getClassLoader(), actor.getClass().getInterfaces(), starProxy);
        Actor star = (Actor) object;
        star.dance();
        star.sing();
    }
}
```

## CGLib动态代理
前面的JDK动态代理是基于接口实现的，要使用JDK动态代理，被代理类必须要实现接口。那如果一个类没有实现接口呢，就只能使用CGLib动态代理了。

cglib动态代理基于类实现代理，通过继承机制，继承类中的所有方法，在进行覆盖时实现加强，从而实现代理。

cglib原理：它生成被代理类的继承类，这个类持有一个MethodInterceptor，我们在setCallback()时把这个参数传入，这个代理类重写代理类中的所有方法，<font style="color:rgb(52, 73, 94);">在代理类中，构建名叫“CGLIB”+“$父类方法名$”的方法（所有非private的方法都会被构建），方法体中只有一句super.方法名调用父类方法。这样代理类中有重写方法，CGLib方法，父类方法，还有一个统一的拦截方法（Intercept）,其中重写方法与CGLIb方法有某种映射关系。这里面的重写方法是外部调用的入口，他会调用统一的拦截方法，传入四个参数，第一个参数传递的是this，代表代理类本身，第二个参数标示拦截的方法，第三个参数是入参，第四个参数是cglib方法，intercept方法完成增强后，我们调用cglib方法间接调用父类方法完成整个方法链的调用。</font>

<font style="color:rgb(52, 73, 94);">代码实现：</font>

```java
package Proxy;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class CGLibProxy implements MethodInterceptor{

    public Object createCGLibProxy(Class<?> clazz){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("记录日志");
        return methodProxy.invokeSuper(o, objects);
    }
}
```

```java
package Test;

import Impl.Star;
import Proxy.CGLibProxy;
import annotation.Actor;

public class CGLibProxyTest {

    public static void main(String[] args) {
        Actor actor = new Star();
        CGLibProxy cgLibProxy = new CGLibProxy();
        Object cgLibProxy1 = cgLibProxy.createCGLibProxy(actor.getClass());
        Actor star = (Actor) cgLibProxy1;
        star.sing();
        star.dance();
    }
}
```

## 总结
JDK动态代理只能针对实现了接口的类，而CGLib动态代理则没有这个限制，但是使用CGLib动态代理时，被代理类最好不要申明为final，因为声明为final的类是无法被继承的

+ Spring在默认情况下是采用JDK动态代理，但可以强制使用CGLib
+ 在类没有实现接口时，只能使用CGLib
+ Spring会根据情况自动切换