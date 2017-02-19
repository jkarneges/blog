---
layout: post
title:  Why I don't document or unit test
date:   2009-01-17 09:19:00
---
I apologize, the headline was mostly an attention grabber. I do write documentation and create unit tests, and in fact I think they are great things. All software projects should be completely documented and have thorough automated testing. However, more often than not, projects fall short in providing these things. Maybe it's because there is no glory in documenting and testing? Tests are especially a bore, as they just prove what should have already been true.

Or maybe it's because documentation and tests provide only marginal value compared to what else you could be doing? For a moment, I wanted to say that volunteer open source projects are more likely to be resource-strapped, but even software companies can be guilty of not wanting to assign resources to these things.

With this in mind, I want to say that deferring documentation and testing is a natural result of trying to get the most bang for your development buck. I won't disagree that it's easy to be lazy about these things, but, personally, even when I attempt to discipline myself, I'm quite cautious about investing time in documentation or tests that I will only need to repair in the future. Consider how often many of us refuse to make fixes to our implementations if we are planning to "rewrite it all" later. It is this very same reason that I resist documentation and testing, and I'm not just making excuses. :)

<!--more-->

## I don't document, because I hate my design

So here I am writing a paragraph of doxygen text about a method call, of course with an example usage and with cross-referencing, etc. Hey, I like good, high quality documentation like the best of them. Then, later on during development, I realize that this method is all wrong and needs to go. Heck, this whole class was written with a misunderstanding about the data it operates on. Hrmph.

It happens in software that you have to rewrite stuff. That's just the way it goes. It's hard to get a design right, even when you're sure it's right. However, if you *know* your design is wrong, then you really have no business documenting. Even if you have no opinion as to whether the design is good or bad, because you wrote it one day and it's done the job fine, you're probably still better off not writing documentation until you do a design audit. Oh, and you really don't want to change your design but then not update your documentation. As the saying goes, wrong documentation is worse than no documentation.

In general, if other users and programmers are figuring out how your software works and are not doing idiotic things all the time, then skip the documenting for now and put your energies elsewhere.

## I don't test, because I hate my implementation

And I almost always hate my implementation. Design comes first, then implementation, and with so much design still left to get right it's easy to see how implementation takes a back seat. Often, I write implementations well enough to satisfy a given design, and then quickly move on to the next design issue. What a day it will be when I only have to worry about implementations.

You may say: but a test need not depend on the implementation. I agree, to a point. There are basic properties you can test, for example you could verify that a return value received is the correct one based on the arguments you provided, or you could verify that some public value is within an acceptable range, etc. However, to test all code paths for full coverage, you need to make tests that are intimately related to the implementations. Anytime the object changes private state, this state needs to be confirmed by a test. It is not enough to check merely the public state.

In general, we all write working code. If Psi did not consist of a majority of working code, then nobody would use it. Even without having any unit tests for many years, the software more-or-less still did the right thing. There is not a strong reason to bend over backwards doing unit testing when the software naturally has such a high correctness rate. Spend your energies on tasks with greater impact (like new features or refactoring, which will end up having a high probability of correctness) and return to unit testing once you're happy with your implementation.

## Exceptions

Yes, there are times when you should document an incomplete design, and that's when you're depending on other people understanding your work-in-progress. The documentation needn't necessarily have a glossy finish. Sometimes even the smallest notes can go a long way in directing the reader. I'm not just talking about programming documentation here. Even a bare bones readme file describing how to compile your darn project can be immensely helpful and takes little effort.

I don't know if it makes sense to unit test an incomplete implementation. At least, doing mediocre unit testing seems like a waste of time compared to mediocre documentation which can be quite valuable.

## Conclusion

Deferring documentation or unit tests is not always about laziness. It's about efficiency. Note my usage of the word "defer". Documentation and tests should be done eventually. I just feel that it is acceptable to prioritize tasks of higher impact first, and to only put serious effort into documentation and tests when the probability of having to rewrite them later is low.

Happy coding!
