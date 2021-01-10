---
title: Testing the Untested
date: "2020-08-19T22:00:00.169Z"
description: Testing a codebase that was not designed for being tested
---

# Introduction

The last couple of weeks I had to work on a task that demanded most of my time. The assignment was about increasing the code coverage in a module by adding tests to one layer at a time.

Reducing technical debt is one of the things I love doing the most in the context of software development. Increasing the codebase quality is more and more important as the time goes on and the codebase gets bigger. 

However, it didn't come without the pain, and I learnt some important things along the process. During this article, I'll tell you some of the things I've learnt.

Let's start by defining some basic concepts so we can then go on our specific case.

# The basic questions

## What

There are different types of tests: Scenario-based testing, Unit testing, E2E testing, etc. Unit testing, in particular, is the process of testing one module at a time. 

That's REALLY important to notice. We are testing ONE module at a time, in isolation, which is called SUT (system under test). When I'm testing a presenter, I don't care if the database is working properly or not, or even if it wasn't developed yet. The only thing I care is that my module is working properly and doing whatever it is supposed to do in a good manner.

## Why

Unit testing the codebase has a number of outcomes that are important:

- **Ensure the code works properly**: This one is, of course, the main reason to test our code.
- **Ensure the code is properly structured:** This is less obvious than the previous one. But it's important to note that a code that is testable, is almost always a good structured code.
- **Document the right behavior of the code**: Tests are a way of documenting what the code is supposed to do.

## How

As mentioned earlier, a unit test implies that the code is tested in isolation, that means:

- A code that is failing in one class shouldn't break a test for another class.
- A class that depends in another class, should be tested isolated from those dependencies.

In addition to that, there are a couple of other properties from Unit tests that we should pay attention to:

- Unit test cases should be fast to run.
- Unit test cases should be reproducible.

All of these properties will ensure us that our test will give us rich information about what works, what doesn't, and where the error actually is.

# Dependencies

Dependencies are modules (functions/classes) another module depends upon.

For example, in this (probably incorrect) code fragment:

```swift
class ViewController: UIViewController {
	@IBOutlet private weak var tableView: UITableView!
	let api = API()

	var contacts: [Contact] = []

	override func viewDidLoad() {
		super.viewDidLoad()

		self.configureTableView()

		api.fetchContacts { [weak self] result in
			switch result {
			case let .success(contacts):
				self?.contacts = contacts
				self?.tableView.reloadData()
			case let .failure:
				break
			}
		}
	}
}
```

The `ViewController` depends upon a `API` that brings an array of `Contact` to it. That is a dependency. Imagine there is a bug in the `API` `fetchContacts` method, and we are testing `ViewController`, we would see there is an error in the `ViewController` while the error is actually in the `API`! That's why we need to isolate the classes from their dependencies. An architecture that lets us test each of its components in isolation, is a testable architecture.

# Techniques

As we've seen, we need to isolate our modules from one another, so we can test each module separated from the others. In order to do so, there are two techniques we need to master:

1. Test doubles
2. Dependency injection

## Test doubles

Test doubles are classes or functions that have the same signature or interface than another class or function but have their functionality especially adapted to be used for testing purposes. We'll focus in classes for the purpose of this article.

All of them are based on the idea of interfaces/protocols. Basically you let the class to be dependent on an interface instead of a concrete implementation. In our previous example, you could have an interface called `APIType` with the following shape:

```swift
protocol APIType {
	func fetchContacts(completion: @escaping (Result<[Contact], Error>) -> Void)
}
```

That way we could have

```swift
class API: APIType {
	func fetchContacts(completion: @escaping (Result<[Contact], Error>) -> Void) {
		//...
	}
}
```

And the view controller could have a dependency on `APIType` and not on `API`.

```swift
class ViewController: UIViewController {
	let api: APIType
	// ...
}
```

Well, so going back to test doubles, there are two of them that I've used in my unit tests for the project I've talked about in the beginning, mocks and stubs.

### Mocks

Mocks are classes that record the calls on each method or property they have. They are used to do behavior-based testing. 

Imagine the case of an analytics layer. We could use a mock, in order to record the calls our view controller does for certain events.

