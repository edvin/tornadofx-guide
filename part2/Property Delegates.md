# Property Delegates

Kotlin is packed with great language features, and [delegated properties](https://kotlinlang.org/docs/reference/delegated-properties.html) are a powerful way to specify how a property works and create re-usable policies for those properties. On top of the ones that exist in Kotlin's standard library, TornadoFX provides a few more property delegates that are particularly helpful for JavaFX development.

### Single Assign

It is often ideal to initialize properties immediately upon construction. But inevitably there are times when this simply is not feasible. When a property needs to delay its initialization until it is first called, a lazy delegate is typically used. You specify a lambda instructing how the property value is initialized when its getter is called the first time.

```kotlin
val fooValue by lazy { buildExpensiveFoo() }
```

But there are situations where the property needs to be assigned later not by a value-supplying lambda, but rather some external entity at a later time. When we leverage type-safe builders we may want to save a `Button` to a class-level property so we can reference it later. If we do not want `myButton` to be nullable, we need to use the [`lateinit` modifier](https://kotlinlang.org/docs/reference/properties.html#late-initialized-properties).

```kotlin
class MyView: View() {

    lateinit var myButton: Button

    override val root = vbox {
        myButton = button("New Entry")
    }
}
```

The problem with `lateinit` is it can be assigned multiple times by accident, and it is not necessarily thread safe. This can lead to classic bugs associated with mutability, and you really should strive for immutability as much as possible ( *Effective Java* by Bloch, Item #13).

By leveraging the `singleAssign()` delegate, you can guarantee that property is *only* assigned once. Any subsequent assignment attempts will throw a runtime error, and so will accessing it before a value is assigned. This effectively gives us the guarantee of immutability, although it is enforced at runtime rather than compile time.

```kotlin
class MyView: View() {

    var myButton: Button by singleAssign()

    override val root = vbox {
        myButton = button("New Entry")
    }
}
```

Even though this single assignment is not enforced at compile time, infractions can be captured early in the development process. Especially as complex builder designs evolve and variable assignments move around, `singleAssign()` is an effective tool to mitigate mutability problems and allow flexible timing for property assignments.

By default, `singleAssign()` synchronizes access to its internal value. You should leave it this way especially if your application is multithreaded. If you wish to disable synchronization for whatever reason, you can pass a `SingleAssignThreadSafetyMode.NONE` value for the policy.

```kotlin
var myButton: Button by singleAssign(SingleAssignThreadSafetyMode.NONE)
```

### JavaFX Property Delegate

Do not confuse the JavaFX `Property` with a standard Java/Kotlin "property". The `Property` is a special type in `JavaFX` that maintains a value internally and notifies listeners of its changes. It is proprietary to JavaFX because it supports binding operations, and will notify the UI when it changes. The `Property` is a core feature of JavaFX and has its own JavaBeans-like pattern.

This pattern is pretty verbose however, and even with Kotlin's syntax efficiencies it still is pretty verbose. You have to declare the traditional getter/setter as well as the `Property` item itself.

```kotlin
 class Bar {
    private val fooProperty by lazy { SimpleObjectProperty<T>() }
    fun fooProperty() = fooProperty
    var foo: T
        get() = fooProperty.get()
        set(value) = fooProperty.set(value)
}
```

Fortunately, TornadoFX can abstract most of this away. By delegating a Kotlin property to a JavaFX `property()`, TornadoFX will get/set that value against a new `Property` instance. To follow JavaFX's convention and provide the `Property` object to UI components, you can create a function that fetches the `Property` from TornadoFX and returns it.

```kotlin
class Bar {
    var foo by property<String>()
    fun fooProperty() = getProperty(Bar::foo)
}
```

Especially as you start working with `TableView` and other complex controls, you will likely find this pattern helpful when creating model classes, and this pattern is used in several places throughout this book.

Note you do not have to specify the generic type if you have an initial value to provide to the property. In the below example, it will infer the type as `String.

```kotlin
class Bar {
    var foo by property("baz")
    fun fooProperty() = getProperty(Bar::foo)
}
```

#### Alternative Property Syntax

There is also an alternative syntax which produces almost the same result:

```kotlin
import tornadofx.getValue
import tornadofx.setValue

class Bar {
    val fooProperty = SimpleStringProperty()
    var foo by fooProperty
}
```

Here you define the JavaFX property manually and delegate the getters and setters directly from the property. This might look cleaner to you, and so you are free to choose whatever syntax you are most comfortable with. However, the first alternative creates a JavaFX compliant property in that it exposes the `Property` via a function called `fooProperty()`, while the latter simply exposes a variable called `fooProperty`. For TornadoFX there is no difference, but if you interact with legacy libraries that require a property function you might need to stick with the first one.

#### Null safety of Properties

By default properties will have a [Platform Type](https://kotlinlang.org/docs/reference/java-interop.html#notation-for-platform-types) with uncertain nullability and completely ignore the null safety of Kotlin:

```kotlin
class Bar {
    var foo:String by property<String>()
    fun fooProperty() = getProperty(Bar::foo)
    
    val bazProperty = SimpleStringProperty()
    var baz: String? by bazProperty
    
    init {
        foo = null
        foo.length // Will throw NPE during runtime
        
        baz = null
        baz.length // Will throw NPE during runtime
    }
}
```

To remedy this you can set the type of your property on the `var` (not on the Property-Object itself!). But keep in mind to set a default
value on the property object or you will get an NPE anyways:

```kotlin
class Bar {
    var foo:String by property<String>("") // Non-nullable String with default value
    fun fooProperty() = getProperty(Bar::foo)
    
    val bazProperty = SimpleStringProperty()
    var baz: String? by bazProperty // Nullable String
    
    init {
        foo = null // Will no longer compile
        foo.length
        
        baz = null
        baz.length // Will no longer compile
    }
}
```

### FXML Delegate

If you have a given `MyView` View with a neighboring FXML file `MyView.fxml` defining the layout, the `fxid()` property delegate will retrieve the control defined in the FXML file. The control must have an `fx:id` that is the same name as the variable.

```xml
<Label fx:id="counterLabel">
```

Now we can inject this `Label` into our `View` class:

```kotlin
val counterLabel : Label by fxid()
```

Otherwise, the ID must be specifically passed to the delegate call.

```kotlin
val myLabel : Label by fxid("counterLabel")
```

Please read Chapter 10 to learn more about FXML.
