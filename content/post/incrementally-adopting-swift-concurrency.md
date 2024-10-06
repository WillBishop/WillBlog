+++
title = "Incrementally adopting strict concurrency in a preexisting project"
description = "How I'm adopting strict concurrency in Pestle, as a UIKit codebase that is multiple years old."
date = 2024-10-06T02:13:50Z
author = "Will Bishop"
+++

Beginning with Xcode 16, Swift 6 is the new default language mode for new projects. For existing projects however, it's likely you're running Swift 5.10 or earlier.

One of the biggest changes to Swift in version 6 is the new concurrency model, which enforces thread safety at compile time, rather than crashing or causing hard to diagnose bugs at run time. 

It's a lofty goal from the Swift team, and one that I think is a step in the right direction, _however_, and this is a big hurdle, **strict concurrency is hard**. Adopting it in a new codebase is _more_ straightforward (though the error messages are still inscruitable, not helped by [the decline of StackOverflow](https://observablehq.com/@ayhanfuat/the-fall-of-stack-overflow)), but migrating old codebases, especially in cases when Apple themselves have not updated system libraries, is considerably harder. 

Nonetheless, much like SwiftUI will eventually replace UIKit, the writing is on the wall, and eventually (in many years) all codebases will need to compile against Swift 6 to get new language features.

## A crossroads

[Pestle](https://pestlechef.app), if you're unaware, is my recipe management app. It's primarily a UIKit app, with a codebase dating back to 2021 (which is before Swift even had `async`!). 

Flicking on strict concurrency checking in the Swift 5.10 language mode quickly produced a littany of errors, which left me in a tricky spot. 

I could:

A) Turn it on and spent weeks picking apart all the errors and fixing the entire codebase, or 

B) Devise a way to adopt it one file at a time, and then take a detour to write an entirely new blog website and blog post (hello).

**I chose B.**

### SPM to the recuse

The plan was pretty simple:

1. Create two new packages,`PestleUI`and`PestleCore`, both using 

```
// swift-tools-version: 6.0
```

![Creating a new package](/images/newpackage.png)
![A screenshot of Xcode showing the two new packages](/images/packages.png)

2. Add both as a dependency to my main project

3. Then in my `AppDelegate.swift` I could do

```
@_exported import PestleCore
@_exported import PestleUI
```

The reason for `@_exported`, in case you haven't seen it before, is that it allows you to import a dependency module wide. Which meant if I had a class, `ShoppingListViewController`, which was referenced in a couple places which I planned to move to `PestleUI`, I wouldn't have to `import PestleUI` in every file which referenced that controller.

And just like that, I had a place to write new code, which I'd be able to reference from my main `Pestle` module without constant imports, which is guaranteed to be thread safe and able to adopt all new language features as they come out.

![A screenshot of Xcode showing an error in a function from writing thread unsafe code.](/images/unsafefunction.png)

------

So that's how I'm writing code in Pestle moving forward, it'll ensure that for years to come Pestle will continue to be able to adopt new language features and should result in safer and more reliable code. 

I hope you liked my first blog post on my new blog, I hope to write here more often, perhaps about my experience writing a demo app for Apple Retail?
