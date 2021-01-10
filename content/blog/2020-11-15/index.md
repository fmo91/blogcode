---
title: Starting Quick / BDD in iOS
date: "2020-11-15T22:00:00.284Z"
description: "Starting Quick / BDD in iOS"
---

Using Quick and Nimble is a great support for your BDD code, but keep in mind they are just tools, and all of this can be written in XCTest. Use tools as tools, really. And pay attention to the attitude BDD encourages you to take.

# Introduction

Apart from the intended pun in the title, BDD in iOS is a way to work faster. Or maybe not much faster, but keeping a constant pace in iOS development, a way of achieving painless development. In this article, I'll talk to you about Behavior Driven Development, a quick theoric introduction to it, and a hands-on demonstration using Quick and Nimble in a few typical scenarios. Let's get started.

# Agenda

- What is BDD?
- How is BDD related to TDD?
- The problem with XCTest syntax
- Quick (and Nimble) to the rescue!
- Extending Nimble
- Conclusion

# What is BDD?

According to [Cucumber docs](https://cucumber.io/docs/bdd/), BDD is "a way for software teams to work that closes the gap between business people and technical people [...] We do this by focusing collaborative work around concrete, real-world examples that illustrate how we want the system to behave."

So, in more concrete way:

- We start from the acceptance criteria we define in our user stories.
- We define real world examples in the acceptance criteria using the template "Given [initial state] when [event] then [final state]".
- We tipically use a DSL to enable translate the acceptance criteria to machine executable code.

# How is BDD related to TDD?

BDD and TDD are similar in the sense that both of them start by defining the expected result and how to test that expected result is achieved even before writing any code. The difference is in **how they define the expected result**.

In TDD we write tests like this:

```swift
func test_userHasBeenSaved() {
	// Arrange
	let userRepository = UserRepository(mode: .inMemory)
	userRepository.deleteAll()
	userRepository.populateTestUsers()
	let currentUsersCount = usersRepository.currentCount

	// Act
	let user = User()
	user.email = "user.test@sampleemail.com"
	userRepository.save(user)

	// Assert
	XCTAssertEqual(
		usersRepository.currentCount, 
		currentUsersCount + 1,
		"The users count should have increased in one."
	)
}
```

That is a typical test in TDD. We follow the structure `Arrange - Act - Assert`, by creating our initial setuo (Arrange), then execute an action (Act), and finally checking if the final state is the one that we are expecting to be (Assert).

In BDD, the process is similar, but **we are focusing on the application behavior instead of the objects state**.

Let's check the sample example using a BDD approach:

```swift
func test_userHasBeenSaved() {
	// Arrange
	let userRepository = UserRepository(mode: .inMemory)
	userRepository.deleteAll()
	userRepository.populateTestUsers()

	// Act
	let user = User()
	user.email = "user.test@sampleemail.com"
	userRepository.save(user)

	// Assert
	XCTAssertNotNil(
		usersRepository.findOne(email: "user.test@sampleemail.com"),
		"The user we have saved should be recoverable."
	)
}
```

**Disclaimer**: This example has been extracted almost exactly from the [Quick documentation](https://github.com/Quick/Quick/blob/master/Documentation/en-us/BehavioralTesting.md), which I highly recommend.

# The problem with XCTest syntax

The previous example works, and can be used as it is. Even more, most people use it and it works for them. So, what is wrong with it? Shouldn't we just use it and stop bothering?

Well, if you are like me, and you suspect that there should be a better way of doing things, let me tell you a couple of things that could be improved.

1. As BDD should be based on the acceptance criteria, and should be a methodology of interacting with business people, the XCTest syntax is more machine oriented than people oriented.
2. It's hard to describe different interrelated scenarios in XCTest. Let's suppose you want to make the same test, but under different conditions. Doing that in XCTest is not the most pleasing experience, or at least it doesn't feel natural.
3. The tests start to make sense after you write them. This can feel like obvious, but it comes with a consequence: you need to actually write code in order to think the tests!

Let me show you the solution 🙌

# Quick (and Nimble) to the rescue!

At a glance, this is how the previous test could be written in Quick/Nimble:

```swift
describe("Given the User repository") {
	var userRepository: UserRepository!

	beforeEach {
		userRepository = UserRepository(mode: .inMemory)
	}

	context("When a user is saved") {
		beforeEach {
			userRepository.deleteAll()
			userRepository.populateTestUsers()

			let user = User()
			user.email = "user.test@sampleemail.com"
			userRepository.save(user)
		}

		it("Should be recoverable") {
			let recoveredUser = usersRepository.findOne(
				email: "user.test@sampleemail.com"
			)
			
			expect(recoveredUser).toNot(beNil())	
		}
	}
}
```

So, what is this:

1. **Quick** is the library that provides the structure for these tests. It has three main functions:
    1. `describe` is used to describe a module, or a function inside the module. It's the topmost function you should add in your test. It's the **Given** in the acceptance criteria.
    2. `context` lets you describe a particular scenario for your module. It's the **When** in the acceptance criteria.
    3. `it` is finally used to describe a test case for the scenario. It's the **Then** in the acceptance criteria.
2. **Quick** also provides a **beforeEach** function and a **afterEach** function. They can be called at any level in the test, in order to be run before or after each of the sub-specs inside the scope.
3. **Nimble** is the matcher that comes with **Quick**.
    1. Inside the `expect` function, you enter the element to match. If you want to know if a `sum` function works, when you could write something like `expect(sum(2 + 2))`.
    2. After you have the `expect` function configured, you need to match it against something. You do it by writing `.to(...)` or `.toNot(...)`.
    3. Finally, you set an expectation in the right side, inside the `to` / `toNot`.
    4. An example for the `sum` function would be `expect(sum(2 + 2)).to(equal(4))`.

Please, again, note that you can write all the **Quick** code without even writing any **Nimble** code.

This code, for instance, is perfectly valid:

```swift
describe("Given the User repository") {
	context("When a user is saved") {
		it("Should be recoverable") {}
	}
}
```

And it makes a lot of sense when you are talking to QA or business people and defining your feature.

# Extending Nimble

**Nimble** is a great tool. And what makes it even more amazing is the fact that you can extend it with custom matchers. Let's explore the most basic way of writing custom matchers. Let's suppose we want a matcher to check if a number is odd or even.

```swift
func beOdd() -> Predicate<Int> {
    return Predicate<Int>.simple("be odd") { actual in
        let value = try actual.evaluate()!
        return PredicateStatus(bool: value % 2 == 1)
    }
}

func beEven() -> Predicate<Int> {
    return Predicate<Int>.simple("be even") { actual in
        let value = try actual.evaluate()!
        return PredicateStatus(bool: value % 2 == 0)
    }
}
```

Pretty simple, huh? 

Given an expression like `expect(EXPRESSION).to(PREDICATE)`, the generic parameter in the `Predicate<T>` is the expression type.

In the example above, we are matching `Int` expressions against a simple test. The test in the example is simple, we just checking if the `EXPRESSION` is divisible by 2, but it can be much more complex even with this `simple` function.

Finally, the `"be odd"` / `"be even"` are used to describe the error in case the expression doesn't match:

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6a3bd3cd-cecf-4882-9c69-5afc053f7b88/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6a3bd3cd-cecf-4882-9c69-5afc053f7b88/Untitled.png)

Use custom Nimble matchers whenever it improves the readability of your testing code. It needs to feel natural at any step.

# Conclusion

BDD is about tech/business interaction for writing code and defining behavior in a natural and readable way.