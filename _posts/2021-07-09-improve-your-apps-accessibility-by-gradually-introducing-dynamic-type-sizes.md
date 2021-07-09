---
layout: post
title:  "Improve Your Apps Accessibility by Gradually Introducing Dynamic Type Sizes"
tag: [Accessibility, Helper, UIKit]
category: Helper
---

After talking about my UIColor extension helper in my last post, I would like to discuss another helper extension I have used for quite some time now. This time it is not about colors; it is about another topic important to me, which does not get enough attention from product people in my eyes. I am referring to accessibility (short: AX).

Whenever new projects are started, AX is neglected from the beginning: "There is no time for that", "Why should we spend engineering effort on this?", "Our consumers do not need this", etc. But in fact, you are missing out on consumers!

> "Users who have accessibility needs often face an unnecessary amount of friction when using software, and they shouldn't. Simply ensuring everyone can use your app means you let everyone in, and everyone should get to experience the joy of well-built software. I truly believe that. [...] At this point, it's a competitive advantage. There are so many popular apps that simply don't do much of it at all. You can literally sway users from them, to you, by simply opening the door. You're supporting their needs while still supporting your app as a business. I can't think of a more apt win-win." - [Jordan Morgan. _A Best-in-Class iOS App: Book I. Accessibility_](https://www.bestinclassiosapp.com)

To be fair, my helper extension only starts touching a fraction of AX. If you want to learn more about accessibility, you should definitely check out Jordan Morgan's _"A Best-in-Class iOS App"_ book series. You can get it from [https://www.bestinclassiosapp.com](https://www.bestinclassiosapp.com). So, what is it what this ominous helper extension really does? Well, introducing AX to an existing application is a long way. My extension provides static fonts with dynamic type sizes enabled only in DEBUG mode. This allows us to see how the app would behave with other text sizes while developing but still keeps the default text size in production until your app is accessible enough.

```swift
let titleLabel = UILabel()
titleLabel.adjustsFontForContentSizeCategory = true
titleLabel.font = .body
stackView.addArrangedSubview(titleLabel)
```

Of course, there are other ways to gradually introduce AX. You could even enable/disable it on a per-screen basis. However, I found it incredibly simple to just add one UIFont extension and set `adjustsFontForContentSizeCategory` to `true` to gain dynamic type sizes in your application across the board in DEBUG mode. You can find this UIFont extension [here](https://gist.github.com/hoppsen/6b752057b98fa7527ee1f4d4ba05fef0):

<script src="https://gist.github.com/hoppsen/6b752057b98fa7527ee1f4d4ba05fef0.js"></script>
