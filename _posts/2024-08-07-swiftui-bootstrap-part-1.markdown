---
layout: post
title:  "SwiftUI Bootstrap. Initial project."
date:   2024-08-07 20:20:14 +0200
categories: swiftui bootstrap english
---

I think you know Bootstrap for web development. The first versions was named Twitter Bottstrap, now it is open sourced and free to use.

My idea is to create a design system for Apple Ecosystems, not only for iOS platform that represents the same idea of Bootstrap.

For example, Alerts is a good example.

{% highlight html %}
<div class="alert alert-primary" role="alert">
  A simple primary alert—check it out!
</div>
{% endhighlight %}

Alerts it is a simple text with a background where represents a semantic color like primary, secondary, success, danger, warning, and so on.

# Accessibility

Another feature for a good design system is accessibility. In these days, accessibility is a mandatory feature for all apps.

# Dynamic Settings

If we want to reuse these components in multiple projects, we need to create the way to inject custom colors, custom assets in a easy way. Here, one solution is to use Tuist. Tuist can manage multiple projects and multiple schemes and we can select specific target for settings.