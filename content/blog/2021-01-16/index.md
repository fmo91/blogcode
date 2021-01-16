---
title: Persistence with Core Data and SwiftUI
date: "2021-01-16T22:00:00.284Z"
---

I have worked with Core Data long time ago and left with a bad impression about the framework. I've never used it again (started using Realm short after it). 

However, it seems that something changed. I'm not sure if Core Data became much better (`NSPersistenceContainer` has been added after I stopped using Core Data), or if I leveled up as a developer. The thing is that I used it for a project recently and it was a pleasure to work with.

In this article, I will show you how you can start using Core Data, and how you can enjoy using it most of the time. Of course this is a brief introduction. If you want more information about how to correctly use Core Data, I would highly recommend reading Donny Wals' Practical Core Data book [https://gumroad.com/l/practical-core-data](https://gumroad.com/l/practical-core-data) . I bought it and it has been worth the money spent.

# What is Core Data?

Core Data is not an ORM. Really, it is not an ORM. Core Data is a graph-based optionally persisted model framework. Core Data can take care of your model layer, it can hold your entities, so you can ask Core Data for sorted, or filtered sets of entities that you need in some point of your app execution. 

Core Data can store your data in a SQLite database, or it can have your data in-memory, or even synchronized with CloudKit.

# An Example

I'll give you an introduction throughout this post on how to start developing apps in SwiftUI using Core Data.

The app we'll be building will let us create many ToDo lists, complete, and delete them. Really typical.

