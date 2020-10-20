# Scopes

Scope is a simple construct that enables some interesting and helpful behavior in a TornadoFX application. A Scope can be viewed as the "context" with which the parent singleton `Component` and any possible children `Component`s that may exist in the same context. Within that context, it is easy to pass around the subset of instances from one `Component` to another.

![alttext](https://github.com/ahinchman1/TornadoFX-DnD-TilesFX/blob/master/Scopes.png)

When you use `inject()` or `find()` to locate a `Controller` or a `View`, you will, by default, get back a singleton instance, meaning that wherever you locate that object in your code, you will get back the same instance. Scopes provide a way to make a `View` or `Controller` unique to a smaller subset of instances in your application.

Each `Component`, like `View`, `Fragment` and `Controller`, inherit whatever scope they were looked up in, so you normally don't need to mention the scope after looking up the "root" of your tree of elements.

It can also be used to run multiple versions of the same application inside the same JVM, for example with [JPro](https://www.jpro.one/), which exposes TornadoFX application in a web browser.

## A Master/Detail example

In an [MDI Application](https://en.wikipedia.org/wiki/Multiple_document_interface) you can open an editor in a new window, and ensure that all the injected resources are unique to that window. We will leverage that technique to create a person editor that allows you to open a new window to edit each person.

We start by defining a table interface where you can double click to open the person editor in a separate window.

```kotlin
class PersonList : View("Person List") {
    val ctrl: PersonController by inject()

    override val root = tableview<Person>() {
        column("#", Person::idProperty)
        column("Name", Person::nameProperty)
        onUserSelect { editPerson(it) }
        asyncItems { ctrl.people() }
    }

    fun editPerson(person: Person) {
        val editScope = Scope()
        val model = PersonModel()
        model.item = person
        setInScope(model, editScope)
        find(PersonEditor::class, editScope).openWindow()
    }
}
```

The `edit` function creates a new `Scope` and injects a `PersonModel` configured with the selected user into that scope. Finally, it retrieves a `PersonEditor` in the context of the new scope and opens a new window. `find` allows for the ability to pass in scopes as a parameter easily between classes, so be sure not to forget this step!  TornadoFX gives more insight on the ability for passing scopes in new instances of components:

```kotlin
fun <T: Component> find(componentType: Class<T>, scope: Scope = FX.defaultScope): T = 
     inline fun <reified T: Component> find(scope: Scope = FX.defaultScope): T = 
         find(T::class, scope)
```

When the `PersonEditor` is initialized, it will look up a `PersonModel` via injection. The default context for `inject` and `find` is always the scope that created the component, so it will look in the `personScope` we just created.

```kotlin
val model: PersonModel by inject()
```

## Breaking Out of the Current Scope

When no scope is defined, injectable resources are looked up in the default scope. There is an item representing that scope called `FX.defaultScope`. In the above example, the editor might have called out to a `PersonController` to perform a save operation in a database or via a REST call. This `PersonController` is most probably stateless, so there is no need to create a separate controller for each edit window. To access the same controller in all editor windows, we supply the scope we want to find the controller in:

```kotlin
val controller: PersonController by inject(FX.defaultScope)
```

This effectively makes the `PersonController` a true singleton object again, with only a single instance in the whole application.

The default scope for new injected objects are always the current scope for the component that calls `inject` or `find`, and consequently all objects created in that injection run will belong to the supplied scope.

## Keeping State in Scopes

In the previous example we used injection on a scope level to get a hold of our resources. It is also possible to subclass `Scope` and put arbitrary data in there. Each TornadoFX `Component` has a `scope` property that gives you access to that scope instance. You can even override it to provide the custom subclass so you don't need to cast it on every occasion:

```kotlin
override val scope = super.scope as PersonScope
```

Now whenever you access the `scope` property from your code, it will be of type `PersonScope`. It now contains a `PersonModel` that will only be available to this scope:

```kotlin
class PersonScope : Scope() {
    val model = PersonModel()
}
```

Let's change our previous example slightly to access the model inside the scope instead of using injection. First we change the editPerson function:

```kotlin
fun editPerson(person: Person) {
    val editScope = PersonScope()
    editScope.model.item = person
    find(PersonEditor::class, editScope).openWindow()
}
```

The custom scope already has an instance of `PersonModel`, so we just configure the item for that scope and open the editor. Now the editor can override the type of scope and access the model:

```kotlin
// Cast scope
override val scope = super.scope as PersonScope
// Extract our view model from the scope
val model = scope.model
```

Both approaches work equally well, but depending on your use case you might prefer one over the other.

## Global application scope

As we hinted to initially, you can run multiple applications in the same JVM and keep them completely separate by using scopes. By default, JavaFX does not support multi tenancy, and can only start a single JavaFX application per JVM, but new technologies are emerging that leverages multitenancy and will even expose your JavaFX based applications to the web. One such technology is JPro.one, and TornadoFX supports multitenancy for JPro applications by leveraging scopes.

There is no special JPro classes in TornadoFX, but supporting JPro is very simple by leveranging scopes:

## Using TornadoFX with JPro

JPro will create a new instance of your App class for each new web user. Also, to access the JPro WebAPI you need to get access to the stage created for each user. In this example we subclass `Scope` to create a special JProScope that contains the stage that was given to each application instance:

```kotlin
class JProScope(val stage: Stage) : Scope() {
    val webAPI: WebAPI get() = WebAPI.getWebAPI(stage)
}
```

The next step is to subclass `JProApplication` to define our entry point. This app class is in addition to our existing TornadoFX App class, which boots the actual application:

```
class Main : JProApplication() {
    val app = OurTornadoFXApp()

    override fun start(primaryStage: Stage) {
        app.scope = JProScope(primaryStage)
        app.start(primaryStage)
    }

    override fun stop() {
        app.stop()
        super.stop()
    }
}
```

Whenever a new user visits our site, the `Main` class is created, together with a new instance of our actual TornadoFX application.

In the `start` function we assign a new `JProScope` to the TornadoFX app instance and then call `app.start`. From there on out, all instances created using `inject` and `find` will be in the context of that JPro instance.

As usual, you can break out of the `JProScope` to access JVM level globals by supplying the `DefaultScope` or any other shared scope to the `inject` or `find` functions.

We should provide a utility function that makes it easy to access the JPro WebAPI from any Component:

```kotlin
val Component.webAPI: WebAPI get() = (scope as JProScope).webAPI
```

The `scope` property of any `Component` will be the `JProScope` so we can cast it and access the `webAPI` property we defined in our custom scope class.

## Testing with Scopes

Since Scopes allow you to create separate instances of components that are usually singletons, you can leverage Scopes to test Views and even whole App instances.

For example, to generate a new Scope and lookup a View in that scope, you can use the following code:

```kotlin
val testScope = Scope()
val myView = find<MyView>(testScope)
```



