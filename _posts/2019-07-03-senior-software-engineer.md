---
layout: post
title:  "The Senior Software Engineer"
date:   2019-07-03 18:00:00
categories: [Book, Software Engineering, Technical Leader, Code Reviews, Technical Interview, Greenfield projects, Technical decisions]
comments: true
---

<img src="/img/senior-software-engineer.jpg" alt="Cover Page" width="40%">

## The Senior Software Engineer

I've just finished the book 'The Senior Software Engineer' of [David Bryant](https://twitter.com/davetron5000), and I think is a really good book for any software developer who wants to know what really seniority means. Some developers think that to be a senior software engineer is related to the years that he or she is has been working in software engineering and I have always thought that this concept is a mistake. David Bryant thinks the same than me, and besides, he proposes several capabilities that a real senior software engineer must have.

In the first chapter, the author explains to the reader one of the most important topics in the book: How to focus on delivering results. This topic is the guiding theme of the book and one of the most important capabilities that a software engineer should have, in my humble opinion. Focus on delivering results is a very interesting chapter in which, the author shows to the reader with hands-on experience what are the results, and how to manage it.

It's not my intention, to write a recap of each chapter. However, I would like to highlight some chapters or sections that I've been very impressed by the content and the main concepts. Therefore,  I will outline these sections in the following list.

* In the chapter 'Add new features with ease', the author introduces a diagram whereby the flow to develop a new feature is explained step by step. I have to admit that this flow diagram is one of the concepts that I most like in the book.
This flow diagram, is a mix between TDD and developing good acceptance test and, as a summarize, it's more or less as follow:

1. Understand the context
2. Implement acceptance test. (Failing acceptance test)
3. Implement the feature or the use case with TDD.
4. Acceptance test passing
5. Code review

* In the middle of the book, the reader is taught about how to make a techincal decision. In this chapter, I've taken note of two main ideas since this is a weak point of mine.

1. Facts, priorities and conclusions. Making a decision about what solution to use can be difficult; programmers are an opinionated bunch. Therefore, when you are making a decision, at a high level, you must identify facts (as distinct from opinions), identify your priorities (as well as the priorities or other) and then combine the two in order to make your decision or put forth your argument.

2. Falacies. Making decisions based on fallacies can create problems later on and fallacies have a way of cropping up in many debates. The author exposes some of the main fallacies existing in the software engineering like hasty generalization, correlation does not imply causation, false equivalence or appeal to authority.

* In the following chapter, 'Bootstrap Greenfield Systems', I would like to highlight a new term, for me at least, MDS or minimum deployable system. This section explains that once you have made the required technical decisions and the ecosystem has been established, you must focus on deploying your system as fast as possible to production.  When the system has been deployed, developers can add new features more easily and that features can be deployed and delivered more quickly than if you had started developing features at the beginning and leaving the deployments for later.  
In my humble opinion, this idea should be taken into account whenever possible and particularly in the projects that continuous deployment cannot be performed.

* One of the aspects that generates the most controversy among the technical leaders is how to make effective technical interviews. In the ninth chapter, David Bryant explains how the ideal technical interview should be with a step by step process:

1. Informal technical discussion to meet and get to know the candidate
2. Have the candidate do a "homework assignment"
3. Conduct a technical phone screen.
4. Pair program with the candidate person.  
___  
Although this process may look very difficult and very hard, after some bad experiences, I think this procedure is a really good approach for making a technical interview.  The homework is a good filter and an assurance about the candidate who will be interviewed in person. Moreover, a pair programming session is one of the best techniques to exchange views on any technical debate.

* In one of the final chapters, the author teaches the reader how to be more productive. Being honest, I have to put emphasis on this capability.  
In one of the final chapters, the author teaches the reader how to be more productive. Being honest, I have to put emphasis on this capability.  For this assignment, I would like to highlight the improved graph of the chapter 'Add features with ease' to the which, the author has added when to do long and short breaks in order to manage better the email or answering some chat without losing the focus.
In this flow diagram, the author has added the breaks concisely to fit with the stages in which a developer need a short break, like when in the TDD cycle, a unit test is passing or a longer break, like when the code is refactored. Besides,  He stipulates the stages in which a break is not allowed like when a developer is writing a unit test or when a developer is cleaning up the code.

* At the end of the book, the author gives several tips about how to lead a team. In particular, I would like to stress in the section 'General tips on Reviewing code'.  
As a senior software engineer, you have to review code every day and being a great code reviewer is a great capability that your team will appreciate. The following tips will help you to do a great code reviews:

1. Read the entire changeset before commenting on it.
2. Speak about the code, not the coder
3. If you don't understand something, ask
4. Approach style-related issues tactfully.
5. Don't be afraid to have a real-time conversation
6. Convey this information to others.
7. Interacting with QA.
8. Interacting with a Security Team.

To sum up, the senior software engineer is a great book in which capabilities that a senior software engineer must have are been explained, and besides, several techniques for improving your daily work. I've learned a lot of useful concepts and ideas that I'm already applying in my current job that has helped me to improve my performance. I would like to recommend this book to every software engineer, not only junior developers but also to more experienced developers since it offers some valuable guidelines for everyone who wants to improve its capabilities and to be a better software engineer.






