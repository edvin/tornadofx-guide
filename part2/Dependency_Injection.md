# Dependency Injection

`View` and `Controller` are singletons, so you need some way to access the instance of a specific component. Tornado FX supports dependency injection, but you can also lookup components with the `find` function.

```kotlin
val myController = find(MyController::class)
```

When you call `find`, the component corresponding to the given class is looked up in a global component registry. If it did not exist prior to the call, it will be created and inserted into the registry before the function returns.

If you want to declare the controller referance as a field member however, you should use the `inject` delegate instead. This is a lazy mechanism, so the actual instance will only be created the first time you call a function on the injected resource. Using `inject` is always prefered, as it allows your components to have circular dependencies.

```kotlin
val myController: MyController by inject()
```

## Third party injection frameworks

TornadoFX makes it easy to inject resources from a third party dependency injection framework, like for example Guice or Spring. All you have to do is implement the very simple `DIContainer` interface when you start your application. Let's say you have a Guice module configured with a fictive `HelloService`. Start Guice in the `init` block of your `App` class and register the module with TornadoFX:

```kotlin
val guice = Guice.createInjector(MyModule())

FX.dicontainer = object : DIContainer {
    override fun <T : Any> getInstance(type: KClass<T>)
        = guice.getInstance(type.java)
}
```
> The DIContainer implementation is configured to delegate lookups to `guice.getInstance`

To inject the `HelloService` configured in `MyModule`, use the `di` delegate instead of the `inject` delegate:

```kotlin
val MyView : View() {
    val helloService: HelloService by di()
}
```

The `di` delegate accepts any bean type, while `inject` will only allow beans of type `ScopedInstance`, which includes TornadoFX's `View` and `Controller`. This keeps a clean separation between your UI beans and any beans configured in the external dependency injection framework.

## Setting up for Spring

Above the setup for Guice is shown. Setting up for Spring, in this case using `beans.xml` as `ApplicationContext` is done as follows:

### beans.xml

```xml
<?xml version = "1.0" encoding = "UTF-8"?>

<beans xmlns = "http://www.springframework.org/schema/beans"
       xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context = "http://www.springframework.org/schema/context"
       xsi:schemaLocation = "http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
   http://www.springframework.org/schema/context
   http://www.springframework.org/schema/context/spring-context-3.0.xsd">

    <context:component-scan base-package="no.tornadofx.fxsample.springexample"/>
    <context:annotation-config/>
</beans>
```
This sets Spring up to scan for beans.

### Application startup
```kotlin
class SpringExampleApp : App(SpringExampleView::class) {
    init {
        val springContext = ClassPathXmlApplicationContext("beans.xml")
        FX.dicontainer = object : DIContainer {
            override fun <T : Any> getInstance(type: KClass<T>): T = springContext.getBean(type.java)
        }
    }
}
```
This initialized the spring context and hooks it into tornadoFX via the `FX.dicontainer`. Now you can inject Spring beans like this:
```kotlin
val helloBean : HelloBean by di()
```

It is quite common in the Spring world to name a bean like so:

```xml
<bean id = "helloWorld" class = "com.tutorialspoint.HelloWorld">
     <property name = "message" value = "Hello World!"/>
</bean>
```
The bean is then accessible using the `id`. This can be done in tornadoFX too:

```kotlin
class SpringExampleApp : App(SpringExampleView::class) {
    init {
        val springContext = ClassPathXmlApplicationContext("beans.xml")
        FX.dicontainer = object : DIContainer {
            override fun <T : Any> getInstance(type: KClass<T>): T = springContext.getBean(type.java)
	        override fun <T : Any> getInstance(type: KClass<T>, name: String): T = springContext.getBean(type.java,name)
        }
    }
}
```
The second `getInstance` uses both the type of the bean and the id of the bean. Instantiating a bean is down as:

```kotlin
val helloBean : HelloBean by di("helloWorld")
```
