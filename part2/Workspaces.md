# Workspaces

Java Business applications have traditionally been based on one of the Rich Client Frameworks,
namely the NetBeans Platform or Eclipse RCP. An important reason for choosing an RCP platform has been the workspace-like functionality they provide. Some important features of a workspace are:

* Common action buttons that tie to the state of the docked view (Save, Refresh, etc)
* Context-based UI nodes added to the common workspace interface
* Navigation stack for traversing visited views, controlled through back and forward buttons like a web browser
* Menu system with dynamic contributions and modifications

TornadoFX has begun to bridge the gap between the RCP platforms by providing Workspaces. While still in its infancy,
the default functionality is a solid foundation for business applications in need of the features discussed above.

## A Simple Workspace Example

To kick off a Workspace app, all you need to do is to subclass `App` and set the primary `View` to `Workspace::class`.
The result can be seen below (Figure 16.1):

```kotlin
class MyApp : App(Workspace::class)
```

**Figure 16.1**

![](http://i.imgur.com/7k3Uskm.png)

The resulting Workspace consists of a button bar with four default buttons and an empty content area below it.
The content area can house any `UIComponent`. You add a component to the `content` area by calling `workspace.dock()` on it. If you
show the `Workspace` without a docked `View`, it will by default only take up the space needed for the buttons. The window in Figure 16.1
was resized after it was opened.

Let's pretend we have a `CustomerList` component that we would like to dock in the `Workspace` as the application starts. We do this by overriding the `onBeforeShow` callback:

```kotlin
class MyApp : App(Workspace::class) {
    override fun onBeforeShow(view: UIComponent) {
        workspace.dock<CustomerList>()
    }
}
```

**Figure 16.2**

![](http://i.imgur.com/E79aeDl.png)

To keep things focused, we will leave out the `CustomerList` implementation code which simply displays a `TableView` with some Customers. What is interesting however, is that the **Refresh** button in the `Workspace` was enabled when the `CustomerList` was docked, while the **Save** button remained disabled.

### Leveraging the Workspace buttons

Whenever a `UIComponent` is docked in the `Workspace`, the **Refresh**, **Save**, and **Delete** buttons will be enabled by default. This happens because the `Workspace` looks at the `refreshable`, `savable` and `deletable` properties in the docked component. Every `UIComponent` returns a boolean property with the default value of `true`, which the `Workspace` then connects to the enabled state of these buttons. In the `CustomerList` example, the TornadoFX maintainers made sure the **Save** button was always disabled by overriding this property:

```
override val savable = SimpleBooleanProperty(false)
```

We can achieve the same result by calling `disableSave()` in the `init` block, and the same goes for `disableRefresh()` and `disableDelete()`.

We did not touch the other buttons, so they remain `true` as per the default. Whenever the `Refresh` button is called, it will fire the `onRefresh` function in the `View`. You can override this to provide your refresh action:


```
class MyApp: App(MyWorkspace::class) {
    override fun onBeforeShow(view: UIComponent) {
        workspace.dock<MyView>()
    }
}

class MyWorkspace: Workspace() {
    override fun onRefresh() {
      customerTable.asyncItems { customerController.listCustomers() }
    }
}
```

Same goes for the **Delete** button. We will revisit the **Save** button and introduce a neat trick to only activate it when there are dirty changes later in this chapter.

### Tabbed Views

You may at one point dock a View containing a `TabPane` inside of a `Workspace`, and then add tabs which represents further UIComponents. You can quite easily proxy the savable, refreshable and deletable state and actions from the `Workspace` onto the `View` represented by the currently active Tab. Consider a Customer Editor which has tabs for editing customer data, and one for editing contacts for that customer. Whenever the user selects one of the tabs, the buttons in the Workspace should interact with the state and actions from the selected tab view.

```kotlin
class CustomerEditor : View("Customer Editor") {
    override val root = tabpane {
        tab(CustomerBasicDataEditor::class)
        tab(ContactListEditor::class)
        connectWorkspaceActions()
    }
}
```

That single call to `connectWorkspaceActions()` takes care of everything for us. The actual implementation of the two sub views are omitted for brevity, but you can imagine that they share a `CustomerViewModel` injected into the scope they share for example.

The actual implementation of `connectWorkspaceActions` is quite simple, and reveals what's going on under the cover:

```kotlin
fun TabPane.connectWorkspaceActions() {
    savableWhen { savable }
    whenSaved { onSave() }

    deletableWhen { deletable }
    whenDeleted { onDelete() }

    refreshableWhen { refreshable }
    whenRefreshed { onRefresh() }
}
```

This function is declared inside `UIComponent`, so the `savableWhen`, `deletableWhen`, and `refreshableWhen` are performed on the UIComponent. Those state are then bound to the `savable`, `deletable` and `refreshable` state of the `TabPane`. But wait... a `TabPane` does not have those functions?! Yes, in TornadoFX it does :) You can probably guess that the implementation is again another proxy into the currently selected `Tab` in the `TabPane`, and a lookup of the UIComponent represented by the `content` property of that `Tab`. Whenever the `Tab` changes (or when the content of the tab changes), the underlying UIComponent is looked up, and the pertinent states are bound to the `Workspace`.

It would also be possible to bind these states and connect the actions more explicitly. You will never or seldom need to do that, but the following example might help your understanding of the proxy mechanism.

```kotlin
class TooExplicitCustomerEditor : View() {
    override val root = tab {
        ...
    }
    override val savable = root.savable
    override val refreshable = root.refreshable
    override val deletable = root.deletable

    override fun onSave() {
        root.onSave()
    }

    override fun onDelete() {
        root.onDelete()
    }

    override fun onRefresh() {
        root.onRefresh()
    }
}
```

As mentioned, you never need to do this and should always use the `connectWorkspaceActions` call, but you might want to override one of `onSave`, `onDelete`or `onRefresh` to perform some action in the main editor before calling the same action inside the active tab by calling `root.onXXX`. Let's say that the refresh call in the main editor reloads the customer, but you also want to have the contact list refresh if that view is currently active. This could be done
like this:

```kotlin
class CustomerEditor : View() {
    val customerController : CustomerController by inject()
    val customer: CustomerModel by inject()

    override val root = tabpane {
        tab(CustomerBasicDataEditor::class)
        tab(ContactListEditor::class)
        connectWorkspaceActions()
    }

    override val onRefresh() {
        runAsync {
            customerController.getCustomer(customer.id.value)
        } ui {
            customer.item = it
            root.onRefresh()
        }
    }
}
```

This little trick enables you to handle the actual reload of the customer in the main view instead of reimplementing it in every tab.

### Forwarding button state and actions

As we have seen, the currently docked View controls the Workspace buttons. Some times you dock nested Views inside the main View, and you would like that nested View to control the buttons and actions instead. This can easily be done with the `forwardWorkspaceActions` function. You can change the forwarding however you see fit, for example on focus or on click on some component inside the nested View.

```kotlin
class CustomerEditor : View() {
    override val root = hbox {
        val basicDataEditor = find<CustomerBasicDataEditor>()
        add(basicDataEditor)
        forwardWorkspaceActions(basicDataEditor)
        add(ContactListEditor::class)
    }
}

```

### Modifying the default workspace

The default workspace only gives you basic functionality. As your application grows you will want to suplement the
 toolbar with more buttons and controls, and maybe a `MenuBar` above it. For small modifications you can augment
 it in the `onBeforeFunction` as we did above, but you will most probably want to subclass as the customizations
 become more advanced. The following code and image is taken from a real world CRM application:

```kotlin
class CRMWorkspace : Workspace() {
    init {
        add(MainMenu::class)
        add(RestProgressBar::class)
        add(SearchView::class)
    }
}
```

The `CRMWorkspace` loads three other views into it. One providing a `MenuBar`, then the default `RestProgressBar` is
added, and lastly a `SearchView` providing a search input field is added.

The Workspace has a pretty good idea about where to place whatever you add to it. For example, buttons will by default
be added after the four default buttons, while other components are added to the far right of the ToolBar. The MenuBar
is automatically added above the ToolBar, at the top of the screen.

Figure 16.3 shows how it looks in production, with a little bit of custom styling and a `CustomerEditor` docked into it.
This application happens to be in Norwegian, and some of the information in the Customer card has been removed.

**Figure 16.3**

![](http://i.imgur.com/BbqCYg9.png)

You will notice that the **Save** button is enabled in this View. This is because the `savable` property is bound to
the dirty state property of the view model:

```kotlin
val model: CustomerModel by inject()
override val savable = model.dirty
```

When a customer is loaded, the **Save** button will stay disabled until an edit has been made. To save, we override the `onSave` function:

```kotlin
override fun onSave() {
    runAsync {
        customerController.save(customer.item)
    } ui { saved ->
        customer.update(saved)
    }
}
```

This particular `customerController.save` call will return the `Customer` from the server once it is saved. If the server made any changes
to our customer object, they would have been reflected in the saved customer we got back. For that reason, we call
`customer.update(saved)` which is function you get for free if you implement `JsonModel`. This makes sure that changes
from the server is pushed back into the model. This is completely optional, and you might just want to do `customerController.save(customer.item)`.

## Title and heading

When a view is docked, the title of the Workspace will match the title of that view. There is also a heading
text in the workspace that by default shows the same text as the title. The heading can be overriden by assigning to
the `heading` variable or binding to the `headingProperty` property. If you want to completely remove the heading, augment
the workspace with `workspace.headingContainer.removeFromParent()` or just hide it. You can also put whatever
nodes you want inside the heading container. You saw this trick in the CRM screenshot, where a Gravator icon was placed
to the left of the customer name.

## Dynamic elements in the ToolBar

Some views might need more buttons or functionality added to the ToolBar, but once you navigate away from the view it
wouldn't make sense to keep them around. The Workspace will actually track whatever elements you add to it while a view is
docked and remove those changes when the view is undocked. The perfect place to add these extra buttons would be the `onDock`
call of the view.

Every `UIComponent` has a property called `workspace` which will point to the current Workspace for the current Scope. Let's
add an "Add Customer" button to the Workspace whenever the `CustomerList` is docked:

```kotlin
override fun onDock() {
    with (workspace) {
        button("Add Customer").action {
            addCustomer()
        }
    }
}
```

The Workspace will now look like in Figure **16.4**

![](http://i.imgur.com/mSKv57I.png)

It looks like a default button. You can remove the border around the button by adding the `icon-only` css class to it. Optionally
you can configure an icon for the graphic node if you like. The built in icons are svg shapes added in the built in `workspace.css`
but feel free to add your icon in any way you see fit. Let's add an icon from the FontAwesomeFX library and make it look like
the other buttons:

```kotlin
button("Add Customer") {
    addClass("icon-only")
    graphic = FontAwesomeIconView(PLUS_CIRCLE).apply {
        style {
            fill = c("#818181")
        }
        glyphSize = 18
    }
    action { addCustomer() }
}
```

In a real application you would use a css class so you don't need to configure the fill for every button you add. The result can be seen in Figure 16.5:

**Figure 16.5**

![](http://i.imgur.com/32pmJAs.png)

## Navigating between docked views

Our Customer List is configured so that whenever you double click a customer you will be taken to an editor for that customer.
The TableView binds the selected user to a `CustomerModel` view model object, and the action is performed like this:

```kotlin
tableview(customers) {
    column("First Name", Customer::firstNameProperty)
    column("Last Name", Customer::lastNameProperty)
    bindSelected(model)
    onUserSelect { workspace.dock<CustomerEditor>() }
}
```

The only thing we need to do is actually dock the `CustomerEditor` when the user selects a row. Since the `CustomerEditor`
will be looked up in the same scope we are currently in, it will have access to the selected customer as well:

```kotlin
class CustomerEditor : Fragment("Customer Editor") {
    val customer: CustomerModel by inject()
    override val savable = customer.dirty
    override val headingProperty = customer.fullName

    override val root = form {
        fieldset("Customer Details") {
            field("First Name") {
                textfield(customer.firstName)
            }
            field("Last Name") {
                textfield(customer.lastName)
            }
        }
    }

    override fun onSave() {
        customer.commit()
    }

    override fun onRefresh() {
        customer.rollback()
    }
}
```

The customer model is injected, and will contain the selected customer from the list. The `savable` property is bound
to the `dirty` property of the model and the `headingProperty` is bound to a `StringBinding` called `fullName`, which
concatinates the first and last names and updates whenever they are changed. The form fields bind to the name properties
and lastly the `onSave` and `onRefresh` functions are implemented to react to the corresponding Workspace buttons.

**Figure 16.6**

![](http://i.imgur.com/ZPomPti.png)

We can see that the `title` and `heading` are indeed displaying separate information. Since we haven't made any edits
yet, the **Save** button is disabled, while the **Refresh** button is available, and would roll back any changes made
since the last commit.

The `back` button is enabled as well, and clicking it would navigate back to the Customer list. This is a very powerful
feature which enables browser like navigation in your application with very little effort on your part. The Workspace
keeps a navigation stack of configurable depth. By default it will contain 10 previously docked views. You can configure the
`maxViewStackDepth` to change the number of views held in the navigation stack.

## Alternative to overriding `onSave` and `onRefresh`

Some times you want to access an object in one of the workbench button actions but you want to avoid creating a variable
 for that object. Instead you can use the `whenSaved` and `whenRefreshed` callbacks, which can be configured from anywhere.
 Important: They are alternatives to `onSave` and `onRefresh` so you should only do one or the other. Let's say we want to
 refresh a TableView when the **Refresh** button is clicked. We can configure this inside the builder for the TableView:

```kotlin
tableview {
    whenRefreshed {
        asyncItems { controller.loadItems() }
    }
}
```

This is a handy alternative in some situations, but make sure you only choose one of the strategies.

## Advanced scope navigation

When you leverage injected view models together with a navigation stack, some interesting challenges appear that need
to be addressed. If you removed the **Back** button (`workspace.backButton.removeFromParent()`) or set the `maxViewStackDepth` to
`0` you can disregard this particular challenge, but to leverage this powerful navigation paradigm, there are some things
you need to think about.

Consider our prevous example with an injected `CustomerModel` that represents the currently selected customer in the `CustomerList`
while also being used by the `CustomerEditor` to edit that same customer. Then let's assume that there is a way to search for
a customer and edit it, perhaps using a `TextField` in the ToolBar of the Workspace as a search entry point. If you search for
a new customer and go on to edit it, then navigate back to the previous customer editor, it would suddenly operate on the
last customer you set in the `CustomerModel`. You can probably imagine the ensuing havoc.

Fortunately, the scoping support stretches far into the Workspace feature and provides some handy tools for this particular situation.

We need to find a way to contain the scope for the pair of CustomerList and CustomerEditor so they can work together while allowing
other views to use the `CustomerModel`, but in a different scope. It's actually quite easy. Whenever you create a new `CustomerList`,
also create a new Scope. If you were to do this manually, it would look something like this:

```kotlin
// Create a new scope, but keep the current workspace
val newScope = Scope(workspace)

// Find the CustomerList in the new scope
val customerList = find<CustomerList>(newScope)

// Dock the customerList in the workspace
workspace.dock(customerList)
```

Those three distinct operations can be performed in a single call:

```kotlin
workspace.dockInNewScope<CustomerList>()
```

When the CustomerList docks the CustomerEditor later on, it happens in this new scope. But what about the search field?

We would need to provide a separate scope for the `CustomerEditor` that should show the result of the search, but
were we would also also need to inject the customer model containing the selected customer into the new scope. This
following code is imagined inside the action that selects a customer from the search result:

```kotlin
fun editCustomer(customer: Customer) {
// Create a view model for the customer
val model = CustomerModel(customer)

// Create a new scope, but keep the current workspace
val newScope = Scope(workspace)

// Insert the customer model into the new scope
newScope.set(model)

// Find the CustomerEditor in the new scope
val editor = find<CustomerEditor>(newScope)

// Dock the editor
workspace.dock(editor)
}
```

That's a lot of steps. Fortunately, we can do that as well in a single call:

```kotlin
fun editCustomer(customer: Customer) {
    workspace.dockInNewScope<CustomerEditor>(CustomerModel(customer))
}
```

The `dockInNewScope` function takes a vararg list of injectable objects to insert into the new scope before looking
up our CustomerEditor and docking it.

Separating scopes this way makes sure we can utilize injected view models without being afraid of other views stepping
on our data. It is a pragmatic approach to an intricate problem. It also gives you a way of bleeding injectables
into new scopes, should your use case require it.

## Custom ViewStack optimizations

Some use cases might require you to make sure that the user cannot go back to a certain view after he has navigated
to the prior view. You can remove your self from the View Stack on unDock like this:

```kotlin
override fun onUndock() {
    workspace.viewStack.remove(this)
}
```

## Docking multiple views in the editor area

The Workspace provides an alternative way to navigate between views. Instead of back and forward buttons, you can choose
to dock multiple views inside a TabPane in the editor area. The Workspace has a `navigationMode` property that lets
you change how the views are represented in the editor area. The default is `Workspace.NavigationMode.Stack`. The following example
creates a tabbed Workspace that automatically docks two views inside it when it's created:

```kotlin
class TabbedWorkspace: Workspace("Tabbed Workspace", NavigationMode.Tabs) {
    init {
        dock<FirstView>()
        dock<SecondView>()
    }
}
```

**Figure 16.7**

![](http://i.imgur.com/jq6Ymsr.png)

> A Workspace in Tabs mode automatically hides the navigation buttons as they are no longer needed

You can create a starting point for this Workspace from a normal `App` class:

```kotlin
class TabbedWorkspaceApp : App(TabbedWorkspace::class)
```

The views docked inside the Workspace tabs will have their `onDock` function called whenever they are added and
also when they are subsequently chosen as the active Tab. Correspondingly, the `onUndock` function is called
whenever it is no longer the active Tab, as well as when it's removed from the TabPane using the close button on the tab.

You can control the closable state of a View docked inside the TabPane via the `closeable` property in `UIComponent`.
It returns a `BooleanExpression` with the default value of `true` but you can override it to bind against another
property or simply return another `SimpleBooleanValue(false)` to make it uncloseable. This example makes sure you
cannot close the tab before the CustomerModel inside it is committed or rolled back:

```kotlin
class CustomerEditor : View("Customer Editor") {
    val customer: CustomerModel by inject()
    override val closeable = customer.dirty.not()
}
```

## Drawer navigation

The Workspace has built in support for the Drawer control. You can access `workspace.leftDrawer` and `workspace.rightDrawer` to
add items to each drawer. They will show up on either the left or right side whenever you have added one or more items to them.

Items added from a View in `onDock` will automatically be removed when the View is undocked. Items added directly in the Workspace
subclass, from the `onBeforeShow` App callback or from any other place will stay until they are manually removed.

The combination of static and dynamic drawer items makes for a very powerful navigation and menu structure. Only your imagination is the limit!

The following example creates a customize Workspace primed with a docked Customer Editor in the editor area and the three
drawer items we created in the Drawer chapter configured statically in the `leftDrawer` of the Workspace:

```kotlin
// A Form based View we will dock in the workspace editor area
class CustomerEditor : View("Customer Editor") {
    override val root = form {
        fieldset(title) {
            field("Name") { textfield() }
            field("Username") { textfield() }
            button("Save")
        }
    }
}

class DrawerWorkspace : Workspace() {
    init {
        // Dock the Customer Editor by default
        dock<CustomerEditor>()
    }

    init {
        // Add items to the left drawers
        with(leftDrawer) {
            item("Screencasts") {
                webview {
                    prefWidth = 470.0
                    engine.userAgent = iPhoneUserAgent
                    engine.load(TornadoFXScreencastsURI)
                }
            }
            item("Links") {
                listview(links) {
                    cellFormat { link ->
                        graphic = hyperlink(link.name).action {
                            hostServices.showDocument(link.uri)
                        }
                    }
                }
            }
            item("People") {
                tableview(people) {
                    column("Name", Person::name)
                    column("Nick", Person::nick)
                }
            }
        }
    }

    // Sample data and configuration omitted for this example
}
```

In Figure 16.8 we have expanded the _Links_ drawer item. Notice how it pushes the Customer Editor to the right.

**Figure 16.8**

![](http://i.imgur.com/HnQ2pFW.png)

By right clicking the drawer and checking the `Floating drawers` option, the expanded drawer item content will
instead float above the content, like in Figure 16.9:

**Figure 16.9**

![](http://i.imgur.com/oiDvrMh.png)

This could be a good idea depending on the available space and the nature of the docked content. You can change the
floating drawer mode in code as well, by setting `leftDrawer.floatingDrawers = true`.

Remember that Views can contribute drawer items programmatically in their `onDock` callback. Use this to
provide extra tools for an advanced editor for example. They can easily communicate between each other
using ViewModels. It is recommended to create a new scope to make it easier for these view parts to work in concert
on shared data structures.

## Vetoing navigation from the docked View

The currently docked View will receive a callback whenever the Back or Forward buttons of the Workspace is clicked. These functions
are called `onNavigateBack` and `onNavigateForward`. The default implementation returns true to signal that the navigation should proceed.
You can however return false to stop the navigation and instead implement your own logic to decide what happens in the UI when one of the
navigation buttons are clicked.
