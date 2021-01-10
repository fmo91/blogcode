---
title: On abstractions and architecture
date: "2020-12-07T22:00:00.121Z"
---

Few things draw more my attention to programming than the ability to abstract complex domains and concepts behind concise, beautiful representations. Abstraction is at the very heart of software engineering. It is the taming of complexity in favor of manageable structures.

When you start learning about programming, you first learn that a program is "a sequence of instructions written using a Computer Programming Language to perform a specified task by the computer", you accept this as a fact and start learning the basics of programming. Loops, selection, variable declarations, and then you become to know the first abstraction you have to deal with: **the function**.

Functions are the most fundamental and the most important abstraction in computer programming. Functions, like any other abstraction, let you hide complexity behind a reusable construct.

But why is abstraction so important for us? Imagine that, if we weren't abstracting things, we would need to deal with assembly, or even worse concepts daily. Abstraction lies in the center of the human mind and have been used much before computer programming was a thing. 

However, with the rise of software engineering, abstraction became more and more important, because the outcome of software engineering is nothing else than the representation of a moving, live mental model. We get a problem, and we start thinking on how to describe the solution in a way that a computer can understand it. The ability to express ideas in an elegant way is the job of a software engineer.

Apart from abstractions, another thing caught me in love in the first years of career: how can many engineers work together in a single project, with so many classes, functions, and data structures interacting with each other, and how that can not become a mess. Well, apart from the fact that it is actually often a mess, the architecture plays a central role in that.

**I would define architecture as abstractions in advance**. Architecture defines how you build components, and how you let components interact with each other. And it's more like an agreement with the whole team of developers working in a project.

**Software architecture is all about stardardization**. Defining some standard components, and relationships among them, which is called **design patterns**. We can say that architecture is a coordinated, agreed approach to design patterns. It is the next step in design patterns and abstractions. It defines how abstractions are going to happen.

The senior software engineer learns (almost never without a traumatic experience), that **software engineering is all about making wise tradeoffs among different solutions**. Is MVVM the best architecture for a mobile app? Is it MVP? Or maybe VIPER or a Flux-based one? The junior or mid-level developer would choose one. The experience engineer would always say: it depends.

On what does it depend? On many things, but fundamentally:

- **The size of your team**: I certainly wouldn't recommend a microservices architecture with 10 microservices for a solo dev, neither would I recommend VIPER for building an iOS app if you are only yourself, or you and another developer.
- **The nature of your problem**: A simple app shouldn't be designed the same way as a complex one.
- **Time to market**: This affects the architecture decision as well. If you are writing a MVP or participating in a hackathon, it doesn't make sense for you to start working on a complex architecture to it.

Having said all of this, I would sum up by saying that software engineering is the art of balancing trade-offs, make wise choices and evaluate your decisions regularly.