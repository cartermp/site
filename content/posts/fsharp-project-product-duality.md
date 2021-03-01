---
title: The F# project-product duality
date: 2020-12-30
author:
  - Phillip Carter
tags:
  - fsharp
  - open source

# Set how many table of contents levels to be showed on page.
geekblogToC: 3

# Set true to hide page or section from side menu (file-tree menu only).
geekblogHidden: false

# Add an anchor link to headlines.
geekblogAnchor: true
---

The F# language is kind of wierd (in a good way). Born out of Microsoft Research and initially sold to the world by Microsoft as a part of Visual Studio, it now has a vibrant open source community and ecosystem around it. This puts it in a funny place. A great way to illustrate that is to visit the following links:

* [F# homepage on the Microsoft .NET site](https://dotnet.microsoft.com/languages/fsharp)
* [Homepage for F# and the F# Software Foundation](https://fsharp.org/)

The first link a page on the Microsoft marketing site for .NET. The other is 100% independent of Microsoft, run by the F# Software Foundation, and has been around longer than the marketing site (ironic, given that Microsoft's involvement with F# came well before the F# Software Foundation).

There are other ways you can see this duality play out:

* F# is delivered as a part of Visual Studio vs. F# can be built from source on most Linux distributions
* F# can target the .NET Framework for Windows apps vs. [F# can target the JavaScript runtime](https://fable.io/)
* F# users who primarily use it for OSS vs. those who primarily use it for work (such as at a financial institution)
* People who use Visual Studio Pro/Enterprise or Rider vs. those who use [Ionide](https://ionide.io/) and Visual Studio Code
* F# users are participate in the F# Software Foundation vs. those who do not
* F# users who submit issues on GitHub vs. those who do not

There are several other examples that I could think of if I was feeling more thoughtful. The two sides of the "vs." aren't necessarily opposed. In fact, I know of several F# developers who have different personas depending on their work. But they are very different from one another.

The rest of this post will be a short meditation on this duality. I don't really have a whole lot of insight to offer here. But it is a unique challenge that I face every day.

F# is an open source project
I mention this first because I think it's actually the most important side of the F# project-product duality. Hell, we even say it in the F# marketing page: it's an open source, cross-platform functional language. There you have it, open source and cross-platform. But what does that even mean?

Firstly, in case you aren't aware, the F# compiler, F# core library, F# language service, and F# Visual Studio tooling are all 100% open source. You get to observe every bit of mind-numbing process related to flowing commits to different branches, every stupid mistake we make when we introduce a bug, and every "whoopsie" whenever we say something to a community member and we're proven wrong. And all the good stuff too, like active language development, new tooling features, and countless contributions from people all around the world.

The F# repository is governed, mostly, as such:

* We have a code of conduct and we act on it if necessary. It'd be nice if we didn't need it. We banned a white supremacist for sending death threats to F# users and locked any issues he filed.
* Issues are labeled and assigned a milestone (most of the time!). Milestones are always either "Backlog" or a release corresponding to a Visual Studio release. Backlog items get pulled into a release milestone if they're completed during the appropriate timeframe.
* We may not comment on an issue, often because it is a straightforward bug and just needs to get tagged correctly. But we do strive to look at everything that is filed at least once, and typically do.
* Code contributions usually require tests or some proof that they work if tests are silly/untenable.
* Performance improvements usually require evidence of the improvement to be approved. Sometimes improvements are just obvious, so we'll take them without evidence.
* Not all parts of the repo can be built everywhere, so we have several Solution files to ensure you can always build the repo in your environment of choice.
* The best ideas usually win out on how to do something, and they do get discussed out in the open.
* Sometimes an obvious improvement will be rejected, but it's usually for complicated reasons related to long-term maintenance and support. Sometimes the maintenance cost of adding something outweighs the benefit of users having it. It sucks, I know. But we cannot let technical debt hamstring us.
* We have the privilege and the curse of incorporating Visual Studio components. It comes with great testing, compelling language tools, and a much larger userbase than many open source projects. But we are tethered to its processes and this often results in improvements taking time - sometimes months - before they are actually released to users. We try to keep a steady flow of improvements constantly moving in the codebase to remidiate the impact this has.
* We have some internal servers that run a "signed build" for VS integration, it's closed-source, it sucks, and it sometimes stalls our infrastructure from moving forward for a month or two. We apologize, but if it makes you feel better, it slows us down too.
* There's more to it, but I think this captures most of how we work in the open. There's some elements of being a product involved (especially as far as Visual Studio integration is concerned), but I like to think that we're really just a very active and fairly well-run open source project.

But F# as an open source project is more than just the very literal aspects of the actual F# codebase being open source. The "F# project" involves many other open source entities, some of which include:

* The [fsprojects](https://github.com/fsprojects) organization, a community-run incubation space that serves as a home for several open source projects (some of which are long past an "incubation" period!) and has a core group of administrators who ensure that active projects are always assigned at least one maintainer.
* The [fslab](https://github.com/fslaborg?type=source) organization, which is basically "fsprojects but for data science and machine learning". Several independent projects suited towards analytical work live here, and like fsprojects, there is a core group of administrators who ensure that active projects are always assigned at least one maintainer.
* The [Fable](https://fable.io/) project and [ecosystem](https://fable.io/community/), which transforms F# into a JavaScript-targeting language and comes with a vibrant set of libraries for building beautiful, modern web applications in F#.
* The [FsAutoComplete](https://github.com/fsharp/FsAutoComplete) project, which powers [Ionide and Visual Studio Code](https://ionide.io/), and F# support in Vim and Emacs.
* The [Ionide](https://ionide.io/) project, which turns VSCode into an IDE and also provides several other awesome F# tools.
* The [SAFE stack project](https://safe-stack.github.io/), which composes several open source projects into a cohesive set that can be used to build full-stack F# apps (F# on the frontend and backend, yes!). Also comes with commercial support through several parties that contribute to SAFE.
* The [WebSharper project](https://websharper.com/), which is actually also a commercial product, where you build full-stack web apps in F#.
* The [bolero project](https://fsbolero.io/), which makes F# a WASM language (backed by a commercial entity).
* The [Jetbrains Rider F# plugin](https://github.com/JetBrains/fsharp-support) is how F# tooling works in Jetbrains Rider, powered by the same open source language service used by several other open source tools.

There are many more, but hopefully my point about F# being an open source project matters. There are so many amazing things you can do with the language, all of which is rooted in the fact that F# is cross-platform, open source, and inviting towards a community of developers to empower them to build awesome shit. **This is what being an open source project is all about**.

## F# is a product

The other side of F# is very different, and also has a more storied history. In fact, much of the early history of F# is well-documented in [The Early History of F#](https://fsharp.org/history/), which I highly recommend reading. I won't discuss aspects of F# the product much before the time when I started working on it at Microsoft. This is partly because it's already well-documented, but also because the priorities of a product change over time, and I just don't know the details of what those priorities were prior to 2015.

For what I wager is the majority of F# developers, F# is a product more than it is an open source project. Far more people use the F# compiler and use F# packages than contribute back. This isn't inherently a bad thing, it's just the reality of how things work. Not everyone can or wants to spend their spare time doing open source work, and most people don't have an employer that lets them spend work time on OSS. I really wish employers paid employees to contribute back to the tools and components they use, but most don't, so that's just how it is right now.

F# the product surfaces in several ways:

* Delivered as a part of the .NET SDK, both Current and LTS trains, across the various mechanisms it can be installed
* Delivered as a part of the [build from source](https://github.com/dotnet/source-build) packaging for .NET
* Delivered as a part of Visual Studio
* Delivered as a part of Visual Studio for Mac
* Delivered as a part of Jetbrains Rider
* Delivered as NuGet packages, FSharp.Core and FSharp.Compiler.Service

If you're using F# in Visual Studio, VS for Mac, or Rider then you're doing so under a license. Sometimes that license means it's 100% free (e.g., VS Community - which can be used by students, OSS developers, and small entities making under a certain threshold of revenue). But it's usually a paid license when you're using it professionally, since you're usually doing it for some company that makes money.

When you're in this setting you likely care deeply about several things:

* Setup/prerequisities - Getting a developer and runtime environment set up should be straightfoward. Complex aspects of this are improved over time.
* Language compatibility - What compiled and ran yesterday must compile and run today.
Tooling compatibility - IDE behaviors don't randomly change. Actions you performed yesterday can be performed today.
* Language reliability - New compiler bugs aren't introduced over time. Existing bugs are fixed over time.
* Tooling reliability - Tooling doesn't break with updates. Existing problems are fixed over time.
Compiler performance - As your codebase grows, the compile times increase linearly with respect to your code size. The linear function isn't too high either. No exponential compile times, thank you! Compiler performance improves over time.
* Runtime performance - The run time of your application doesn't regress over time. Existing problems are resolved over time and the run time gets faster with updates.
* Tooling performance - As your codebase grows, you don't end up "fighting the IDE" or some other tool. Existing tooling performance problems are resolved over time, and the tools get faster too.
* Language features - New features that unlock certain capabilities or let you simplify an approach to a problem are regularly delivered. Existing, high-use language features are improved over time.
* Productivity features - New tooling features that let you see more information, navigate code, refactor code, etc. are regularly delivered. Existing features are enhanced over time.

There's more, but I think that list covers the fundamentals. None of what I mentioned is exclusive to being a product. Every OSS developer cares about these things too. However, all of this can be achieved by a closed-source, proprietary language implementation and tooling. Community is not required. Extending capabilities for others to explore is not required. Having a diverse group of contributors is not required (although it does mean you need more staff on hand). And if feedback I've seen over the years is any indicator, plenty of F# developers really don't care much about the aspects of "F# is an open source project". After all, there's likely something not running 100% smoothly and the fact that F# can be used to build wonderful new web applications doesn't fix the problem they're seeing. Nor does any number of open source contributors adding features or fixing bugs that don't address the problem they're facing.

Dealing with the "product" aspects of F# takes a significant amount of time for me and everyone on the F# team at Microsoft. This isn't exactly surprising, though. We're paid to improve F# the product, so we do exactly that. What does that mean in practice? A few things.

First and foremost, we try to get more information out of paying customers. Are you working at a company using F# through Visual Studio? Are you paying Microsoft a lot of money through VS subscriptions, Azure subscriptions, and so on? If so, **we want to hear from you**. If you're paying a lot of money then your problems are "premium problems" that we prioritize above pretty much anything else. Microsoft has gone so far as to pay for me to fly out to a customer and just collect general feedback about using F# because their Azure bill was big enough for us to address general issues they might be having. In one such occaison, we found an embarassing compiler bug where we'd stack overflow at build time. The person who told me about it didn't think much of it until my eyes got all wide, baffled as to why his build would fail like that. We got the bug fixed in time for the next Visual Studio update, and with it, a new F# compiler delivered with the fix across all channels. **We would not have known about this issue without direct engagement**.

We have a [feedback system through Visual Studio](https://docs.microsoft.com/en-us/visualstudio/ide/how-to-report-a-problem-with-visual-studio?view=vs-2019) that can let us know if the person filing the bug is doing so from a paying account, and these are often treated with higher priority than other bugs. Exceptions to this rule are for obscure or "the behavior sucks but it's by design and your use case is really just not reasonable" issues. If it's one of the rare exceptional cases, we still treat replying and giving the best suggestion we can as a high priority. And in most cases, we'll still move it to GitHub and tag the issue as a low priority so it's clear how we'll treat it. At least then it's known, documented, and if more people come across it we can change the priority based on feedback. Or a curious OSS contributor can fix it themselves!

If you're not using a direct feedback channel or the Visual Studio one, our GitHub tracker is active and we treat triage with a high priority. At this point the blending of "F# is a product" and "F# is an open source project" occurs. Once you're on GitHub, it's an open source project. If an issue is low priority for us but high priority for you, you'll need to make a compelling case for us to fix it. Otherwise, we strive to make it as easy as possible for you to fix it yourself and unblock your use case. We'll help you along the way, just like we do any other contributor.

On GitHub, as mentioned earlier, we assign issues to milestones that match a Visual Studio release. Although we're an open source project, users of "F# the product" really do want to know when a fix or a new feature will become available. It lets us communicate this to people and acts as a nice historical record too. So, what gets priority? This is where things get interesting.

Managing an open source project that is also a product
Hopefully it's clear by now that the project-product duality of F# means that the two worlds get blended a lot. This manifests itself in several ways.

Most of the time, regressions, crashes, or anything affecting a paying customer gets the top priority for the F# team. Then comes divisional initiatives, like aligning with a .NET release, shipping interop support for a critical new component in .NET, a new tooling experience deemed essential to the .NET product strategy, and so on. Then come longstanding issues that impact your experience a lot, but aren't really a bug so much as a fundamental capability that's missing or designed incorrectly. Finally, the nebulous collection of "medium severity" issues are dealth with, which is kind of our way of acknowledging that a bug is a legit problem we'll try to get it if we can, but not low enough impact that we'll intentionally scope it out all the time.

But we also go the extra mile to ensure that "F# as an open source project" is healthy and improving over time. We're constantly tweaking the build of our repository, and usually improving it along the way. We spend a lot of time removing cruft so contributors have less concepts to understand before they can write code. We flesh out APIs that a handful of other open source projects consume, and extend new features so that they are more "pluggable" for tools like Fable or Bolero. We spend a significant amount of time reviewing contributions and helping people improve their contributions. We [host webinars](https://www.youtube.com/c/fsharporg/videos?view=0&sort=p&shelf_id=2) where we walk through the F# compiler, demonstrate how to fix bugs, and give hints about low-friction ways you can contribute. We also try to document things and keep everything up to date through semi-regular audits of our contribution docs.

Concretely, this means we all work a lot more than just the working hours we put in for Microsoft. Does that mean we're overworked? I guess so. But it's our choice. It would be a hell of a lot easier for the F# team if F# were another closed-source product, like most of what Microsoft produces. But there's no way in hell we'd ever consent to reverting back to the closed-source state that F# the product used to be in. The community we've been able to connect with and help develop over time is essential. It's essential for F#'s standing in the world. It's also essential for our own sanity. Fixing product bugs or doing performance analysis work can actually be quite therapeutic when the person you're fixing it for is positive, engaged, and excited to help. It makes coding fun, and even though it's still "product work" it's rewarding in its own right. That said, we do tend to be a lot more choosy about the things we work on when it's not working hours!

There are two traps we've fallen into in the past, and are sure to continue to fall into in the future. The first is not treating a paying customer's issue with the appropriate amount of priority. In the world of open source, most people are consuming something for free. The maintainers have zero obligation to actually fix their issues. However, that is not true for a customer using a product! Maintainers absolutely have an obligation to fix their issues. The inverse can also be a problem. Treating curious but ultimately unimportant problems that people identify as if they impact paying customers is a dangerous trap to fall into. We try to do our best, but please be aware that this is just part and parcel of being open source.

Have ideas for how to approach this duality better? I'd love to hear about it. You can reach me on twitter or over email.