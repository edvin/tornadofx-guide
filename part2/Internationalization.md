# Internationalization

TornadoFX makes it very easy to support multiple languages in your app.

### Internationalization in Components

Each `Component` has access to a property called `messages` of type `ResourceBundle`. This can be used to look messages in the current locale and assign them to controls programmatically:

```kotlin
class MyView: View() {
    init {
        val helloLabel = Label(messages["hello"])
    }
}
```
> A label is programmatically configured to get it's text from a resource bundle

As well of the shorthand syntax `messages["key"]`, all other functions of the `ResourceBundle` class is available as well.

The bundle is automatically loaded by looking up a base name equal to the fully qualified class name of the `Component`. For a Component named `views.CustomerList`, the corresponding resource bundle in `/views/CustomerList.properties` will be used. All normal variants of the resource bundle name is supported, see [ResourceBundle Javadocs](https://docs.oracle.com/javase/8/docs/api/java/util/ResourceBundle.html) for more information.

### Internationalization in `FXML`

When an `FXML` file is loaded via the `fxml` delegate function, the corresponding `messages` property of the component will be used in exactly the same way.

```xml
<HBox>
    <Label text="%hello"/>
</HBox>
```
> The message with key `hello` will be injected into the label.

### Default Global Messages

You can add a global set of messages with the base name `Messages` (for example `Messages_en.properties`) at the root of the class path.

### Automatic lookup in parent bundle

When a key is not found in the component bundle, or when there is no bundle for the currrent component, the global resource bundle is consulted. As such, you might use the global bundle for all resources, and place overrides in the per component bundle.

### Friendly error messages

In stead of throwing an exception when a key is not available in your bundle, the value will simply be `[key]`. This makes it easy to spot your errors, and your UI is still fully functional while you add the missing keys.

### Configuring the locale

The default locale is the one retrieved from `Locale.getDefault()`. You can configure a different locale by issuing:

```kotlin
FX.locale = Locale("my-locale")
```

The global bundle will automatically be changed to the bundle corresponding to the new locale, and all subsequently loaded components will get their bundle in the new locale as well.

### Overriding resource bundles

If you want to change the bundle for a component after it's been initialized, or if you simply want to load a spesific bundle without relying on the conventions, simply assign the new bundle to the `messages` property of the component.

If you want to use the overriden resource bundle to load `FXML`, make sure you change the bundle before you load the root view:

```kotlin
class MyView: View() {
    init { messages = ResourceBundle.getBundle("MyCustomBundle") }
    override val root = HBox by fxml()
}
```
> A manually overriden resource bundle is used by the `FXML` file corresponding to the View

The same technique can be used to override the global bundle by assigning to `FX.messages`.

### Startup locale

You can override the default locale as early as the `App` class `init` function by assigning to `FX.locale`.

### Controllers and Fragments as well

The same conventions are valid for `Controllers` and `Fragments`, since the functionality is made available to their common super class, `Component`.

