---
title: On testing private methods
date: "2020-12-13T22:00:00.121Z"
---

# Do you have to test private methods?

There is a lot said about this. Just check this [thread](https://stackoverflow.com/questions/105007/should-i-test-private-methods-or-only-public-ones).

The short answer is **no**.

The slightly longer answer is as follows.

# The goal of Unit testing

When you write unit tests, you are testing the logic behind some module in your project. There are many ways of writing unit tests, with and without mocking or stubbing, or maybe testing pure functions. 

In any case, when you write a unit test, you first create and configure the object you are testing (the SUT, system under test). This step is called **arrange**. After that, you **act** on your SUT. And finally **assert** that some state in your SUT or result returned from it is as expected. However, the act part can be as complex behind the scenes as you can imagine.

# What is a private method?

Private methods are hidden complexity. That is great. I mean, you hide the complexity behind your implementation so the user of your module doesn't need to bother on how did you write the logic in it. The user just call on the public interface of your module, and the magic happens.

![private-0001](https://dev-to-uploads.s3.amazonaws.com/i/ijykqwesw78za4af9pja.png) 

This diagram shows how simple can it be for the client perspective to call a method in the module.

However, hidden complexity is free when using the module, but has its cost when testing it.

![private-0002](https://dev-to-uploads.s3.amazonaws.com/i/7baftausk12jlbn220zh.png)
 
Imagine the method you are testing from the SUT has 10 lines of code. If it calls three 15 LoC methods, you are testing 55 LoC. And you can imagine how complex the test can become... 

# A better approach

Single responsibility principle encourages us to create simple, small, concise functions and modules that do one and only one thing. If your module has many private functions, the SRP is not correctly implemented.

The better approach is to split the private SUT in a set of objects that can be injected in your SUT as needed, and also be mocked.

![private-003](https://dev-to-uploads.s3.amazonaws.com/i/gw1atlwu53loguxt21tt.png) 

This is the first step in making a class more testable. The next step I always recommend taking is to hide every dependency under an interface, so you can mock each dependency in a simple way.

![private-0004](https://dev-to-uploads.s3.amazonaws.com/i/cq991rdf0lzmv7xskc9w.png) 

Once you have this, your module becomes much more testable. 

If you now need to test that method with complex private dependencies, you can just inject mocked objects that implement the right interfaces so you can test just the piece of logic you are interested in.

Of course, the new objects you have created are now testable as well, because each of them is a simple, concise unit of logic.

# To sum up

When you write a module, keep testability in mind, keep them small, and take care of their private interfaces using this simple technique.