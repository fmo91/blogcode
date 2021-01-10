---
title: "Unidirectional Architectures and time traveling in Swift: I"
date: "2020-12-21T22:00:00.121Z"
---

Time traveling in Swift is possible. This is a series that describe how Unidirectional architectures can be applied in Swift and how we can do very impressive things by enforcing them.

This is the first article in the series, but before deep diving into Swift specifics. I'd like to give you an introduction to the general concepts that make all of this possible.

# What is time traveling?

Of course this is not my idea. Time Traveling has been discussed in the Javascript world [several times](https://www.youtube.com/watch?v=uvAXVMwHJXU&ab_channel=ReactEurope). However, there are not so many examples of it in Swift. 

When we talk about time traveling in the context of a Frontend application, [this is what we have in mind](https://youtu.be/8QNGrxOKVW8).

We can record user sessions, and then reproduce the actions, go back, go forward, and "travel" in time, because all the actions are reproducible.

# The main components

A word in the previous description is important: `Actions`. An `Action` is an object that describes something that happened in the system. 

Did the user change the text in a textfield? That is an `Action`.

Did the user tap the button to create a To Do? That is an `Action`.

Did the user complete a To Do? That is also an `Action`.

Given that, if we want to describe something that happened in the system, anything, we need to create an `Action` that describes the event.

![arch-001](https://dev-to-uploads.s3.amazonaws.com/i/cxhqio1124haq3up27br.png) 

—

In order to time travel, we need to be able to get a certain `State` for the system by applying the `Action` objects, sequentially. Nothing else can possibly affect or modify the `State`. If anything else was able to mutate the `State` other than an action, then we couldn't reproduce old states accordingly.

![unidirectional-005](https://dev-to-uploads.s3.amazonaws.com/i/vmbo9vw98n99vcqejocr.png) 

In the diagram above, I illustrate how a sequence of actions resulted in a state with an empty `text` attribute and an array with a single `todo` object that is completed. If we removed the `toggleTodo` action, we could apply all the other actions and get a state which is in turn the previous state. **It is an undo event.**

![unidirectional-006](https://dev-to-uploads.s3.amazonaws.com/i/qxtvf84x8xv5erzcd0fk.png) 

We can also store the deleted actions in a separate stack, so we can bring those back to the main actions stack and calculate the state again. **It is a redo event.**

![unidirectional-004](https://dev-to-uploads.s3.amazonaws.com/i/fuaxnw8gxoxqcxfofbh4.png) 

So, we apply a set of `Action` objects, and we get a resulting application `State`. So far, so good. But, we can think of actions as an `enum` with a number of cases with associated values, and the `State` of a struct or a class with all the values inside, and that's ok. However, something is missing: how is state mutated? Where is that mutation described?

# The missing piece: The reducer

A `reducer` is a pure function. That means that its output is only calculated based on its input. It is a function in the traditional mathematical sense of the concept. Each element in its domain has only one element in its image.  

![Untitled-001](https://dev-to-uploads.s3.amazonaws.com/i/2191mzcy0dfeaf1kzint.png) 

Image from [Math is fun](https://www.google.com/url?sa=i&url=https%3A%2F%2Fwww.mathsisfun.com%2Fsets%2Ffunction.html&psig=AOvVaw29Q5XLkHBd9oCYOmfZcTA_&ust=1608299865313000&source=images&cd=vfe&ved=0CA0QjhxqFwoTCPjAwYOW1e0CFQAAAAAdAAAAABAI)

The `reducer` is the only function which is allowed to perform mutations in the `State`. Ok, it doesn't mutate the `State`. It actually produces new states based on the previous `State` and an `Action`. It does so because it should only depend on its input. If it mutated the current state, it would also depend in a variable from the outside.

![unidirectional-008](https://dev-to-uploads.s3.amazonaws.com/i/scp4o657rif164cgz89z.png)
 
For each `Action` we get in the system, we send that action to the `reducer` along with the current `State`. The `reducer` returns the new `State`, so we store that value and we are prepared for the next `Action` that we will eventually get. 

![unidirectional-007](https://dev-to-uploads.s3.amazonaws.com/i/kx34772t9lzaan5iaail.png)
 
Due to the **functional purity** of the `reducer`, and given that all the events that could possibly mutate the `State` are expressed through an `Action`, each step is reproducible. We can travel in time by storing all the actions and calculate previous and next `State` values by applying all the `Action` values sequentially using the `reducer` function.

# f(State) = UI

For all of this to work, we need to enforce **functional purity** in another transformation: **the view should be a pure function of the application state**.

![unidirectional-009](https://dev-to-uploads.s3.amazonaws.com/i/xw12qh1wv5ix98mi6v4e.png)
 
Changes in the application states will produce modifications in the View to happen. This may sound as a lot of work, but keep in mind to things:

1. The power of programming depends more on the things you can't do than on the things you are able to do. Applying restrictions to the things you can do is a way of getting properties in your system that allow you to do this kind of things.
2. SwiftUI already does this for you for free. Views in SwiftUI are rendered depending only on the state you have on them and on your `ObservableObjects`. This is taking that approach one step forward, and centralizing all your state in a single, big State object that can be mutated only by a function called `reducer`.

# The store

There is another component that hasn't been discussed so far. The `Store` is an object, that can be implemented in SwiftUI as a `ObservableObject`, that holds our `State`, our `reducer`, and exposes a function called `dispatch(action:)`, which takes an `Action` and replaces the current `State` for the result of the `reducer` function.

![unidirectional-11_(1)](https://dev-to-uploads.s3.amazonaws.com/i/jqxk0pgke8jewg731lak.png)
 
# To sum up

As said, **The power of programming depends more on the things you can't do than on the things you are able to do.** If you enforce functional purity in your application, and express all mutations through actions, the state then becomes reproducible, you can go back and forward as you want, you can record user sessions, and many many many other things we havent covered in this article. 

In the next article, we will translate all of this in actual Swift/SwiftUI code, so you will get a more practical understanding on how all of this fits in a real app.

If you want to learn an iOS architecture that applies all of this and is production ready, I would HIGHLY recommend [The Composable Architecture](https://github.com/pointfreeco/swift-composable-architecture).