#Getting Reactive with RxKotlinFX

If you have never experienced reactive programming before, you are in for a treat. It takes functional programming concepts to a whole new level to create resilient and simplified applications using push-based notifications. But these notifications are composable and can be transformed through hundreds of operators. RxJava uses Observables, and they feel very similar to Java 8 Streams or Kotlin Sequences. Interestingly, they also work as event buses, This puts events and data on a level playing field, because Observables are "push-based" rather than "pull-based". This subtle difference opens up an entire realm of new capabilities that is making Rx increasingly a standard.

Because Observables treat events and data the same, reactive programming works wonderfully with JavaFX. TornadoFX does not have any reactive programming built into it, but you can use lightweight reactive JavaFX libraries like ReactFX or RxKotlinFX. Although ReactFX is a great reactive API, it does not support asynchronous operations like RxJava does. RxKotlinFX is built on RxJava and leverages all the powerful operators and ecosystem it brings. You can apply most of the patterns here to ReactFX if you choose to use it. 

##A Brief History

Reactive programming was conceptualized  well before it got mainstream usage. In the late 2000's Erik Meijer developed a practical reactive framework called Rx.NET. In the next few years, Rx ("reactive extensions") 