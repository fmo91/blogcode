---
title: "Noise and sound"
date: "2021-01-03T22:00:00.121Z"
---

What is the purpose of language constructs and software architectures in software development? The answer, in my opinion, is **to bring more sound and less noise.**

From my viewpoint, developers can make sound or noise. The **sound** is the code we write that actually makes sense at a glance. You read the code, and it's like somebody is talking to you, telling you a story, or a fact. 

In other cases, the code is just boilerplate that supports the actual functionality that is in other places. I consider this to be **noise**.

The React Context is a great example on how to make more sound and reduce noise in a codebase. 

To be brief for those who don't know React, React components are software components that are responsible of transforming a piece of state into a view description. You can use React components to compose even bigger components by placing them into trees of components. So that, one component may have other components as children, which in turn may have other components as children. 

![NoiseSound-01](https://dev-to-uploads.s3.amazonaws.com/i/im3d00r8pvvtadpgtweb.png) 

The problem we had back then was that some components in the top levels needed to send information and be notified of events that happened in the lowest levels. To support that, the components in the intermediate levels needed to get the data from the top level and bring it to the lower level, even when that data wasn't needed for the intermediate components to work. That was clearly **noise**.

![NoiseSound-02](https://dev-to-uploads.s3.amazonaws.com/i/qn3bnicxqe9yzpeltqyy.png) 

A great solution came when React started to officially support (out of beta) **React Context.** React Context is basically a data container that is defined for a specific subtree. So, using React Context, the top level component can now insert data into a context, and let the lower level component get the data from the context. This leaves out of noise the components in the middle of the hierarchy.

![NoiseSound-03](https://dev-to-uploads.s3.amazonaws.com/i/a4s8z9oxh7c7m0l0n3a9.png) 

Of course, the story is not always as happy as this one. Some architectural patterns, such as VIPER in the iOS landscape has introduced noise to codebases that could have been more sound otherwise. This is not always the case and I've seen great implementations of VIPER architectures. However, in general, it bring more noise than the sound it could provide.

Most coding tasks, unlike what many might think at the beginning of their careers is more about communication and language than about maths. If you want to be a great software engineer, you need to become great at communicating, great at telling stories, and great at recognizing when you are making noise and when you are making good, relevant, and enjoyable sound.