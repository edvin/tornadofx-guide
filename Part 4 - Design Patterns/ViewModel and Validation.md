# ViewModel

TornadoFX doesn't force any particular architectural pattern on you as a developer, and works equally great with both [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller), [MVP](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter) and their derivatives.

To help with implementing these patterns, TornadoFX provides a tool called `ViewModel` that helps you cleanly separate your ui and business logic and gives you features like *rollback*/*commit* and *dirty state checking*. These features are hard or cumbersome to implement manually so in cases where you need them we strongly encourage you to leverage the `ViewModel`.

## A typical use case

Say you have a given domain type `Person`. We allow its two properties to be nullable for the sake of editing, and those values may have to be inputted later by the user. 

```kotlin
class Person(name: String? = null, title: String? = null) {
    var name by property(name)
    fun nameProperty() = getProperty(Person::name)

    var title by property(title)
    fun titleProperty() = getProperty(Person::title)
}
```

Consider a master detail view where you have a `TableView` displaying a list of people, and a `Form` where the currently selected user's information can be edited. Before we get into the ViewModel we will create a minimalistic version of this View without using the View Model.

![](http://i.imgur.com/KDWkFBy.png)

The code for our first attempt has a number of problems and we will address them one by one.

```kotlin
class PersonEditor : View() {
    override val root = BorderPane()
    var nameField : TextField by singleAssign()
    var titleField : TextField by singleAssign()
    var personTable : TableView<Person> by singleAssign()
    // Some fake data for our table
    val persons = listOf(Person("John", "Manager"), Person("Jay", "Worker bee")).observable()

    init {
        title = "Person Editor"

        with(root) {
            center {
                // TableView showing a list of people
                tableview(persons) {
                    personTable = this
                    column("Name", Person::nameProperty)
                    column("Title", Person::titleProperty)

                    // Edit the currently selected person
                    selectionModel.selectedItemProperty().onChange {
                        editPerson(it)
                    }
                }
            }

            right {
                form {
                    fieldset("Edit person") {
                        field("Name") {
                            textfield() {
                                nameField = this
                            }
                        }
                        field("Title") {
                            textfield() {
                                titleField = this
                            }
                        }
                        button("Save") {
                            setOnAction {
                                save()
                            }
                        }
                    }
                }
            }
        }
    }

    private fun editPerson(person: Person?) {
        if (person != null) {
            prevSelection?.apply {
                nameProperty().unbindBidirectional(nameField.textProperty())
                titleProperty().unbindBidirectional(titleField.textProperty())
            }
            nameField.bind(person.nameProperty())
            titleField.bind(person.titleProperty())
            prevSelection = person
        }
    }

    private fun save() {
        // Extract the selected person from the tableView
        val person = personTable.selectedItem!!

        // A real application would persist the person here
        println("Saving ${person.name} / ${person.title}")
    }

}
```

We define a View consisting of a `TableView` in the center of a BorderPane and a `Form` on the right side. We start by defining some properties for the form fields and the table itself so we can reference them later.

Then while we build the table we attach a listener to the selected item so we can call the `editPerson` function when the table selection changes. The `editPerson` function binds the properties of the selected person to the text fields in the form.

## Problems with our initial attempt

At first glance it might look OK, but when we dig deeper there are several issues.

### Manual binding

Every time the selection in the table changes, we have to rebind the data for the form fields manually. Apart from the added code and logic there is another huge problem with this: The data is updated for every change in the text fields, and the changes will even be reflected in the table. While this might look cool, it presents one big problem: What if the user doesn't want to save the changes? We have no way of rolling back, so to prevent this, we would have to skip the binding all together and manually extract the values from the text fields and create a new person object on save. In fact, this is a pattern found in many applications, even today. Implementing a "reset" button for this form would mean managing variables with the initial values and again assigning those values manually to the text fields.

### Tight coupling

When it's time to save the edited person, the save function has to extract the the selected item from the table again. For that to happen the save function has to know about the TableView. Alternatively it would have to know about the text fields like the `editPerson` function does, and manually extract the values to reconstruct a `Person` object.

## Introducing ViewModel

In the following example we have introduced ViewModel as a mediator between the `TableView` and the `Form`, but equally important it acts as a mediator between the data in the text fields and the data in the actual Person object. As you will see, the code is much shorter and easier to reason about.

```kotlin
class PersonEditor : View() {
    override val root = BorderPane()
    val persons = listOf(Person("John", "Manager"), Person("Jay", "Worker bee")).observable()
    val model = PersonModel(Person())

    init {
        title = "Person Editor"

        with(root) {
            center {
                tableview(persons) {
                    column("Name", Person::nameProperty)
                    column("Title", Person::titleProperty)

                    // Update the person inside the view model on selection change
                    model.rebindOnChange(this) { selectedPerson ->
                        person = selectedPerson ?: Person()
                    }
                }
            }

            right {
                form {
                    fieldset("Edit person") {
                        field("Name") {
                            textfield(model.name)
                        }
                        field("Title") {
                            textfield(model.title)
                        }
                       button("Save") {
                            disableProperty().bind(model.dirtyStateProperty().not())
                            setOnAction {
                                save()
                            }
                        }
                        button("Reset") {
                            setOnAction {
                                model.rollback()
                            }
                        }
                    }
                }
            }
        }
    }

    private fun save() {
        // Flush changes from the text fields into the model
        model.commit()

        // The edited person is contained in the model
        val person = model.person

        // A real application would persist the person here
        println("Saving ${person.name} / ${person.title}")
    }

}
```

This looks a lot better, but what's going on here?

We have introduced a subclass of ViewModel called `PersonModel`. The model holds a `Person` object and has properties for the `name` and `title` fields. We will discuss the model further after we have looked at the rest of the code.

Note that we hold no reference to the tableview or the the text fields. A part from a lot less code, the first big change is the way we update the person inside the model:

```kotlin
model.rebindOnChange(this) { selectedPerson ->
    person = selectedPerson ?: Person()
}
```

The `rebindOnChange` function takes the `TableView` as an argument and a function that will be called when the selection changes. This function is called on the model itself and has the selected person as it's single argument. We assign the selected person to the `person` property of the model, or to a new `Person` if the selection was empty. That way we ensure that there is always data for the model to present.

Now when we create the textfields, we bind the model properties directly to it:

```kotlin
field("Name") {
    textfield(model.name)
}
```

Even when the selection changes, the model properties remain the same, but the underlying data inside the properties are updated. We totally avoid the manual binding from our previous attempt.

Another big change in this version is that the data in the table does not update when we type into the text fields. This is because the model has exposed a copy of the properties from the person object and does not write back into the actual person object before we call `model.commit()`. This is exactly what we do in the `save` function. Once `commit` has been called, the data in the facade is flushed back into our person object and the table will now reflect our changes.

## Rollback

Since the model holds a reference to the actual person object we can can reset the text fields to reflect the actual data in our person object. We could add a reset button like this:

```kotlin
button("Reset") {
    setOnAction {
        model.rollback()
    }
}
```

When the button is pressed, any changes are discarded and the text fields show the actual person object data again.

## Dirty checking

The model has an observable property called `dirtyStateProperty`. This is a boolean observable property you can observe to enable or disable certain features. For example, we could easily disable the save button until there are actual changes. The updated save button would look like this:

```kotlin
button("Save") {
    disableProperty().bind(model.dirtyStateProperty().not())
    setOnAction {
        save()
    }
}
```

We connect the `disableProperty` of the button with an observable boolean property that is true when the model is not dirty (notice the `.not()` call to create a negated binding. Note that there is also a function called `isDirty` that returns a plain `Boolean` value representing the current state of the model.

There is also a `val` called `isDirty` which returns a boolean representing the dirty state.

## Dirty Properties

You can check if a property is dirty, meaning that it has been changed compared to the underlying source object value.

```kotlin
val nameWasChanged = model.isDirty(model.name)
```

There is also a shorter version that does the same:

```kotlin
val nameWasChange = model.name.isDirty
```

The shorter version is an extension `val` on `Property<T>` but it will only work for properties that are bound inside a ViewModel.

Correspondingly you will find `model.isNotDirty` properties as well.

## Extracting the source object value

To find the backing object value for a property you can call `model.backingValue(property)`.

## The PersonModel

You've probably been wondering about how the PersonModel is implemented. Well, here it is:

```kotlin
class PersonModel(var person: Person) : ViewModel() {
    val name = bind { person.nameProperty() }
    val title = bind { person.titleProperty() }
}
```

It can hold a `Person` object, and it has defined two weird looking properties called `name` and `title` via the `bind` delegate. I know, I know.. this looks weird, but it is a very good reason for it. The `{ person.nameProperty() }` parameter to the `bind` function is a function that returns a property. This returned property is examined by the ViewModel, and a new property of the same type is created and put into the `name` property of the ViewModel.

When we bind a text field to the `name` property of the model, only the copy is updated when you type into the text field. The ViewModel keeps track of which actual property belongs to which facade, and when you call `commit` the values from the facade are flushed into the actual backing property. On the flip side, when you call `rollback` the exact opposite happens: The actual property value is flushed into the facade.

The reason the actual property is wrapped in a function is that this makes it possible to change the `person` variable and then extract the property from that new person. You can read more about this below (rebinding).

## Supporting source objects that doesn't expose JavaFX properties

You probably wondered how to deal with domain objects that doesn't use JavaFX properties. Maybe you have a simple POJO with getters and setters, or normal kotlin `var` type properties. Since `ViewModel` requires JavaFX properties, TornadoFX comes with powerful wrappers that can turn any type of property into an observable JavaFX property. Here are some examples:

```kotlin
// Java POJO getter/setter property
class JavaPersonViewModel(person: JavaPerson) : ViewModel() {
    val name = bind { person.observable(JavaPerson::getName, JavaPerson::setName) }
}

// Kotlin var property
class PersonVarViewModel(person: Person) : ViewModel() {
    val name = bind { person.observable(Person::name) }
}
```

As you can see, it's very easy to convert any property type to an observable property. With the upcoming Kotlin 1.1 the above syntax will be further simplified for non-JavaFX based properties.

## Specific Property subtypes (IntegerProperty, BooleanProperty)

If you bind for example an `IntegerProperty`, the type of the facade property will look like `Property<Int>` but it is infact an `IntegerProperty` under the hood. If you need to access the special functions provided by `IntegerProperty`, you will have to cast the bind result:

```kotlin
val age = bind { person.ageProperty() } as IntegerProperty
```

The reason for this is an unfortunate shortcoming on the type system that prevents the compiler from differentiating between overloaded `bind` functions for these specific types, so the single `bind` function inside ViewModel inspects the property type and returns the best match, but unfortunately the return type signature has to be `Property<T>` for now.

## Rebinding

As you saw in the TableView example above, it is possible to change the domain object that is wrapped by the ViewModel. This test case from the framework tests sheds some more light on that:

```kotlin
@Test fun swap_source_object() {
    val person1 = Person("Person 1")
    val person2 = Person("Person 2")

    val model = PersonModel(person1)
    assertEquals(model.name, "Person 1")

    model.rebind {
        person = person2
    }

    assertEquals(model.name, "Person 2")
}
```

The test creates two `Person` objects and a `ViewModel`. The model is initialised with the first person object. It then checks that `model.name` corresponds to the name in `person1`. Now something weird happens:

```kotlin
model.rebind {
    person = person2
}
```

The code executed inside the `rebind` block above will be carried out and then all the properties of the model is updated with values from the new source object. This is actually analogous to writing:

```kotlin
model.person = person2
model.rollback()
```

The form you choose is up to you, but the first form makes sure you don't forget to call rebind. After `rebind` is called, the model is not dirty and all values will reflect the ones form the new source object or source objects. It's important to note that you can pass multiple source objects to a view model and update all or some of them as you see fit.

### Rebind listener

Our TableView example called the `rebindOnChange` function and passed in a TableView as the first argument. This made sure that rebind was called whenever the selection of the TableView changed. This is actually just a shortcut to another function with the same name that takes an observable and calls rebind whenever that observable changes. If you call this function you don't need to call rebind manually as long as you have an observable that represent the state change that should cause the model to rebind.

As you saw, TableView has special rebind support for the `selectionModel.selectedItemProperty`. If not for the special support, you would have to write it like this:

```kotlin
model.rebindOnChange(table.selectionModel.selectedItemProperty()) {
    person = it ?: Person() 
}
```

The above example is included to clarify how the `rebindOnChange` function works under the hood. For real use cases involving a TableView you should opt for the shorter version.