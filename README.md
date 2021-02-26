# StaticContextAccessor
after creating Component class `StaticContextAccessor` in your Spring Project, you can access inistantioate the been anywhere whithout worying about the null point exception.

```java
UserRepository userRepository = StaticContextAccessor.getBean(UserRepository.class);
```

# Statically spilling your (Spring) Beans
## Accessing Spring Beans from a static method

There are some edge cases where you want to access Spring Beans in a static method. While you should always try to refactor your code so you do not have to do this… this post will still show you how it can be done. Continue reading at your own risk! You have been warned!

Autowiring Beans
Getting instances of beans in Spring is pretty simple. You add @Autowired on your constructor and assuming your class is a bean as well, an instance will be injected in your constructor.
```
@Component
public class UserServiceImpl implements UserService {

    UserRepository repo;

    @Autowired
    public UserServiceImpl(UserRepository repo) {
        this.repo = repo; //JEEJ! an instance!
    }
```
Some notes about the above code example before we continue:

@Autowired can also be placed on a field or a setX(x) method. Constructor injection is generally accepted as the best way to inject dependencies;
When a class only has a single constructor you can omit the @Autowired annotation (since Spring 4.3), I kept it here for clarity;
Time to spill some beans! Statically!
Time to spill some beans! Statically!

## The problem
In our use-case, we don’t want to create a class. We want to be able to have static access to the beans! This means the standard @Autowire is not an option. There is however another way to get beans, namely directly from the Application Context.

The Application Context is at the heart of the Spring Framework. It provides a couple of different features:

Load file resources in a generic fashion;
Publish events to registered listeners;
Resolve messages, supporting internationalization;
It has bean factory methods for accessing application components;
That last one is what we are interested in. We want to use the Application Context to get another bean. The ApplicationContext interface provides a nice convenient method to access beans it knows about: getBean(Class<>)?
```
// Our static method
public static <T> T getBean(Class<T> clazz) {
    ApplicationContext context = //excluded
    return context.getBean(clazz);
}
```
So all we need now is to get an ApplicationContext and we are done! How do we get one? Well, you could @Autowire one… but we want to use it in a static context. Here is how we can do it!
```
@Component
public class StaticContextAccessor {

    private static ApplicationContext context;

    @Autowired
    public StaticContextAccessor(ApplicationContext applicationContext) {
        context = applicationContext;
    }

    public static <T> T getBean(Class<T> clazz) {
        return context.getBean(clazz);
    }
}
```
Create a class marked with `@Component.` This way, Spring will initialize it on startup by default.
`@Autowire the ApplicationContext.`
Set the ApplicationContext to a static field.
static method can now use the ApplicationContext.
Doing this, we can now use the getBean() method on the StaticContextAccessor to get any bean in a static context.
```
UserRepository userRepo = StaticContextAccessor.getBean(UserRespository.class)
```
ApplicationContext Initialized -> Get Bean Statically -> Call method on bean

## Timing and the looming Nullpointer
While the solution above works, there is a major problem with it. It is possible to use the StaticContextAccessor.getBean(class) method before the ApplicationContext is autowired. This would crash the system because you can’t invoke a method on a reference pointing to null.

Get Bean Statically -> ApplicationContext Initialized
"This goes boom!"

We have a timing issue here. We know the ApplicationContext will be autowired eventually, usually within seconds if not milliseconds after the application is started. But we also can’t really stop anyone from invoking the static method to get the bean before that happens.

Returning a Proxy
To avoid the timing issue, we need to bridge the time between those two events. What we could do is provide a temporary object, so our static getBean() returns at least something. This can be achieved by using a Proxy object, which is a part of the Java specification.

A proxy class is a class created at runtime that implements a specified list of interfaces, known as proxy interfaces. A proxy instance is an instance of a proxy class. Each proxy instance has an associated invocation handler object, which implements the interface InvocationHandler.

