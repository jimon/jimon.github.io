---
layout: post
title: OSX app in plain C
---

Usually Objective-C / Swift is used for native OSX apps, no wonder most parts of AppKit is written in Objective-C, though sometimes one can observe a struggle to create native apps in [Python](http://blog.adamw523.com/os-x-cocoa-application-python-pyobjc/) / [JS](https://github.com/parmanoir/jscocoa) / [Lua](https://github.com/torus/Lua-Objective-C-Bridge) / etc. Usually this requires some sort of magic dances around Objective-C runtime and language FFI.

Today we gonna talk about creating OSX app in plain C, it should be much more easier then in other languages.

In most cases Objective-C can be seen as syntax sugar on top of C plus runtime framework to support it's features. It's somewhat like CRT but more language-features oriented. With it you can invoke methods, create classes, create new methods, etc. This is what they use in other languages through FFI, but because we use C we don't really need FFI here.

So let me show you the code :

{% highlight objc %}
// this :
[NSApplication sharedApplication];
// is the same as this :
((id (*)(id, SEL))objc_msgSend)((id)objc_getClass("NSApplication"), sel_registerName("sharedApplication"));
{% endhighlight %}

### Run loop

### ARC