```swift
protocol AnalyticsType {
	func purchaseDidEnd()
}

final class AnalyticsMock: AnalyticsType {
	struct Calls {
		var purchaseDidEnd = 0
	}

	var calls = Calls()

	func purchaseDidEnd() {
		calls.purchaseDidEnd += 1
	}
}
```

By using a mock, we could test the *behavior* of the SUT. We perform some action in the view controller, for instance, and then we verify that the mock has registered the event properly.

### Stubs

Stubs, on the other hand, are object with a predefined return value on each of their methods. They are used to do state-based testing.

Our previous example for the `APIType` could suit perfectly for a stub.

```swift
final class APIStub: APIType {
	var fetchContactsResponse: Result<[Contact], Error>?

	func fetchContacts(completion: @escaping (Result<[Contact], Error>) -> Void) {
		completion(
			fetchContactResponse ?? .success([])
		)
	}
}
```

By using a stub, we could test the *resulting state* of the SUT. We perform some action in the view controller, and then we verify that the SUT has ended in the expected state. In this case, with the correct value in the `contacts` property.

## Dependency injection

The key to isolate the SUT from the rest of the system is to *inject* mocks and stubs instead of the concrete implementation for each of the SUT's dependencies.

Dependency injection consists in give an object its dependencies instead of letting it create them by itself. There are different ways of doing dependency injection, but let's stick with one of the simplest ones for this article: constructor-based dependency injection.

In our example, we could give the view controller an instance of an `APIType` instead of letting it create it by itself.

```swift
class ViewController: UIViewController {
	@IBOutlet private weak var tableView: UITableView!
	// 1
	let api: APIType

	var contacts: [Contact] = []

	override func viewDidLoad() {
		super.viewDidLoad()

		self.configureTableView()

		api.fetchContacts { [weak self] result in
			switch result {
			case let .success(contacts):
				self?.contacts = contacts
				self?.tableView.reloadData()
			case let .failure:
				break
			}
		}
	}

	// 2
	init(api: APIType) {
		self.api = api
		super.init(nibName: "ViewController", bundle: nil)
	}

	required init?(coder aDecoder: NSCoder) {
		return nil
	}
}
```

If you pay attention to the comments, in (1) we don't create the `APIType` because in (2) we receive the object in the constructor. As simple as that can be a dependency injection. 

# A simple unit test

```swift
import XCTest
@testable import OurApp

final class ViewControllerTests: XCTestCase {
	func testFetchContacts() {
		// We start by creating the stub
		let api = APIStub()
		let contacts = [
			Contact(name: "Fernando"), 
			Contact(name: "Martin")
		]
		api.fetchContactsResponse = .success(contacts)

		// When we create our view controller, we inject the stub as its api.
		let viewController = ViewController(api: api)

		// Trigger viewDidLoad()
		_ = viewController.view

		// Then we assert that the contacts have been
		// properly stored in the view controller's state.
		// This is state-based unit testing
		XCTAssertEqual(
			viewController.contacts,
			contacts
		)
	}
}
```

# The three types of Unit tests

There are three types of unit tests that I use the most in my day to day job.

## 1. Input-Output

These are the simplest tests, and most frequently seen applied to functions. The process is very simple: you take a function, give it an input and expect a certain output of it.

```swift
// Given this function
func sum(x: Int, y: Int) -> Int {
	return x + y
}

// We can test it this way
final class MathTests: XCTestCase {
	func testSum() {
		XCTAssertEqual(sum(x: 1, y: 2), 3)
	}
}
```

There is an important think to consider before doing this kind of tests: **The function must be pure.** And what does it mean for a function to be pure? Well, it's that the function shouldn't have any side-effect, and for each input that you give to it, you will get the same output. That will let us be sure that regardless of the state of the system, we'll get the same result once and once again.

## 2. State-based

It would be great if every piece of code would be tested by Input-Output unit tests. However, we're not that lucky.

There are certain scenarios where we are forced to use other techniques. State-based testing consists on arrange the initial state of the object to be tested (arrange), then apply some action (act) on it, and finally check (assert) if the resulting state in the system under test is correct. Let's see an example:

```swift
// Given this view controller
final class LoginViewController: UIViewController {
	@IBOutlet var emailTextField: UITextField!
	@IBOutlet var passwordTextField: UITextField!
	@IBOutlet var errorLabel: UILabel!

	@IBAction func loginButtonPressed() {
		guard 
			let email = emailTextField.text, 
			let password = passwordTextField.text,
		else {
			return
		}

		guard password.count >= 6 else {
			errorLabel.text = "Password too short"
		}

		// ...
	}
}

// It can be tested this way
final class LoginViewControllerTests: XCTestCase {
	var viewController: LoginViewController!

	override func setUp() {
		self.viewController = LoginViewController()
	}

	func testLoginButtonPressed() {
		// Arrange
		viewController.emailTextField.text = "ortizfernandomartin@gmail.com"
		viewController.passwordTextField.text = "1234"

		// Act
		viewController.loginButtonPressed()

		// Assert
		XCTAssertEqual(viewController.errorLabel.text, "Password too short")
	}
}
```

It's important to note that we haven't used neither Mocks nor Stubs nor Dependency Injection for this example. The reason is simple: this view controller doesn't have any dependency.

Let's extract the logic to get the error in a separate module and then do a stub for it.

```swift
// Let's start by defining the protocol for our new functionality
protocol LoginErrorProvider {
	func error(forEmail email: String, andPassword password: String) -> String?
}

// Given this view controller
final class LoginViewController: UIViewController {
	@IBOutlet var emailTextField: UITextField!
	@IBOutlet var passwordTextField: UITextField!
	@IBOutlet var errorLabel: UILabel!

	// We have added the dependency on it
	private let errorProvider: LoginErrorProvider

	// And a custom constructor
	init(errorProvider: LoginErrorProvider) {
		self.errorProvider = errorProvider
		super.init(nibName: "LoginViewController", bundle: nil)
	}

	// Other initializers...

	@IBAction func loginButtonPressed() {
		guard 
			let email = emailTextField.text, 
			let password = passwordTextField.text,
		else {
			return
		}

		// And we use the error provider here
		if let error = errorProvider.error(forEmail: email, andPassword: password) {
			errorLabel.text = error
			return
		}

		// ...
	}
}

// We will now need a stub for testing this!
// It will be VERY simple
struct LoginErrorProviderStub: LoginErrorProvider {
	var errorValue: String?

	func error(forEmail email: String, andPassword password: String) -> String? {
		return errorValue
	}
}

// It can be tested this way
final class LoginViewControllerTests: XCTestCase {
	var errorProvider: LoginErrorProviderStub!
	var viewController: LoginViewController!

	override func setUp() {
		self.errorProvider = LoginErrorProviderStub()

		// And inject the provider!
		self.viewController = LoginViewController(
			errorProvider: errorProvider
		)
	}

	func testLoginButtonPressed() {
		// Arrange
		viewController.emailTextField.text = "ortizfernandomartin@gmail.com"
		viewController.passwordTextField.text = "1234"
		errorProvider.errorValue = "Sample error"

		// Act
		viewController.loginButtonPressed()

		// Assert
		XCTAssertEqual(viewController.errorLabel.text, "Sample error")
	}
}
```

It's important to not HOW IMPORTANT is the **Stub** test double in this kind of tests. As a general rule of thumb, Stubs are used for state-based testing and Mocks for behavior-based unit testing.

## 3. Behavior-based

An alternative way to make unit tests, that complements the state-based testing, is the behavior-based unit testing. It consists on making assertions on HOW the SUT used its dependencies. Let's go straight to an example:

```swift
// Let's start by defining the protocol for our new functionality
protocol LoginErrorProvider {
	func error(forEmail email: String, andPassword password: String) -> String?
}

// Given the same view controller
final class LoginViewController: UIViewController {
	@IBOutlet var emailTextField: UITextField!
	@IBOutlet var passwordTextField: UITextField!
	@IBOutlet var errorLabel: UILabel!

	private let errorProvider: LoginErrorProvider

	init(errorProvider: LoginErrorProvider) {
		self.errorProvider = errorProvider
		super.init(nibName: "LoginViewController", bundle: nil)
	}

	// Other initializers...

	@IBAction func loginButtonPressed() {
		guard 
			let email = emailTextField.text, 
			let password = passwordTextField.text,
		else {
			return
		}

		if let error = errorProvider.error(forEmail: email, andPassword: password) {
			errorLabel.text = error
			return
		}

		// ...
	}
}

// We will now need a mock for testing this now
// It will be VERY simple
struct LoginErrorProviderMock: LoginErrorProvider {
	var errorCalls: [(String, String)] = []

	func error(forEmail email: String, andPassword password: String) -> String? {
		errorCalls.append((email, password))
		return ""
	}
}

// It can be tested this way
final class LoginViewControllerTests: XCTestCase {
	var errorProvider: LoginErrorProviderMock!
	var viewController: LoginViewController!

	override func setUp() {
		self.errorProvider = LoginErrorProviderMock()

		// And inject the provider!
		self.viewController = LoginViewController(
			errorProvider: errorProvider
		)
	}

	func testLoginButtonPressed() {
		// Arrange
		viewController.emailTextField.text = "ortizfernandomartin@gmail.com"
		viewController.passwordTextField.text = "1234"

		// Act
		viewController.loginButtonPressed()

		// The assertion will be different this time
		// We'll check if the error provider has been called just once
		XCTAssertEqual(errorProvider.errorCalls.count, 1)
	
		// And we'll check if the mock has been called in the correct way
		XCTAssertEqual(
			errorProvider.errorCalls.first!.0, 
			"ortizfernandomartin@gmail.com"
		)
		XCTAssertEqual(
			errorProvider.errorCalls.first!.1, 
			"1234"
		)
	}
}
```

# Testable architecture

Let's talk for a moment about what makes a testable architecture. In the previous article, I explained about dependency injection and test doubles. A testable architecture lets us inject test doubles to test each part of it in a relatively easy way.

# VIPER

VIPER stands for View, Interactor, Presenter, Entity and Router, and it's an example of a testable architecture based on the idea of [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) by Robert Martin.

To be brief:

- The view involves the `UIKit` related classes, such as `UIViewController` and `UIView` descendants.
- The interactor consists in the business use cases for the app. It's completely platform independent and doesn't know anything about the way the data will be presented.
- The presenter talks to the interactor, gets the data and prepares some presentation-ready structures that can be requested by the view.
- The entities are just the models that are used by the interactor.
- The router is the class that is responsible for the navigation. It holds a reference of a `UINavigationController` or a `UIViewController` and is hold by the presenter.

In VIPER, each layer can be tested in isolation. If you take the presenter, you can give it a mock of the router and the interactor, and test how the it calls them to verify its correct behavior.

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d9ccf3d8-6e47-489a-b274-14c2ff8e42c3/VIPERArchitecture.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d9ccf3d8-6e47-489a-b274-14c2ff8e42c3/VIPERArchitecture.png)

Each of these modules are self contained pieces of the "architecture cake", going from the core to the boundaries.

## VIPER: The revelation

You can divide the layers in the VIPER architecture in three important categories:

- **Core**: The Entities and the Data Sources are nearer to the core of the business logic.
- **Policies**: The Interactor, Router and to a certain extent, also the presenter, are policies inside the app. They decide what to call and how, and make some decisions in the logic flow.
- **UI**: The View and the Presenter to some extent conform the layer that is nearest to the user.

I've noticed that:

- The core is usually more testable in a input-output way
- The Policies are usually more testable in a Behavior-based way
- The UI is usually more testable in a State-based way

    ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d8e888bb-8fd6-4eb5-bf0b-a992a7f72880/VIPERTesting.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d8e888bb-8fd6-4eb5-bf0b-a992a7f72880/VIPERTesting.png)

The reason for that comes from the own nature of the layers. The View is stateful by nature, so State-based testing is perfect for it. The intermadiate layers know what to call and how to do it, so Behavior-based testing fits perfectly. Finally, the Core layer know nothing about the other layers, and almost never store state on them, so they fits perfect in the Input-Output way.

# Summary

Unit testing consists on testing a unit of code (a class, a function) isolated from the rest of the system. In order to do so, there are two powerful techniques we can use: test doubles (mocks or stubs), and dependency injection.