A method invocation on a proxy instance through one of its proxy interfaces will be dispatched to the invoke method of the instance’s invocation handler, passing the proxy instance, a java.lang.reflect.Method object identifying the method that was invoked, and an array of type Object containing the arguments.

Source: https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/lang/reflect/Proxy.html

So we need a couple of things now. Let’s start by seeing how we create the proxy object.
```
public static <T> T getBean(Class<T> clazz) {
    if (context == null) {
        return getProxy(clazz);
    }
    return context.getBean(clazz);
}

private static <T> T getProxy(Class<T> clazz) {
    // Our custom invocation handler, will be explained below!
    DynamicInvocationhandler<T> invocationhandler = new DynamicInvocationhandler<>();
    return (T) Proxy.newProxyInstance(
            clazz.getClassLoader(),
            new Class[]{clazz},
            invocationhandler
    );
}
```
The Proxy.newProxyInstance() method will return a dynamic object of the given interface. It creates a new Object, at runtime, which extends Proxy and implements the given interface. Since this object implements the given interface, we can return it in our getBean() method. Thanks Polymorphism!

Note: This method will only work when requesting a interface. If you want to be able to create a proxy-object for non-interfaces, you could use ByteBuddy. ByteBuddy is out of the scope of this post.

Next up is the InvocationHandler. Whenever a method is invoked on our proxy object, the invoke method of this handler will be called. If your handler has the actual bean set, it will invoke the correct method on our bean instance. If the actual bean isn’t set yet, it will throw a RuntimeException().
```
class DynamicInvocationhandler<T> implements InvocationHandler {

    private T actualBean;

    public void setActualBean(T actual) {
        this.actualBean = actual;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (actualBean == null) {
            throw new RuntimeException("Not initialized yet! :(");
        }
        return method.invoke(actualBean, args);
    }
}
```
Now all that is left is to set the actual bean on the handler as soon as it becomes available and we are done!


# Full Code Solution
```
@Component
public class StaticContextAccessor {

    private static final Map<Class, DynamicInvocationhandler> classHandlers = new HashMap<>();
    private static ApplicationContext context;

    @Autowired
    public StaticContextAccessor(ApplicationContext applicationContext) {
        context = applicationContext;
    }

    public static <T> T getBean(Class<T> clazz) {
        if (context == null) {
            return getProxy(clazz);
        }
        return context.getBean(clazz);
    }

    private static <T> T getProxy(Class<T> clazz) {
        DynamicInvocationhandler<T> invocationhandler = new DynamicInvocationhandler<>();
        classHandlers.put(clazz, invocationhandler);
        return (T) Proxy.newProxyInstance(
                clazz.getClassLoader(),
                new Class[]{clazz},
                invocationhandler
        );
    }

    //Use the context to get the actual beans and feed them to the invocationhandlers
    @PostConstruct
    private void init() {
        classHandlers.forEach((class, invocationHandler) -> {
            Object bean = context.getBean(class);
            invocationHandler.setActualBean(bean);
        });
    }

    static class DynamicInvocationhandler<T> implements InvocationHandler {

        private T actualBean;

        public void setActualBean(T actualBean) {
            this.actualBean = actualBean;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (actualBean == null) {
                throw new RuntimeException("Not initialized yet! :(");
            }
            return method.invoke(actual, args);
        }
    }
}
```
There we go! We can now get Spring components from a static context! All we need to do is call the static getBean method.

UserRepository userRepo = StaticContextAccessor.getBean(UserRespository.class)
About that RuntimeException
Even with this proxying setup, there is still a case which will not work. As you can see in the handler code above, if the actualBean isn’t set yet, we throw a RuntimeException.
```
UserRepository userRepo = StaticContextAccessor.getBean(UserRespository.class)
```
userRepo.getAll() //this will break if called before ApplicationContext is ready.
There is no way around this. If the bean isn’t created and known in the ApplicationContext, we can’t call a method on it. The proxy solves the issue with the time between requesting the Bean and the ApplicationContext being created. However, if the real bean isn’t loaded into the proxy and a method is invoked, the only option is to throw an exception.
