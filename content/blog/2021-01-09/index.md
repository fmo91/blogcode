---
title: "Custom Containers in SwiftUI"
date: "2021-01-09T22:00:00.121Z"
---

# The Container

We call containers in SwiftUI to those views which can render other views passed by argument. Containers are central to SwiftUI. As soon as you start learning SwiftUI, you use `VStack`, `HStack`, `List`, etc.

The main idea behind a container is this:

```swift
struct Container<Content: View>: View {
	private let builder: () -> Content

	init(@ViewBuilder _ builder: @escaping () -> Content) {
		self.builder = builder
	}

	var body: some View {
		builder()
	}
}
```

This container, in particular, does absolutely nothing. I mean, this:

```swift
struct ContentView: View {
    var body: some View {
        Text("Some text")
            .padding()
    }
}
```

Is exactly the same as this:

```swift
struct ContentView: View {
    var body: some View {
        Container {
            Text("Some text")
                .padding()
        }
    }
}
```

Both of them will generate the same layout:

![snp-01](https://dev-to-uploads.s3.amazonaws.com/i/7kwcmd4u73yjk0d81o4l.png)
  

However, this is the base for some interesting possibilities.

# @ViewBuilder

So you might be thinking, ok, but why is that `@ViewBuilder` property wrapper needed?

Well, if you didn't have the `@ViewBuilder` property wrapper, in some Container like this:

```swift
struct Container<Content: View>: View {
    private let builder: () -> Content

    init(_ builder: @escaping () -> Content) {
        self.builder = builder
    }

    var body: some View {
        builder()
    }
}
```

Everything would work fine for just one `View`. The problem will come when we try to render something like this, because the closure argument in the `Container` should return a single view.

```swift
struct ContentView: View {
    var body: some View {
        Container {
            Text("Some text")
                .padding()
            Text("Some text")
                .padding()
            Text("Some text")
                .padding()
        }
    }
}
```

Adding `@ViewBuilder` in the closure, as shown at the beginning will fix this.

# Use Cases

## Layout

I'll mention two uses of custom Containers. The first one is **layout**. We can send views to a container and let the container arrange them as needed.

For instance:

```swift
struct TopArrangementContainer<Content: View>: View {
    private let builder: () -> Content

    init(@ViewBuilder _ builder: @escaping () -> Content) {
        self.builder = builder
    }

    var body: some View {
        VStack {
            builder()
            Spacer()
        }
    }
}

struct ContentView: View {
    var body: some View {
        TopArrangementContainer {
            Text("Some text")
                .padding()
            Text("Some text")
                .padding()
            Text("Some text")
                .padding()
        }
    }
}
```

This would generate this view:

![snp-02](https://dev-to-uploads.s3.amazonaws.com/i/8q80km4seedhnpgffmuu.png)

Of course there are many more interesting layouts you can apply, but I just wanted to show you one of them.

## Data Fetching

**Render Props** is a concept from React ([https://reactjs.org/docs/render-props.html](https://reactjs.org/docs/render-props.html)). In React, you can basically send a closure to a Container, that takes some parameter and builds a View based on it.

This can be useful in many situations. `GeometryReader` is a very good example of this design pattern. 

Let's do something similar but for data fetching. Imagine we want to fetch users from JSONPlaceholder: [https://jsonplaceholder.typicode.com/users](https://jsonplaceholder.typicode.com/users) 

So we define a model and an `ObservableObject` to use as a view model.

```swift
struct User: Codable, Identifiable {
    let id: Int
    let name: String?
}

final class UsersViewModel: ObservableObject {
    @Published var users: [User] = []
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        fetch()
    }
    
    private func fetch() {
        URLSession.shared
            .dataTaskPublisher(for: URL(string: "https://jsonplaceholder.typicode.com/users")!)
            .map(\.data)
            .decode(type: [User].self, decoder: JSONDecoder())
            .replaceError(with: [])
            .assign(to: \.users, on: self)
            .store(in: &cancellables)
    }
}
```

Pretty straightforward. The `User` struct is the model that will hold the data we fetch from the API, and the `UsersViewModel` is the class that will fetch the users and store them in a `@Published` variable. The important thing about `UsersViewModel` is that we can use it as a `@StateObject`, for example, in our `Container`. Let's do it:

```swift
struct UsersProvider<Content: View>: View {
    @StateObject private var viewModel = UsersViewModel()
    private let builder: ([User]) -> Content

    init(@ViewBuilder _ builder: @escaping ([User]) -> Content) {
        self.builder = builder
    }

    var body: some View {
		    builder(viewModel.users)
    }
}
```

Some important things to highlight here:

- This Container is now called `UsersProvider`, since this name describes a bit better what it does.
- We are using the `UsersViewModel` as a `@StateObject` here.
- The `builder` closure now takes the `[User]` array as a parameter. So whenever the users in the `UsersViewModel` change, it will trigger a render in the parent `View`.

In the `ContentView` can be refactored to this:

```swift
struct ContentView: View {
    var body: some View {
        UsersProvider { users in
            List(users) { user in
                Text(user.name ?? "")
                    .padding()
            }
        }
    }
}
```

So we can decide how to render the users we get from the API.

# Summary

This has been a quick introduction to custom containers in SwiftUI. I'm pretty sure there are many many other interesting use cases for this pattern that will appear over time, but the Layout and Data Fetching use cases are very interesting and useful ones.