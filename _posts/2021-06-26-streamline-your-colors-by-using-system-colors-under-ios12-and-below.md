---
layout: post
title: "Streamline Your Colors by Using System Colors under iOS 12 and Below"
tags: [Helper, UIKit]
category: Helper
---

At the time of writing, this might already be close to not being needed anymore soon. I am talking about getting Apple's (not so) new System Colors under iOS 12 and below.

> What I mean by not being needed anymore is that with the release of iOS 15 in September, most apps will drop the support for iOS 12, given they follow the approach of supporting the current and one or two major releases before.

If your app is still supporting iOS 12, the following code snippet will result in a compiler error saying that `'label' is only available with iOS 13 or newer`:

```diff
let titleLabel = UILabel()
- titleLabel.textColor = .label // ERROR: 'label' is only available with iOS 13 or newer`
stackView.addArrangedSubview(titleLabel)
```

To fix this compiler error you could introduce a `if #available` check:

```swift
let titleLabel = UILabel()
if #available(iOS 13.0, *) {
 titleLabel.textColor = .label
} else {
 titleLabel.textColor = .black
}
stackView.addArrangedSubview(titleLabel)
```

However, this would unnecessarily inflate the codebase by introducing code duplications of multiple of those `if #available` checks; we could easily create a helper extension containing all those new shiny colors mentioned here
[https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/color/](https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/color/)
and access them quickly across the project without checking for the iOS version.

Luckily for you, I already did this by copying the values from [Apple's website](https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/color/), or making use of my UIColor playground published [here](https://github.com/hoppsen/uicolors). Recently I put this UIColor helper extension into a [gist](https://gist.github.com/hoppsen/f3cbe2dd51ad40cfe80609b2e9bebada):

<script src="https://gist.github.com/hoppsen/f3cbe2dd51ad40cfe80609b2e9bebada.js"></script>