If you prefer to just go over the code and learn it the hard way, here is the Github repo: [https://github.com/fmo91/TodoListsSwiftUI](https://github.com/fmo91/TodoListsSwiftUI) 

# The Stack

The Core Data Stack is composed of objects which interact between them to persist entities. The main components of the Core Data Stack are:

- `NSManagedObjectModel`: This class represents the descriptions for the entities you'll persist in your apps. Most of times, you'll define those entities using the Xcode visual editor, which will create a file with extension `.xcdatamodeld`.
- `NSPersistentStoreCoordinator`: The persistent store coordinator will interface with the actual `Persistent Stores`, such as the SQLite databases. This is the class that interface between your app and the actual persistent stores. It needs a `NSManagedObjectModel` to work.
- `NSManagedObjectContext`: **The most important class in your Core Data Stack**. Or, at least, the class you'll interact the most during your app development. It works as a scratchpad where you can make modifications to your `NSManagedObject` entities and then save the context to reflect the changes in the persistent stores. It needs a `NSPersistentStoreCoordinator` to work.

![CoreData-01](https://dev-to-uploads.s3.amazonaws.com/i/lq6lwbzs2edb9pt2b07r.png) 

This is how it looks in code:

```swift
private lazy var storeCoordinator: NSPersistentStoreCoordinator = {
    let coordinator = NSPersistentStoreCoordinator(managedObjectModel: model)
    let url = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first!.appendingPathComponent("TodoListsSwiftUI.sqlite")
    try! coordinator.addPersistentStore(ofType: NSSQLiteStoreType,
                                        configurationName: nil,
                                        at: url,
                                        options: nil)
    return coordinator
}()

private lazy var model: NSManagedObjectModel = {
    let url = Bundle.main.url(forResource: "TodoListsSwiftUI", withExtension: "momd")!
    let model = NSManagedObjectModel(contentsOf: url)
    return model!
}()

private(set) lazy var context: NSManagedObjectContext = {
    let context = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
    context.persistentStoreCoordinator = storeCoordinator
    return context
}()
```

However, since iOS 10, we have another class that abstracts most of this complexity from us. `NSPersistentContainer`, which holds all these classes and exposes a `viewContext` property which is a `NSManagedObjectContext` you can use to generate `NSManagedObject` entities, save, delete and update them. And see how easier it is to create the stack:

```swift
private lazy var persistentContainer: NSPersistentContainer = {
    let container = NSPersistentContainer(name: "TodoListsSwiftUI")
    container.loadPersistentStores(completionHandler: { _, error in
        if let error = error {
            fatalError("An error ocurred while instantiating persistentContainer: \(error.localizedDescription)")
        }
    })
    return container
}()
```

The only thing that should change between this code and your app's code is the name you send to the persistent container init. That name is the name of the `.xcdatamodeld` file where you define your entities, and relationships. The `.xcdatamodeld` file is normally named after your project.

All of the information I described in this section is important in order to know how the Core Data stack is structured. However, when you create a new project in Xcode, it gives you the option to start the project with Core Data. By enabling that option, Xcode will generate an `NSPersistentContainer` and make it visible for your app code.

# The model editor

So, you have created a Xcode project with Core Data, or you have initialized a new Core Data stack in an existing project. I recommend doing so in a class that holds your `NSPersistentContainer`, like in this example:

```swift
final class PersistenceProvider {
    let persistentContainer: NSPersistentContainer
    var context: NSManagedObjectContext { persistentContainer.viewContext }
    
    static let `default`: PersistenceProvider = PersistenceProvider()
    init() {
        persistentContainer = NSPersistentContainer(name: "TodoListsSwiftUI")
        
        persistentContainer.loadPersistentStores { (_, error) in
            if let error = error {
                fatalError("Failed loading persistent stores with error: \(error.localizedDescription)")
            }
        }
    }
}
```

You'll also need a `.xcdatamodeld`. You can create it by creating a new Data Model in Xcode:

![cd-000](https://dev-to-uploads.s3.amazonaws.com/i/c8b4phvca4uo8whyx0wh.png) 

If you open the Data Model in Xcode, you'll see an editor where you can create Entities, and add and configure Attributes in your Entities and Relationships among your Entities.

Create two entities: `Todo` and `TodoList`. 

In `Todo`, add `title`, `creationDate` and `completed`, as non-optional properties, as shown in the image:

![cd-001](https://dev-to-uploads.s3.amazonaws.com/i/wmzhxwh1qkbvtzo9gvgq.png) 

In `TodoList`, add `title` and `creationDate`, as non-optional properties, as shown in the image:

![cd-002](https://dev-to-uploads.s3.amazonaws.com/i/dwae1fwd562zrpzq9htk.png) 

There should be a relationship between `Todo` and `TodoList`, since a `TodoList` may have any number of `Todo` objects related to it.

So, as also shown in the images, create a `list` relationship in `Todo` with destination equal to `TodoList` and a `todos` relationship in `TodoList` with destination equal to `Todo` and the inverse equal to `list`. This way, we have the counterpart of the `list` relationship.

Relationships may be To One or To Many. If you select a Relationship and inspect its properties in the editor:

![cd-003](https://dev-to-uploads.s3.amazonaws.com/i/5n9bx7c9tt5u4w6c4wfb.png) 

You'll notice a `type` property. In the `todos` relationship set the `type` as To Many, because a list may have many `Todo` entities related to it. In the `list` relationship, set the `type` as To One, because a `Todo` may only be included in a single list.

There is also another important property you should set in the relationships. Without entering in further details, I recommend setting the `Delete Rule` in the `todos` relationship as Cascade. By doing so, you ensure whenever you delete a list, all its todos entities will also be deleted.

The default Codegen setting is `Class Definition`, which means that Xcode will generate classes for all these entities whenever the project is built. You don't have to manually create anything.

# Operations in Core Data

So far, so good. We have initialized the Stack in a class named `PersistenceProvider`, with the most useful class in it, the `NSManagedObjectContext` being hold by the `NSPersistentContainer`.

The `NSManagedObjectContext` can be used for all the operations we need to do from the app that would impact in the persistent stores.

For all the operations you do in Core Data, I recommend creating classes that will take care of them. In this example app, I created extensions for our `PersistenceProvider` helper class, with methods to create, read, update or delete entities.

## Create

To create a new entity, you just instantiate the object for that entity sending the context to its constructor and then save the context. As simple as it sounds:

```swift
@discardableResult
func createList(with title: String) -> TodoList {
    let list = TodoList(context: context)
    list.title = title
    list.creationDate = Date()
    try? context.save()
    return list
}
```

To create a `Todo` for this `TodoList`, it is as simple as this:

```swift
@discardableResult
func createTodo(with title: String, in list: TodoList) -> Todo {
    let todo = Todo(context: context)
    todo.title = title
    todo.creationDate = Date()
    list.addToTodos(todo)
    try? context.save()
    return todo
}
```

The only maybe weird part is the `addToTodos` method that we call on `list`. This is also generated by Xcode. It adds the `Todo` to the relationship.

## Read

`NSFetchRequest` is a class that represents a query. Fetch requests can be created using a static method in the `NSManagedObject` subclasses, and can include a predicate, which is an object that describes conditions on the query, and sort descriptors, to define the order in which the results will be returned (otherwise, they will be got in random order):

```swift
var allListsRequest: NSFetchRequest<TodoList> {
    let request: NSFetchRequest<TodoList> = TodoList.fetchRequest()
    request.sortDescriptors = [
				NSSortDescriptor(
						keyPath: \TodoList.creationDate, 
						ascending: false
				)
		]
    return request
}
```

This is the request for getting all `TodoList` entities. See I'm not setting any predicate for this fetch request. Instead, I'm requesting all the `TodoList` that are in the store.

If I need all the `Todo` for a list, this is the fetch request I'd set:

```swift
func todosRequest(for list: TodoList) -> NSFetchRequest<Todo> {
    let request: NSFetchRequest<Todo> = Todo.fetchRequest()
    request.predicate = NSPredicate(format: "%K == %@", #keyPath(Todo.list), list)
    request.sortDescriptors = [
        NSSortDescriptor(keyPath: \Todo.completed, ascending: true),
        NSSortDescriptor(keyPath: \Todo.creationDate, ascending: false)
    ]
    return request
}
```

This is a bit more complex, since I'm adding a predicate. The `NSPredicate` takes a `format`, for which I send a `keyPath` referencing the `list` property.

Also, we're using two `NSSortDescriptor`. One for the `completed` property, and the other for the `creationDate`, so we know that the most recently created entity will be at the beginning of the result array.

When we use many `NSSortDescriptor`, they must be written in order of importance. So in this case, it will sort the results using `completed`, and then using `creationDate`.

To execute a request, you just call the method `fetch` in the managed object context:

```swift
let lists = try! context.fetch(allListsRequest)
```

## Update

Updating an entity in CoreData is as simple as setting new values to its properties and then saving the context:

```swift
func toggle(_ todo: Todo) {
    todo.completed.toggle()
    try? context.save()
}
```

## Delete

For deleting an entity you call the method `delete` in the context and then save the context:

```swift
func delete(_ lists: [TodoList]) {
    for list in lists {
        context.delete(list)
    }
    try? context.save()
}

func delete(_ todos: [Todo]) {
    for todo in todos {
        context.delete(todo)
    }
    try? context.save()
}
```

# Adding the context to the environment in SwiftUI

In order to make the `NSManagedObjectContext` available from all the application, you need to inject it in the initial view for your app using the `environment` modifier, injecting the context for the `\.managedObjectContext` key, like this:

```swift
import SwiftUI

@main
struct TodoListsSwiftUIApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(
										\.managedObjectContext, 
										PersistenceProvider.default.context
								)
        }
    }
}
```

# @FetchRequest

Inside your views, you can set a property as a `@FetchRequest`, so you can add relate a property in your view to a request to Core Data. It will trigger the fetch request, and in case the underlying data storage changes, the fetch request will trigger again, making the view re-render.

Let's see the full example of the list of `Todo`:

```swift
import SwiftUI

struct TodoListsListView: View {
    @FetchRequest(fetchRequest: PersistenceProvider.default.allListsRequest) 
		var allLists: FetchedResults<TodoList>
    
    var body: some View {
        VStack {
            List {
                ForEach(allLists) { list in
                    NavigationLink(destination: TodoListDetailView(list: list)) {
                        Text(list.title ?? "")
                            .padding()
                    }
                }
                .onDelete(perform: { indexSet in
                    PersistenceProvider.default.delete(allLists.get(indexSet))
                })
            }
            
            TextInputView(
                title: "Add a new list",
                actionTitle: "Add",
                onCreate: { listText in
                    PersistenceProvider.default.createList(with: listText)
                }
            )
        }
        .navigationTitle("Lists")
    }
}
```

In this example, whenever the `allLists` property changes, the body will be recalculated.

**Note**: `TextInputView` is just a helper component, it doesn't do anything related to Core Data itself. Here is the code:

```swift
import SwiftUI

struct TextInputView: View {
    @State private var text = ""
    
    let title: String
    let actionTitle: String
    let onCreate: (String) -> Void
    
    var body: some View {
        HStack {
            TextField(title, text: $text)
            Button(actionTitle, action: {
                if text.isEmpty { return }
                
                onCreate(text)
                text = ""
            })
        }.padding()
    }
}
```

# FetchRequest

Now, imagine if you had to do some query depending on a parameter you send to the View using its `init` method. The `@FetchRequest` property wrapper won't be useful in that case. We can create a `FetchRequest` object using its `init`. This is the full implementation for `TodoListDetailView`:

```swift
import SwiftUI

struct TodoListDetailView: View {
    let list: TodoList
    var todos: FetchRequest<Todo>
    
    init(list: TodoList) {
        self.list = list
        self.todos = FetchRequest<Todo>(fetchRequest: PersistenceProvider.default.todosRequest(for: list))
    }
    
    var body: some View {
        VStack {
            TodoListView(
                todos: todos.wrappedValue,
                onSelect: { todo in
                    PersistenceProvider.default.toggle(todo)
                },
                onDelete: { todos in
                    PersistenceProvider.default.delete(todos)
                }
            )
            TextInputView(
                title: "Add a new item",
                actionTitle: "Add",
                onCreate: { todoText in
                    PersistenceProvider.default.createTodo(with: todoText, in: list)
                }
            )
        }
        .navigationTitle(list.title ?? "")
    }
}
```

The `FetchRequest` class has a `wrappedValue` property with the query result.

**Note**: `TodoListView` is just a helper component, it doesn't do anything related to Core Data itself. Here is the code:

```swift
import SwiftUI

struct TodoListView: View {
    let todos: FetchedResults<Todo>
    let onSelect: (Todo) -> Void
    let onDelete: ([Todo]) -> Void
    
    var body: some View {
        List {
            ForEach(todos) { todo in
                VStack {
                    Text((todo.completed ? "[COMPLETED] " : "") + (todo.title ?? ""))
                        .foregroundColor(todo.completed ? Color.gray : Color.black)
                    }
                    .padding()
                    .onTapGesture {
                        onSelect(todo)
                    }
            }
            .onDelete { indexSet in onDelete(todos.get(indexSet)) }
        }
    }
}
```

# Demo

So, that's it! Download the code (or copy it from this article), and you can compile it and see the actual app running.

# Testing

If you'd like to unit test your Core Data implementation, you'll need to do some changes in our `PersistenceProvider`. 

First, we want to test each test independently, and start from a clean state for each test case, so we'll create an enum for each case: `inMemory` and `persisted`. The `PersistenceProvider` will be initialized with a case of that enum, defaulting to `persisted`. We'll also share the same `NSManagedObjectModel` for all the tests, since that will be much more efficient, and finally point the database location to `/dev/null` so it won't save anything in disk:

```swift
import Foundation
import CoreData

final class PersistenceProvider {
    enum StoreType {
        case inMemory, persisted
    }
    
    static var managedObjectModel: NSManagedObjectModel = {
        let bundle = Bundle(for: PersistenceProvider.self)
        guard let url = bundle.url(forResource: "TodoListsSwiftUI", withExtension: "momd") else {
            fatalError("Failed to locate momd file for TodoListsSwiftUI")
        }
        guard let model = NSManagedObjectModel(contentsOf: url) else {
            fatalError("Failed to load momd file for TodoListsSwiftUI")
        }
        return model
    }()
    
    let persistentContainer: NSPersistentContainer
    var context: NSManagedObjectContext { persistentContainer.viewContext }
    
    static let `default`: PersistenceProvider = PersistenceProvider()
    init(storeType: StoreType = .persisted) {
        persistentContainer = NSPersistentContainer(name: "TodoListsSwiftUI", managedObjectModel: Self.managedObjectModel)
        
        if storeType == .inMemory {
            persistentContainer.persistentStoreDescriptions.first!.url = URL(fileURLWithPath: "/dev/null")
        }
        
        persistentContainer.loadPersistentStores { (_, error) in
            if let error = error {
                fatalError("Failed loading persistent stores with error: \(error.localizedDescription)")
            }
        }
    }
}
```

After having done this, we can start testing our implementation in our unit tests target:

```swift
import XCTest
@testable import TodoListsSwiftUI

class TodoListsSwiftUITests: XCTestCase {
    var provider: PersistenceProvider!

    override func setUpWithError() throws {
        provider = PersistenceProvider(storeType: .inMemory)
    }

    func test_saveTodoList() throws {
        provider.createList(with: "Lista 1")
        let lists = try provider.context.fetch(provider.allListsRequest)
        XCTAssertEqual(lists.count, 1)
        XCTAssertEqual(lists[0].title, "Lista 1")
    }
    
    func test_deleteTodoList() throws {
        let list1 = provider.createList(with: "Lista 1")
        provider.createList(with: "Lista 2")
        let list3 = provider.createList(with: "Lista 3")
        provider.createList(with: "Lista 4")
        provider.delete([list1, list3])
        let lists = try provider.context.fetch(provider.allListsRequest)
        XCTAssertEqual(lists.count, 2)
    }
    
    func test_saveTodo() throws {
        let list = provider.createList(with: "Lista 1")
        provider.createTodo(with: "Todo 1", in: list)
        provider.createTodo(with: "Todo 2", in: list)
        
        let todos = try provider.context.fetch(provider.todosRequest(for: list))
        XCTAssertEqual(todos.count, 2)
        XCTAssertEqual(
            todos.map { $0.title },
            ["Todo 2", "Todo 1"]
        )
    }
    
    func test_deleteTodo() throws {
        let list = provider.createList(with: "Lista 1")
        provider.createTodo(with: "Todo 1", in: list)
        let todo2 = provider.createTodo(with: "Todo 2", in: list)
        provider.createTodo(with: "Todo 3", in: list)
        
        provider.delete([todo2])
        
        let todos = try provider.context.fetch(provider.todosRequest(for: list))
        XCTAssertEqual(todos.count, 2)
        XCTAssertEqual(
            todos.map { $0.title },
            ["Todo 3", "Todo 1"]
        )
    }
}
```

# To sum up

That's finally it. I covered all the basic steps you need to keep in mind while starting a new project using Core Data in SwiftUI. Of course there are a thousand of things you'll learn after it. Migrations, for example.

I highly recommend reading Donny Wals' Practical Core Data book. It's worth reading it if you'll use Core Data in a project.