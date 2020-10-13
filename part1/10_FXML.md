# FXML and Internationalization

TornadoFX's type-safe builders provide a fast, easy, and declarative way to construct UI's. This DSL approach is encouraged because it is more flexible, reliable, and simpler. However, JavaFX also supports an XML-based structure called FXML that can also build a UI layout. TornadoFX has tools to streamline FXML usage for those that need it.

If you are unfamiliar with FXML and are perfectly happy with type-safe builders, please feel free to skip this chapter. If you need to work with FXML or feel you should learn it, please read on. You can also take a look at the [official FXML documentation](https://docs.oracle.com/javase/8/javafx/fxml-tutorial/why_use_fxml.htm) to learn more.

## Reasons for Considering FXML

While the developers of TornadoFX strongly encourage using type-safe builders, some situations and factors might cause you to consider using FXML.

### Separation of Concerns

With FXML it is easy to separate your UI logic code from the UI layout code. This separation is just as achievable with type-safe builders by utilizing MVP or other separation patterns. But some programmers find FXML forces them to maintain this separation and prefer it for that reason.

### WYSIWYG Editor

FXML files also can be edited and processed by [Scene Builder](http://www.oracle.com/technetwork/java/javase/downloads/javafxscenebuilder-info-2157684.html), a visual layout tool that allows building interfaces via drag-and-drop functionality. Edits in Scene Builder are immediately rendered in a WYSIWYG ("What You See is What You Get") pane next to the editor.

If you prefer making interfaces via drag-and-drop or have trouble building UI's with pure code, you might consider using FXML simply to leverage Scene Builder.

> The Scene Builder tool was created by Oracle/Sun but is now [maintained by Gluon](http://gluonhq.com/labs/scene-builder/), an innovative company that invests heavily in JavaFX technology, especially for the mobile market.

### Compatibility with Existing Codebases

If you are converting an existing JavaFX application to TornadoFX, there is a strong chance your UI was constructed with FXML. If you hesitate to transition legacy FXML to TornadoFX builders or would like to put that off as long as possible, TornadoFX can at least streamline the processing of FXML.

## How FXML works

The `root` property of a `View` represents the top-level `Node` containing a hierarchy of children Nodes, which makes up the user interface. When you work with FXML, you do not instantiate this root node directly, but instead, ask TornadoFX to load it from a corresponding FXML file. By default, TornadoFX will look for a file with the same name as your view with the `.fxml` file ending in the same package as your `View` class. You can also override the FXML location with a parameter if you want to put all your FXML files in a single folder or organize them some other way that does not directly correspond to your `View` location.

## A Simple Example

Let's create a basic user interface that presents a `Label` and a `Button`. We will add functionality to this view so when the `Button` is clicked, the `Label` will update its `text` with the number of times the `Button` has been clicked.

Create a file named `CounterView.fxml` with the following content:

```xml
<?import javafx.geometry.Insets?>
<?import javafx.scene.control.Button?>
<?import javafx.scene.control.Label?>
<?import javafx.scene.layout.BorderPane?>
<?import javafx.scene.layout.VBox?>
<?import javafx.scene.text.Font?>
<BorderPane xmlns="http://javafx.com/javafx/null" xmlns:fx="http://javafx.com/fxml/1">
    <padding>
        <Insets top="20" right="20" bottom="20" left="20"/>
    </padding>

    <center>
        <VBox alignment="CENTER" spacing="10">
            <Label text="0">
                <font>
                    <Font size="20"/>
                </font>
            </Label>
            <Button text="Click to increment" />
        </VBox>
    </center>
</BorderPane>
```

> You may notice above you have to `import` the types you use in FXML just like coding in Java or Kotlin. Intellij IDEA should have a plugin to support using ALT+ENTER to generate the `import` statements.

If you load this file in Scene Builder you will see the following result (Figure 9.1).

**Figure 9.1**

![](https://i.imgur.com/WulUHYa.png)

Next, let's load this FXML into TornadoFX.

### Loading FXML into TornadoFX

We have created an FXML file containing our UI structure, but now we need to load it into a TornadoFX `View` for it to be usable. Logically, we can load this `Node` hierarchy into the `root` node of our `View`. Define the following `View` class:

```kotlin
class CounterView : View() {
    override val root : BorderPane by fxml()
}
```

Note that the `root` property is defined by the `fxml()` delegate. The `fxml()` delegate takes care of loading the corresponding `CounterView.fxml` into the `root` property.  If we placed `CounterView.fxml` in a different location (such as `/views/`) that is different than where the `CounterView` file resides, we would add a parameter.

```kotlin
class CounterView : View() {
    override val root : BorderPane by fxml("/views/CounterView.fxml")
}
```

We have laid out the UI, but it has no functionality yet. We need to define a variable that holds the number of times the `Button` has been clicked. Add a variable called `counter` and define a function that will increment its value:

```kotlin
class CounterView : View() {
    override val root : BorderPane by fxml()
    val counter = SimpleIntegerProperty()

    fun increment() {
        counter.value += 1
    }
}
```

We want the `increment()` function to be called whenever the `Button` is clicked. Back in the FXML file, add the `onAction` attribute to the button:

```xml
<Button text="Click to increment" onAction="#increment"/>
```

Since the FXML file automatically gets bound to our `View,` we can reference functions via the `#functionName` syntax. Note that we do not add parenthesis to the function call, and you cannot pass parameters directly. You can however add a parameter of type `javafx.event.ActionEvent` to the `increment` function if you want to inspect the source `Node` of the action or check what kind of action triggered the button. For this example we do not need it, so we leave the `increment` function without parameters.

## FXML file locations

By default, build tools like Maven and Gradle will ignore any extra resources you put into your source root folders, so if you put your FXML files there they won't be available at runtime unless you specifically tell your build tool to include them. This could still be problematic because IDEA might not pick up your custom resource location from the build file, once again resulting in failure at runtime. For that resource, we recommend that you place your FXML files in `src/main/resources` and either follow the same folder structure as your packages or put them all in a `views` folder or similar. The latter requires you to add the FXML location parameter to the `fxml` delegate and might be messy if you have a large number of Views, so going with the default is a good idea.

## Accessing Nodes with the `fxid` delegate

Using just FXML, we have wired the `Button` to call `increment()` every time it is called. We still need to bind the `counter` value to the `text` property of the `Label`. To do this, we need an identifier for the `Label`, so in our FXML file we add the `fx:id` attribute to it.

```xml
<Label fx:id="counterLabel">
```

Now we can inject this `Label` into our `View` class:

```kotlin
val counterLabel : Label by fxid()
```

This tells TornadoFX to look for a `Node` in our structure with the `fx:id` property with the same name as the property we defined (which is "counterLabel"). It is also possible to use another property name in the `View` and add a name parameter to the `fxid` delegate:

```kotlin
val myLabel : Label by fxid("counterLabel")
```

Now that we have a hold of the `Label`, we can use the binding shortcuts of TornadoFX to bind the `counter` value to the `text` property of the `counterLabel`. Our whole `View` should now look like this:

```kotlin
class CounterView : View() {
    override val root : BorderPane by fxml()
    val counter = SimpleIntegerProperty()
    val counterLabel: Label by fxid()

    init {
        counterLabel.bind(counter)
    }

    fun increment() {
        counter.value += 1
    }
}
```

Our app is now complete. Every time the button is clicked, the `label` will increment its count.

## Internationalization

JavaFX has strong support for multi-language UI's. To support internationalization in FXML, you normally have to register a resource bundle with the `FXMLLoader` and it will in return replace instances of resource names with their locale-specific value. A resource name is a key in the resource bundle prepended with `%`.

TornadoFX makes this easier by supporting a convention for resource bundles: Create a resource bundle with the same base name as your `View`, and it will be automatically loaded, both for use programmatically within the `View` and from the FXML file.

Let's internationalize the button text in our UI. Create a file called `CounterView.properties` and add the following content:

```
clickToIncrement=Click to increment
```

If you want to support multiple languages, create a file with the same base name followed by an underscore, and then the language code. For instance, to support French create the file `CounterView_fr.properties`. The closest geographical match to the current locale will be used.

```
clickToIncrement=Cliquez sur incrément
```

Now we swap the button text with the resource key in the FXML file.

```xml
<Button text="%clickToIncrement" onAction="#increment"/>
```

If you want to test this functionality and force a different `Locale`, regardless of which one you are currently in, override it by assigning `FX.local` when your `App` class is initialized.

```kotlin
class MyApp: App() {
    override val primaryView = MyView::class

    init {
        FX.locale = Locale.FRENCH
    }
}
```

You should then see your `Button` use the French text (Figure 9.2).

**Figure 9.2**

![](https://i.imgur.com/WqovpnM.png)

### Internationalization with Type-Safe Builders

Internationalization is not limited to use with FXML. You can also use it with type-safe builders. Set up your `.properties` files as specified before. But instead of using an embedded `%clickToIncrement` text in an FXML file, use the `messages[]` accessor to look up the value in the `ResourceBundle`. Pass this value as the `text` for the `Button`.

```kotlin
 button(messages["clickToIncrement"]) {
     setOnAction { increment() }
 }
```

## Summary

FXML is helpful to know as a JavaFX developer, but it is definitely not required if you are content with TornadoFX type-safe builders and do not have any existing JavaFX applications to maintain. Type-safe builders have the benefit of using pure Kotlin, allowing you to code anything you want right within the structure declarations. FXML's benefits are primarily separation of concerns between UI and functionality, but even that can be accomplished with type-safe builders. It also can be built via drag-and-drop through the Scene Builder tool, which may be preferable for those who struggle to build UI's any other way.
