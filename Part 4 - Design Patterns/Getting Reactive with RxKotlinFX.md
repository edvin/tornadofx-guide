#Reactive Programming with RxKotlinFX

If you have never experienced reactive programming before, you are in for a treat. It takes functional programming concepts to a whole new level, and creates resilient and simplified applications using push-based notifications. But these notifications are composable and can be transformed through hundreds of operators. RxJava uses Observables, which build on the `Observer` pattern and feel very similar to Java 8 Streams or Kotlin Sequences. They also work as  more elegant event buses. This puts events and data on a level playing field because Observables are "push-based" rather than "pull-based". This subtle difference opens up an entire realm of new capabilities that is making Rx increasingly a development standard.

Reactive programming works great with JavaFX, and while TornadoFX does not have any reactive programming built into it, you can use lightweight reactive JavaFX libraries like ReactFX or RxKotlinFX. 

[ReactFX](https://github.com/TomasMikula/ReactFX) is a great reactive API specifically built from scratch for JavaFX. it does not support asynchronous operations like RxJava does, but it is a solid library if you are only interested in reactive UI events. [RxKotlinFX](https://github.com/thomasnield/RxKotlinFX) is built on [RxJava](https://github.com/ReactiveX/RxJava) and [RxJavaFX](https://github.com/ReactiveX/RxJavaFX). It leverages all the powerful operators and ecosystem RxJava brings, and it supports asynchronous operations easily. We will also use [RxJava-JDBC](https://github.com/davidmoten/rxjava-jdbc) to demonstrate reactive database querying.

Please note that one of the authors (Thomas Nield) is the creator and maintainer of RxJavaFX and RxKotlinFX, so this chapter will use RxJava and friends as the reactive platform. You can apply most of the patterns here to ReactFX or any other reactive framework. 

