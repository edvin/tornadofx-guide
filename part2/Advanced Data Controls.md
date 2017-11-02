# Advanced Data Controls

This section will primarily address more advanced features you can leverage with data controls, particulary with the `TableView` and `ListView`.


##  TableView Advanced Column Resizing

The SmartResize policy brings the ability to intuitively resize columns by providing sensible defaults combined with powerful and dynamic configuration options.

To apply the resize policy to a `TableView` we configure the `columnResizePolicy`. For this exercise we will use a list of hotel rooms. This is our initial table with the `SmartResize` policy activated:

```kotlin
tableview(rooms) {
    column("#", Room::id)
    column("Number", Room::number)
    column("Type", Room::type)
    column("Bed", Room::bed)

    columnResizePolicy = SmartResize.POLICY
}
```

Here is a picture of the table with the SmartResize policy activated (Figure 5.7):

**Figure 13.1**

![](http://i.imgur.com/chugPrR.png)

The default settings gave each column the space it needs based on its content, and gave the remaining width to the last column. When you resize a column by dragging the divider between column headers, only the column immediately to the right will be affected, which avoids pushing the columns to the right outside the viewport of the `TableView`.

While this often presents a pleasant default, there is a lot more we can do to improve the user experience in this particular case. It is evident that our table did not need the full 800 pixels it was provided, but it gives us a nice chance to elaborate on the configuration options of the `SmartResize` policy.

The bed column is way too big, and it seems more sensible to give the extra space to the **Type** column, since it might contain arbitrary long descriptions of the room. To give the extra space to the **Type** column, we change its column definition (Figure 5.8):

```kotlin
column("Type", Room::type).remainingWidth()
```

**Figure 13.2**

![](http://i.imgur.com/fHgkqnZ.png)

Now it is apparent the **Bed** column looks cramped, being pushed all the way to the left. We configure it to keep its desired width based on the content plus 50 pixels padding:

```kotlin
column("Bed", Room:bed").contentWidth(padding = 50.0)
```

The result is a much more pleasant visual impression (Figure 5.9) :

**Figure 13.3**

![](http://i.imgur.com/O0VeONz.png)

This fine-tuning may not seem like a big deal, but it means a lot to people who are forced to stare at your software all day! It is the little things that make software pleasant to use.

If the user increases the width of the **Number** column, the **Type** column will gradually decrease in width, until it reaches its default width of 10 pixels (the JavaFX default). After that, the **Bed** column must start giving away its space. We don't ever want the **Bed** column to be smaller that what we configured, so we tell it to use its content-based width plus the padding we added as its minimum width:

```kotlin
column("Bed", Room:bed").contentWidth(padding = 50.0, useAsMin = true)
```

Trying to decrease the **Bed** column either by explicitly expanding the **Type** column or implicitly by expanding the **Number** column will simply be denied by the resize policy. It is worth noting that there is also a `useAsMax` choice for the `contentWidth` resize type. This would effectively result in a hard-coded, unresizable column, based on the required content width plus any configured padding. This would be a good policy for the **\#** column:

```kotlin
column("#", Room::id).contentWidth(useAsMin = true, useAsMax = true)
```

The rest of the examples will probably not benefit the user, but there are still other options at your disposal. Try to make the **Number** column 25% of the total table width:

```kotlin
column("Number", Room::number).pctWidth(25.0)
```

When you resize the `TableView`, the **Number** column will gradually expand to keep up with our 25% width requirement, while the **Type** column gets the remaining extra space.

**Figure 13.4**

![](http://i.imgur.com/NP04XNd.png)

An alternative approach to percentage width is to specify a weight. This time we add weights to both **Number** and **Type**:

```kotlin
column("Number", Room::number).weigthedWidth(1.0)
column("Type", Room::type).weigthedWidth(3.0)
```

The two weighted columns share the remaining space after the other columns have received their fair share. Since the **Type** column has a weight that is three times bigger than the **Number** column, its size will be three times bigger as well. This will be reevaluated as the `TableView` itself is resized.

**Figure 13.5**

![](http://i.imgur.com/lm47oxU.png)

This setting will make sure we keep the mentioned ratio between the two columns, but it might become problematic if the `TableView` is resized to be very small. The the **Number** column would not have space to show all of its content, so we guard against that by specifying that it should never grow below the space it needs to show its content, plus some padding, for good measure:

```kotlin
column("Number", Room::number).weigthedWidth(1.0, minContentWidth = true, padding = 10.0)
```

This makes sure our table behaves nicely also under constrained width conditions.

#### Dynamic content resizing

Since some of the resizing modes are based on the actual content of the columns, they might need to be reevaluated even when the table or it's columns aren't resized. For example, if you add or remove content items from the backing list, the required content measurements might need to be updated. For this you can call the `requestResize` function after you have manipulated the items:

```kotlin
SmartResize.POLICY.requestResize(tableView)
```

In fact, you can ask the TableView to ask the policy for you:

```kotlin
tableView.requestResize()
```

#### Statically setting the content width

In most cases you probably want to configure your column widths based on either the total available space or the content of the columns. In some cases you might want to configure a specific width, that that can be done with the `prefWidth` function:

```kotlin
column("Bed", Room::bed).prefWidth(200.0)
```

A column with a preferred width can be resized, so to make it non-resizable, use the `fixedWidth` function instead:

```kotlin
column("Bed", Room::bed).fixedWidth(200.0)
```

When you hard-code the width of the columns you will most likely end up with some extra space. This space will be awarded to the right most resizable column, unless you specify `remainingWidth()` for one or more column. In that case, these columns will divide the extra space between them.

In the case where not all columns can be afforded their preferred width, all resizable columns must give away some of their space, but the `SmartResize` Policy makes sure that the column with the biggest reduction potential will give away its space first. The reduction potential is the difference between the current width of the column and its defined minimum width.

## Custom Cell Formatting in ListView

Even though the default look of a `ListView` is rather boring (because it calls `toString()` and renders it as text) you can modify it so that every cell is a custom `Node` of your choosing. By calling `cellCache()`, TornadoFX provides a convenient way to override what kind of `Node` is returned for each item in your list (Figure 5.2).

```kotlin
class MyView: View() {

    val persons = listOf(
            Person("John Marlow", LocalDate.of(1982,11,2)),
            Person("Samantha James", LocalDate.of(1973,2,4))
    ).observable()

    override val root = listview(persons) {
        cellFormat {
            graphic = cache {
                form {
                    fieldset {
                        field("Name") {
                            label(it.name)
                        }
                        field("Birthday") {
                            label(it.birthday.toString())
                        }
                        label("${it.age} years old") {
                            alignment = Pos.CENTER_RIGHT
                            style {
                                fontSize = 22.px
                                fontWeight = FontWeight.BOLD
                            }
                        }
                    }
                }
            }
        }
    }
}

class Person(val name: String, val birthday: LocalDate) {
    val age: Int get() = Period.between(birthday, LocalDate.now()).years
}
```

**Figure 5.2** - A custom cell rendering for `ListView` ![](http://i.imgur.com/o3r2TuR.png)

The `cellFormat` function lets you configure the `text` and/or `graphic` property of the cell whenever it comes into view on the screen. The cells themselves are reused, but whenever the `ListView` asks the cell to update its content, the `cellFormat` function is called. In our example we only assign to `graphic`, but if you just want to change the string representation you should assign it to `text`. It is completely legitimate to assign it to both `text` and `graphic`. The values will automatically be cleared by the `cellFormat` function when a certain list cell is not showing an active item.

Note that assigning new nodes to the `graphic` property every time the list cell is asked to update can be expensive. It might
be fine for many use cases, but for heavy node graphs, or node graphs where you utilize binding towards the UI components inside the cell, you should cache the resulting node so the Node graph will only be created once per node. This is done using the `cache` wrapper in the above example.

#### Assign If Null

If you have a reason for wanting to recreate the graphic property for a list cell, you can use the `assignIfNull` helper,
which will assign a value to any given property if the property doesn't already contain a value. This will make sure that
you avoid creating new nodes if `updateItem` is called on a cell that already has a graphic property assigned.

```kotlin
cellFormat {
    graphicProperty().assignIfNull {
        label("Hello")
    }
}
```

### ListCellFragment

The `ListCellFragment` is a special fragment which can help you manage `ListView` cells. It extends `Fragment`, and
includes some extra `ListView` specific fields and helpers. You never instantiate these fragments manually, instead you
instruct the `ListView` to create them as needed. There is a one-to-one correlation between `ListCell` and `ListCellFragment` instances. Only one `ListCellFragment` instance will over its lifecycle be used to represent different items.

To understand how this works, let's consider a manually implemented `ListCell`, essentially the way you would do in vanilla JavaFX. The `updateItem` function will be called when the `ListCell` should represent a new item, no item, or just an update to the same item. When you use a `ListCellFragment`, you do not need to implement something akin to `updateItem`, but the `itemProperty` inside it will update to represent the new item automatically. You can listen to changes to the `itemProperty`, or better yet, bind it directly to a `ViewModel`. That way your UI can bind directly to the `ViewModel` and no longer need to care about changes to the underlying item.

Let's recreate the form from the `cellFormat` example using a `ListCellFragment`. We need a `ViewModel` which we will
call `PersonModel` (Please see the `Editing Models and Validation` chapter for a full explanation of the `ViewModel`) For now,
just imagine that the `ViewModel` acts as a proxy for an underlying `Person`, and that the `Person` can be changed while the
observable values in the `ViewModel` remain the same. When we have created our `PersonCellFragment`, we need to configure
the `ListView` to use it:

```kotlin
listview(personlist) {
    cellFragment(PersonCellFragment::class)
}
```

Now comes the `ListCellFragment` itself.

```kotlin
class PersonListFragment : ListCellFragment<Person>() {
    val person = PersonModel().bindTo(this)

    override val root = form {
        fieldset {
            field("Name") {
                label(person.name)
            }
            field("Birthday") {
                label(person.birthday)
            }
            label(stringBinding(person.age) { "$value years old" }) {
                alignment = Pos.CENTER_RIGHT
                style {
                    fontSize = 22.px
                    fontWeight = FontWeight.BOLD
                }
            }
        }
    }
}
```

Because this Fragment will be reused to represent different list items, the easiest approach is to bind the ui elements to the ViewModel's properties.

The `name` and `birthday` properties are bound directly to the labels inside the fields. The age string in the last label needs to be constructed using a `stringBinding`to make sure it updates when the item changes.

While this might seem like slightly more work than the `cellFormat` example, this approach makes it possible to leverage
everything the Fragment class has to offer. It also forces you to define the cell node graph outside of the builder hierarchy,
 which improves refactoring possibilities and enables code reuse.

#### Additional helpers and editing support

The `ListCellFragment` also have some other helper properties. They include the `cellProperty` which will
update whenever the underlying cell changes and the `editingProperty`, which will tell you if this the underlying list cell
is in editing mode. There are also editing helper functions called `startEdit`, `commitEdit`, `cancelEdit` plus an `onEdit`
callback. The `ListCellFragment` makes it trivial to utilize the existing editing capabilites of the `ListView`. A complete example
can be seen in the [TodoMVC](https://github.com/edvin/todomvc) demo application.
