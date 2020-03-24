
## Layout Debugger

When you're creating layouts or working on CSS it some times help to be able to visualise the scene graph and make live changes to the node properties of your layout. The absolutely best tool for this job is definitely the [Scenic View](http://fxexperience.com/scenic-view/) tool from [FX Experience](http://fxexperience.com/), but some times you just need to get a quick overview as fast as possible.

### Debugging a scene

Simply hit **Alt-Meta-J** to bring up the built in debugging tool *Layout Debugger*. The debugger attaches to the currently active `Scene` and opens a new window that shows you the current scene graph and properties for the currently selected node.

### Usage

While the debugger is active you can hover over any node in your View and it will be automatically highlighted in the debugger window. Clicking a node will also show you the properties of that node. Some of the properties are editable, like `backgroundColor`, `text`, `padding` etc.

When you hover over the node tree in the debugger, the corresponding node is also highlighted directly in the View.

![](https://i.imgur.com/kKH8ydl.gif)

### Stop a debugging session

Close the debugger window by hitting `Esc` and the debugger session ends. You can debug multiple scenes simultaneously, each debugging session will open a new window corresponding to the scene you debug.

### Configurable shortcut

The default shortcut for the debugger can be changed by setting an instance of `KeyCodeCombination` into `FX.layoutDebuggerShortcut`. You can even change the shortcut while the app is running. A good place to configure the shortcut would be in the `init` block of your `App` class.

### Adding features

While this debugger tool is in no way a replacement for Scenic View, we will add features based on *reasonable* [feature requests](https://github.com/edvin/tornadofx/issues). If the feature adds value for simple debugging purposes and can be implemented in a small amount of code, we will try to add it, or better yet, submit a [pull request](https://github.com/edvin/tornadofx/pulls). Have a look at the [source code](https://github.com/edvin/tornadofx/blob/master/src/main/java/tornadofx/LayoutDebugger.kt) to familiarise yourself with the tool.


### Entering fullscreen

To enter fullscreen you need to get a hold of the current `stage` and call `stage.isFullScreen = true`. The primary stage is the active stage unless you opened a modal window via `view.openModal()` or manually created a stage. The primary stage is available in the variable `FX.primaryStage`. To open the application in fullscreen on startup you should override `start` in your app class:

```kotlin
class MyApp : App(MyView::class) {
    override fun start(stage: Stage) {
        super.start(stage)
        stage.isFullScreen = true
    }
}
```

In the following example we toggle fullscreen mode in a modal window via a button:

```kotlin
button("Toggle fullscreen") {
    setOnAction {
        with (modalStage) { isFullScreen = !isFullScreen }
    }
}
```

# Logging

`Component` has a lazy initialized instance of `java.util.Logger` named `log`. Usage:

```kotlin
log.info { "Log message here" }
```

TornadoFX makes no changes to the logging capabilities of `java.util.Logger`. See the [javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/logging/Logger.html) for more information.
