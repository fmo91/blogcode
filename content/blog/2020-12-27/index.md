---
title: "Unidirectional Architectures and time traveling in Swift: II"
date: "2020-12-27T22:00:00.121Z"
---

In the previous article, we have talked about how functional purity and an unidirectional architecture by using the Action-State-Reducer combo can give us incredible powers such as time traveling.

# Quick Recap

These were the main points of the previous article:

- All events that happen in an application can be described using `Action` values. The `Action` can be any kind of type, but for convenience, can think of `Action` as an enum.
- The application state is stored in a struct or class named `State`.
- We define a `Store`, which is an object that holds the application `State` , and a **pure** function called `reducer`.
- The `reducer` is a pure function (it calculates its output based only on its input, and generates no side effects), that based on the current `State` and an incoming `Action`, generates a new `State`. It is the ONLY part of the code where the `State` is modified.
- Whenever something happens in the `View`, it dispatches an `Action` to the `Store`. The `Store` executes the `reducer`, and updates its internal `State`. The `State` then is sent to the `View` so it can update accordingly.

![unidirectional-11_(100)](https://dev-to-uploads.s3.amazonaws.com/i/b5ww9s4ekbugxoqmnjy9.png) 

- However, we want to travel in time, to do so, we store all the `Action` sent to the store, and the initial `State`. If we want to **undo,** we just remove the last element in the `Action` array and compute the `State` from the beginning. Something similar can be done to **redo**.

# In code

Well, if you understood everything that was described in the last section, the code may look pretty straightforward to you.

Let's dive in.

## The State

Well, this is maybe the simplest component to grasp. The `State` must hold all the application state we want to use. In our case, a simple Todo app, this can be something like this:

```swift
struct AppState {
    let todos: [Todo]
    let todoText: String
}

struct Todo: Identifiable {
    let id: UUID
    let title: String
    let completed: Bool
}
```

For convienience, we will define a `default` or maybe `initial` state getter on our `AppState`

```swift
extension AppState {
    static var `default`: AppState {
        AppState(todos: [], todoText: "")
    }
}
```

Nice, so let's leave the `AppState` as it is at this point.

## The Action

We can define `Action` cases as an `enum`

```swift
enum Action {
    case todoTextChange(String)
    case createTodo(id: UUID)
    case toggleTodo(id: UUID)
}
```

Nice. We can modify the text input to create a new todo, create the todo using the `todoText` string we have in our `AppState`, or toggle the completion of some todo.

There is something here that might look strange. Why are we sending the `id` for the todo we are creating? Well, if the `reducer` has to create the `UUID` itself, and if it uses the `UUID` initializer, we would get a different `UUID` every time we run the `createTodo` action. In such a case, the `reducer` **IS NOT PURE**.

## The reducer

Ok, so we already have the `AppState` and the `Action` types. Let's go for the `appReducer`.

```swift
func appReducer(state: AppState, action: Action) -> AppState {
    switch action {
    case .todoTextChange(let newText):
				// ...
    case .createTodo(let id):
				// ...    
		case .toggleTodo(let id):
    		// ...
    }
}
```

So far, so good. We have our `appReducer` that takes the current `AppState` and an `Action` and returns a new `AppState`.

Let's see how it looks the `todoTextChange`:

```swift
func appReducer(state: AppState, action: Action) -> AppState {
    switch action {
    case .todoTextChange(let newText):
        return AppState(
            todos: state.todos,
            todoText: newText
        )
    case .createTodo(let id):
        // ...
    case .toggleTodo(let id):
        // ...
    }
}
```

Pretty straightforward. If we need to change the `todoText`, we just create a new `AppState`, reusing the `state.todos`, because we don't want to mutate them.

```swift
func appReducer(state: AppState, action: Action) -> AppState {
    switch action {
    case .todoTextChange(let newText):
        // ...
    case .createTodo(let id):
        let newTodo = Todo(
            id: id,
            title: state.todoText,
            completed: false
        )
        
        var currentTodos = state.todos
        currentTodos.insert(newTodo, at: 0)
        
        return AppState(
            todos: currentTodos,
            todoText: ""
        )
    case .toggleTodo(let id):
        // ...
    }
}
```

For the `createTodo` action, we first create a new `Todo` using the `id` we send in the action, and return a new `AppState` with the old todos and the new `Todo`, and clear the `todoText`.

```swift
func appReducer(state: AppState, action: Action) -> AppState {
    switch action {
    case .todoTextChange(let newText):
        // ...
    case .createTodo(let id):
        // ...
    case .toggleTodo(let id):
        var currentTodos = state.todos
        let index = currentTodos.firstIndex(where: { $0.id == id })!
        let todo = currentTodos[index]
        currentTodos[index] = Todo(
            id: id,
            title: todo.title,
            completed: !todo.completed
        )
        
        return AppState(
            todos: currentTodos,
            todoText: ""
        )
    }
}
```

And finally, for the `toggleTodo` we get the `Todo` with the `id` equal to the `id` we send in the `Action`, change its `completed` property, and finally send a new `AppState`.

So the complete reducer implementation is as follows:

```swift
func appReducer(state: AppState, action: Action) -> AppState {
    switch action {
    case .todoTextChange(let newText):
        return AppState(
            todos: state.todos,
            todoText: newText
        )
    case .createTodo(let id):
        let newTodo = Todo(
            id: id,
            title: state.todoText,
            completed: false
        )
        
        var currentTodos = state.todos
        currentTodos.insert(newTodo, at: 0)
        
        return AppState(
            todos: currentTodos,
            todoText: ""
        )
    case .toggleTodo(let id):
        var currentTodos = state.todos
        let index = currentTodos.firstIndex(where: { $0.id == id })!
        let todo = currentTodos[index]
        currentTodos[index] = Todo(
            id: id,
            title: todo.title,
            completed: !todo.completed
        )
        
        return AppState(
            todos: currentTodos,
            todoText: ""
        )
    }
}
```

## The Store

Maybe the store is the most difficult component in this implementation. Lets review the responsibilities the store holds in this example:

- It has to hold the `AppState` and the `appReducer`.
- It has to dispatch `Action` values.
- It has to notify the `View` whenever the `AppState` mutates.
- It has to **undo** and **redo**, at any point.

Let's go one by one:

### It has to hold the `AppState` and the `appReducer`.

For this one, we could just initialize the `Store` with those values.

```swift
final class Store {
    private(set) var state: AppState
    let reducer: (AppState, Action) -> AppState
    
    init(
        initialState: AppState,
        reducer: @escaping (AppState, Action) -> AppState
    ) {
        self.state = initialState
        self.reducer = reducer
    }
    
    static var `default`: Store {
        Store(
            initialState: .default,
            reducer: appReducer
        )
    }
}
```

We also added a `default` static getter so we can create the `Store` using the values we have just defined above.

### It has to dispatch `Action` values.

What does this mean? Well, we need a `dispatch` function, so the view can send an `Action` when something happens. 

```swift
final class Store {
		// ...
    func dispatch(_ action: Action) {
        state = reducer(state, action)
    }
		// ...
}
```

For now, let's leave this as simple as this. Whenever the `Store` dispatches an `Action`, it just executes the `reducer`, passing the current `State` and the received `Action`, and stores the obtained `State` in its internal state.

### It has to notify the `View` whenever the `AppState` mutates.

This will be used in `SwiftUI` in this example. How does an object notify the `View` that its internal state has changed? Exactly, conforming to `ObservableObject`, that is the simplest way of doing so, and we don't want to overcomplicate the things. Also, we need to make the `state` property, `@Published`.

```swift
final class Store: ObservableObject {
    @Published private(set) var state: AppState
    let reducer: (AppState, Action) -> AppState
    
    init(
        initialState: AppState,
        reducer: @escaping (AppState, Action) -> AppState
    ) {
        self.initialState = initialState
        self.state = initialState
        self.reducer = reducer
    }
    
    static var `default`: Store {
        Store(
            initialState: .default,
            reducer: appReducer
        )
    }
    
    func dispatch(_ action: Action) {
        state = reducer(state, action)
    }
}
```

This is how the `Store` looks so far. Nice.

This is the `View`:

```swift
import SwiftUI

struct CotentView: View {
		// 1
    @StateObject var store = Store.default
    
    var body: some View {
        VStack {
            HStack {
                TextField(
                    "Add to do",
										// 2
                    text: Binding<String>(
                        get: { return store.state.todoText },
                        set: { newTodoText in store.dispatch(.todoTextChange(newTodoText)) }
                    )
                )
                Button("Create") {
										// 3
                    store.dispatch(.createTodo(id: UUID()))
                }
            }
            List {
								// 4
                ForEach(store.state.todos) { todo in
                    Text("\(todo.title) \(todo.completed ? "DONE!" : "")")
                        .padding()
                        .onTapGesture(count: 1, perform: {
														// 3
                            store.dispatch(.toggleTodo(id: todo.id))
                        })
                }
            }
						HStack {
                Button("UNDO") {
										// 5
                    store.undo()
                }
                Spacer()
                Button("REDO") {
										// 5
                    store.redo()
                }
            }
        }
        .padding()
    }
}
```

This is not a tutorial of `SwiftUI`, but I want to note a couple of things here

1. We are storing the `store`, as a `@StateObject` inside the `View`.
2. Bindings are a bit unusual when we are using a unidirectional architecture. In this case, we need to define a getter and a setter for it. The getter just returns the `todoText` in the `store`. The setter dispatches an `Action` whenever it changes.
3. The `store` is used to `dispatch` actions when things happen in the `View`.
4. The `store` is also used to get the current state of the application.
5. The `undo` and `redo` methods in the `store` are yet to be defined, but the `View` is already set at this point.

### It has to **undo** and **redo**, at any point.

The last point. This can be done by:

- Storing an array in the `Store` with all of the `Action` that have been dispatched so far. Let's call this `appliedActions`.
- Storing the initial `AppState` in a variable called `initialState` so we can compute any subsequent `State` by applying an array of `Action` values using the `reducer`.
- Storing an array in the `Store` with all of the `Action` values that have been "undone", so we can "redo" them (by appending them again in the `appliedActions` array). Let's call this array `undoneActions`.

In code:

```swift
final class Store: ObservableObject {
    private let initialState: AppState
    private var appliedActions: [Action] = []
    private var undoneActions: [Action] = []
    @Published private(set) var state: AppState
    let reducer: (AppState, Action) -> AppState
    
    init(
        initialState: AppState,
        reducer: @escaping (AppState, Action) -> AppState
    ) {
        self.initialState = initialState
        self.state = initialState
        self.reducer = reducer
    }
    
    static var `default`: Store {
        Store(
            initialState: .default,
            reducer: appReducer
        )
    }
    
    func dispatch(_ action: Action) {
        appliedActions.append(action)
        undoneActions = []
        state = reducer(state, action)
    }
    
    func redo() {
        state = initialState
        if let undoneAction = undoneActions.popLast() {
            appliedActions.append(undoneAction)
            for action in appliedActions {
                state = reducer(state: state, action: action)
            }
        }
    }
    
    func undo() {
        state = initialState
        if let undoneAction = appliedActions.popLast() {
            undoneActions.append(undoneAction)
            for action in appliedActions {
                state = reducer(state: state, action: action)
            }
        }
    }
}
```

That is the final implementation for the `Store`. Note that for the `redo` and `undo` we use the `reducer` to compute all the actions again, and we play with the `appliedActions` and `undoneActions` arrays to move `Action` values from one place to another, achieving in this way, time traveling in Swift.

# Conclusion

To sum up, this has been a quick introduction to Redux (unidirectional architectures) in Swift, completely from scratch. There A LOT of missing pieces in this implementation, of course. We didn't cover testing, and we don't have any way of modeling side effects, like network requests. I highly recommend [The Composable Architecture](https://github.com/pointfreeco/swift-composable-architecture) if you want to use this kind of architectures in production. As far as I have use this architecture, it worked fine to me. However, I have to say this is still early days in SwiftUI architectures and the landscape might change. My last piece of advice here is: remember Software Engineering is about taking decisions balancing tradeoffs. Keep curiosity but think everything carefully.