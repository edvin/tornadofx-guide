# Config settings and state

Saving application state is a common requirement for desktop apps. TornadoFX has several features which facilitates saving of UI state, preferences and general app configuration settings.

## The `config` helper

Each component can have arbitrary configuration settings that will be saved as property files in a folder called `conf` inside the current program folder.

Below is a login screen example where login credentials are stored in the view specific config object.

```kotlin
class LoginScreen : View() {
    val loginController: LoginController by inject()
    val username = SimpleStringProperty(this, "username", config.string("username"))
    val password = SimpleStringProperty(this, "password", config.string("password"))
    
    override val root = form {
       fieldset("Login") {
           field("Username:") { textfield(username) }
           field("Password:") { textfield(password) }
           buttonbar {
                button("Login").action {
                    runAsync {
                        loginController.tryLogin(username.value, password.value)
                    } ui { success ->
                        if (success) {
                            with(config) {
                                 set("username" to username.value)
                                 set("password" to password.value)
                                 save()
                            }
                            showMainScreen()
                        }
                    }
                }
            }
       }
    }
    
    fun showMainScreen() {
        // hide LoginScreen and show the main UI of the application
    }
}
```
> Login screen with credentials stored in the view specific config object

The UI is defined with the `TornadoFx` type safe builders, which basically contains a `form` with two `TextField`'s and a `Button`.
When the view is loaded, we assign the username and password values from the config object.
These values might be null at this point, if no prior successful login was performed.
We then bind the `username` and `password` to the corresponding `TextField`'s.

Last but not least, we define the action for the login button. Upon login, it calls the `loginController#tryLogin` function which takes the username and password from the `StringBindings` (which represent the input of the `TextField`s),
calls out to the service and returns true or false.

If the result is true, we update the username and password in the config object and calls save on it. Finally, we call `showMainScreen` which could hide the login screen and show the main screen of the application.

_Please not that the example is not a best practice for storing sensitive data, it merely illustrates how you can use the config object._

## Data types and default values

`config` also supports other data types. It is a nice practice to wrap multiple operations on the config object in a `with` block.

```kotlin
// Assign to x, default to 50.0
var x = config.double("x", 50.0)

var showPrices = config.boolean("showPrices", boolean)

with (config) {
    set("x", root.layoutX)
    set("showPrices", showPrices)
    save()
}
```

## Configurable config path

The `App` class can override the default path for config files by overriding `configBasePath`.

```kotlin
class MyApp : App(WelcomeView::class) {
    override val configBasePath: Paths.get("/etc/myapp/conf")
}
```

The path can also be relative, which means the path will be created inside the current working directory. By default, the base path is `conf`.

## Override config path per component

By default, a file called `viewClass.properties` is created inside the `configBasePath`. This can be overriden per component:

```kotlin
class MyView : View() {
    override val configPath = Paths.get("some/other/path/myview.properties")
```

You can also create the View spesific config file below the `configBasePath`, which would make sense in most situations. You do this by accessing the App class through the `app` property of the View.

```kotlin
class MyView : View() {
    override val configPath = app.configBasePath.resolve("myview.properties")
```

## Global application config

The App class also has a `config` property and a corresponding `configPath` property. By default, the configuration for the app class is named `app.config`. This can be overridden the same way you do for a View config.

The global configuration can be accessed by any component at any time in the life cycle of the application. Simply access `app.config` from anywhere to read or write your global configuration.

## JSON configuration settings

The `config` object supports `JsonObject`, `JsonArray` and `JsonModel`. You set them using `config.set("key" to value)` and retrieve them using `config.jsonObject("key")`, `config.jsonArray("key")` and `config.jsonModel("key")`.

## The `preferences` helper

As the `config` helper stores the information in a folder called `conf` per component (view, controller) the `preferences` helper will save settings into an OS specific way. In Windows systems they will be stored `HKEY_CURRENT_USER/Software/JavaSoft/....` on Mac os in `~/Library/Preferences/com.apple.java.util.prefs.plist` and on Linux system in `~/.java`. Where the `config` helper saves per component. The `preferences` helper is meant to be used application wide:

```kotlin
preferences("application") {
   putBoolean("boolean", true)
   putString("String", "a string")
}
```
Retrieving preferences:

```kotlin
var bool: Boolean = false
var str: String = ""
preferences("test app") {
    bool = getBoolean("boolean key", false)
    str = get("string", "")
}
```

The `preferences` helper is a TornadoFX builder around [java.util.Preferences](http://docs.oracle.com/javase/8/docs/technotes/guides/preferences/overview.html)
