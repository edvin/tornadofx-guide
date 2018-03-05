# Snippets

### How to load an image to the imageview{...} in more effective way

This snippet covers 2 issues:

* How to get correct view size after image loading
* How to load an image to not affect to (do not to freeze) UI

The snippet below uses 4 techniques with timing each of them. So one can see that more effective way is to use URL as the constructor parameter along with explicit view resizing.

```java
import javafx.application.Application
import javafx.scene.image.Image
import javafx.stage.Stage
import tornadofx.*


/**
 * This is about how to load an image in more effective way.
 */
class LoadImageView : View() {

    override val root = vbox {

        run {
            // 1. Simple synchronous way via property
            println("-- load synchronously #1 -- ")
            val start = System.currentTimeMillis()
            imageview {
                image = Image("/big_image.png")
                println("loaded for ${System.currentTimeMillis() - start} msecs")
            }
            println("finished after ${System.currentTimeMillis() - start} msecs")
        }

        run {
            // 2. Simple synchronous way via constructor
            println("-- load synchronously #2 --")
            val start = System.currentTimeMillis()
            imageview("/big_image.png", lazyload = false) {
                println("loaded for ${System.currentTimeMillis() - start} msecs")
            }
            println("finished after ${System.currentTimeMillis() - start} msecs")
        }

        run {
            // 3. Asynchronous way through outer background task
            println("-- load asynchronously #1 -- ")
            val start = System.currentTimeMillis()
            imageview {
                runAsync {
                    image = Image("/big_image.png")
                    println("loaded for ${System.currentTimeMillis() - start} msecs")
                }
            }
            println("finished after ${System.currentTimeMillis() - start} msecs")
        }

        // Need between 2 async calls
        Thread.sleep(1000)

        run {
            // 4. Asynchronous way through lazy loading
            println("-- load asynchronously #2 -- ")
            val start = System.currentTimeMillis()
            imageview("/big_image.png") {
                setPrefSize(1920.0, 1080.0)
                println("loaded for ${System.currentTimeMillis() - start} msecs")
            }
            println("finished after ${System.currentTimeMillis() - start} msecs")
        }

// After you run you'll see something like this:
//
//            -- load synchronously #1 --
//            loaded for 217 msecs
//            finished after 218 msecs
//            -- load synchronously #2 --
//            loaded for 150 msecs
//            finished after 150 msecs
//            -- load asynchronously #1 --
//            finished after 75 msecs
//            loaded for 171 msecs
//            -- load asynchronously #2 --
//            loaded for 7 msecs
//            finished after 7 msecs
//
// So the winner is no.4: Asynchronous way through lazy loading

    }

}

class LoadImageApp : App(LoadImageView::class)

fun main(args: Array<String>) {
    launch<LoadImageApp>(args)
}
```
