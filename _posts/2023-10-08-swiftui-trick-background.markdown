---
layout: post
title:  "SwiftUI Trick. How to align a list of numbers with different sizes."
date:   2023-10-08 09:20:14 +0200
categories: jekyll update
---

Sometimes you want to show a title with a background in a list, for example, and you have a problem with the size of the background. Imagine that you want to show a digit but when you change the number 9 for the number 10, the background size becomes greater than the previous.

# The solution

There is a trick in this context and this is so simple. You need to put a ZStack with a hidden view.

```
ZStack(alignment: .trailing) {
    Text("999")
        .hidden()
    Text("\(idx)")
}
.padding(.horizontal)
.padding(.vertical, 4)
.background {
    RoundedRectangle(cornerRadius: 8, style: .continuous)
        .fill(.regularMaterial)
}
```