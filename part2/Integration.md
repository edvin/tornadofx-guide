## Integrate with existing JavaFX Applications

TornadoFX can happily coexist with an existing Application written in either Kotlin or Java. This enables a gradual migration instead of performing a complete rewrite before you can benefit from TornadoFX in your apps. Feel free to skip this section if it doesn't apply to you.

**Note**: This feature is available as of version 1.4.3

To make TornadoFX aware of your application, perform a call to `registerApplication` in your `Application` class `start()` method:

```java
public class LegacyApp extends Application {
    public void start(Stage primaryStage) throws Exception {
        // Register JavaFX app with the TornadoFX runtime
        FX.registerApplication(this, primaryStage);
    }
}
```
> Existing JavaFX Application written in Java

### Accessing TornadoFX Views from plain old JavaFX

Let's say you have created your first TornadoFX View and would like to integrate the root node of that `View` into a plain JavaFX view. You could actually just instantiate the View and put the root Node wherever you like, but since `View` is a singleton, you want to make sure you only ever instantiate a single instance. For this, you can use the `FX.find` function.

First let's create a simple TornadoFX View, and this time let's write it in plain Java instead of Kotlin. We don't expect that people will write TornadoFX apps in Java, but it is indeed possible :)

```java
public class MyView extends View {
    public HBox getRoot() {
        return new HBox(new Label("I'm a TornadoFX View written in Java"));
    }
}
```
> TornadoFX View written in Java (!!)

```java
// Create a BorderPane for our Scene
BorderPane root = new BorderPane();

// Lookup a TornadoFX view and set it's root as the center node
HBox fxView = FX.find(MyView.class).getRoot();
root.setCenter(fxView);
```

The same mechanics can be used to access TornadoFX Controllers. For Fragments, simply instantiate them and put the root node where you like.

### Bootstrapping TornadoFX from Swing

You can even start a TornadoFX app inside your existing Swing applications!

```kotlin
public class SwingApp {
    private static void createAndShowGUI() {
         // initialize toolkit
        JFXPanel wrapper = new JFXPanel();
        
        // Init TornadoFX Application
        Platform.runLater(() -> {
            Stage stage = new Stage();
            MyApp app = new MyApp();
            app.start(stage);
        });
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(SwingApp::createAndShowGUI);
    }
}
```

### Integrate with existing Dependency Injection frameworks

You can access your existing beans exposed via any dependency injection framework by implementing a SAM class called `DIContainer` and registering it via `FX.setDicontainer()`. More information on the next page.