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

Let's get a bit into details how this works exactly.

- ```objc_msgSend``` is a C function from objc runtime, used for sending messages
- ```((id (*)(id, SEL))objc_msgSend)``` makes a C cast to required type, so our C compiler can generate compliant Objective-C ABI function call. On OSX for 64 bit binaries we use something called ```System V Application Binary Interface AMD64 Architecture Processor Supplement``` document which describes how exactly to call a "native" function using CPU instructions and registers. Because objc runtime only provides a few declarations of ```objc_msgSend``` function, we need to convert it explicitly to required type, you can look at it as some sort of dynamic function overloading mechanism.
- ```(id)objc_getClass("NSApplication")``` just get's objc class itself, and casts it to ```id``` type, which itself is just a pointer to ```objc_object```. Kinda similar to handle types in other API's.
- ```sel_registerName("sharedApplication")``` returns pointer to objc_selector structure, which later used to figure out what needs to be called.

You can find all Objective C runtime implementation details on [Apple's open source site](http://opensource.apple.com/source/objc4/objc4-680/runtime/).

So that said let's create a simple OSX app in Objective-C first. As many people advocate to use full blown AppKit just to create simplest apps, let's go other way around - use as little frameworks functionality as possible and see what happens. To make life easier let's base our code on [Minimalist Cocoa programming](http://www.cocoawithlove.com/2010/09/minimalist-cocoa-programming.html) tutorial :

{% highlight objc %}
#import <Cocoa/Cocoa.h>

int main ()
{
	NSAutoreleasePool * pool = [[NSAutoreleasePool alloc] init];
	[NSApplication sharedApplication];
	[NSApp setActivationPolicy:NSApplicationActivationPolicyRegular];
	id menubar = [[NSMenu new] autorelease];
	id appMenuItem = [[NSMenuItem new] autorelease];
	[menubar addItem:appMenuItem];
	[NSApp setMainMenu:menubar];
	id appMenu = [[NSMenu new] autorelease];
	id appName = [[NSProcessInfo processInfo] processName];
	id quitTitle = [@"Quit " stringByAppendingString:appName];
	id quitMenuItem = [[[NSMenuItem alloc] initWithTitle:quitTitle
		action:@selector(terminate:) keyEquivalent:@"q"] autorelease];
	[appMenu addItem:quitMenuItem];
	[appMenuItem setSubmenu:appMenu];
	id window = [[[NSWindow alloc] initWithContentRect:NSMakeRect(0, 0, 200, 200)
		styleMask:NSTitledWindowMask backing:NSBackingStoreBuffered defer:NO]
			autorelease];
	[window cascadeTopLeftFromPoint:NSMakePoint(20,20)];
	[window setTitle:appName];
	[window makeKeyAndOrderFront:nil];
	[NSApp activateIgnoringOtherApps:YES];
	[NSApp run];
	[pool drain];
	return 0;
}
{% endhighlight %}

Pretty simple, now let's make it with objc_msgSend calls.

{% highlight objc %}
#import <Cocoa/Cocoa.h>
#define objc_msgSend_id				((id (*)(id, SEL))objc_msgSend)
#define objc_msgSend_void			((void (*)(id, SEL))objc_msgSend)
#define objc_msgSend_void_id		((void (*)(id, SEL, id))objc_msgSend)
#define objc_msgSend_void_bool		((void (*)(id, SEL, BOOL))objc_msgSend)
#define objc_msgSend_id_const_char	((id (*)(id, SEL, const char*))objc_msgSend)

int main ()
{
	id poolAlloc = objc_msgSend_id((id)objc_getClass("NSAutoreleasePool"), sel_registerName("alloc"));
	id autoreleasePool = objc_msgSend_id(poolAlloc, sel_registerName("init"));

	objc_msgSend_id((id)objc_getClass("NSApplication"), sel_registerName("sharedApplication"));
	((void (*)(id, SEL, NSInteger))objc_msgSend)(NSApp, sel_registerName("setActivationPolicy:"), 0);

	id menubarAlloc = objc_msgSend_id((id)objc_getClass("NSMenu"), sel_registerName("alloc"));
	id menubar = objc_msgSend_id(menubarAlloc, sel_registerName("init"));
	objc_msgSend_void(menubar, sel_registerName("autorelease"));
	
	id appMenuItemAlloc = objc_msgSend_id((id)objc_getClass("NSMenuItem"), sel_registerName("alloc"));
	id appMenuItem = objc_msgSend_id(appMenuItemAlloc, sel_registerName("init"));
	objc_msgSend_void(appMenuItem, sel_registerName("autorelease"));
	
	objc_msgSend_void_id(menubar, sel_registerName("addItem:"), appMenuItem);
	((id (*)(id, SEL, id))objc_msgSend)(NSApp, sel_registerName("setMainMenu:"), menubar);

	id appMenuAlloc = objc_msgSend_id((id)objc_getClass("NSMenu"), sel_registerName("alloc"));
	id appMenu = objc_msgSend_id(appMenuAlloc, sel_registerName("init"));
	objc_msgSend_void(appMenu, sel_registerName("autorelease"));

	id processInfo = objc_msgSend_id((id)objc_getClass("NSProcessInfo"), sel_registerName("processInfo"));
	id appName = objc_msgSend_id(processInfo, sel_registerName("processName"));

	id quitTitlePrefixString = objc_msgSend_id_const_char((id)objc_getClass("NSString"), sel_registerName("stringWithUTF8String:"), "Quit ");
	id quitTitle = ((id (*)(id, SEL, id))objc_msgSend)(quitTitlePrefixString, sel_registerName("stringByAppendingString:"), appName);

	id quitMenuItemKey = objc_msgSend_id_const_char((id)objc_getClass("NSString"), sel_registerName("stringWithUTF8String:"), "q");
	id quitMenuItemAlloc = objc_msgSend_id((id)objc_getClass("NSMenuItem"), sel_registerName("alloc"));
	id quitMenuItem = ((id (*)(id, SEL, id, SEL, id))objc_msgSend)(quitMenuItemAlloc, sel_registerName("initWithTitle:action:keyEquivalent:"), quitTitle, sel_registerName("terminate:"), quitMenuItemKey);
	objc_msgSend_void(quitMenuItem, sel_registerName("autorelease"));

	objc_msgSend_void_id(appMenu, sel_registerName("addItem:"), quitMenuItem);
	objc_msgSend_void_id(appMenuItem, sel_registerName("setSubmenu:"), appMenu);

	NSRect rect = { {0, 0}, {200, 200} };
	id windowAlloc = objc_msgSend_id((id)objc_getClass("NSWindow"), sel_registerName("alloc"));
	id window = ((id (*)(id, SEL, NSRect, NSUInteger, NSUInteger, BOOL))objc_msgSend)(windowAlloc, sel_registerName("initWithContentRect:styleMask:backing:defer:"), rect, 15, 2, NO);
	objc_msgSend_void(window, sel_registerName("autorelease"));

	NSPoint point = {20, 20};
	((void (*)(id, SEL, NSPoint))objc_msgSend)(window, sel_registerName("cascadeTopLeftFromPoint:"), point);

	objc_msgSend_void_id(window, sel_registerName("setTitle:"), appName);
	objc_msgSend_void_id(window, sel_registerName("makeKeyAndOrderFront:"), nil);
	objc_msgSend_void_bool(NSApp, sel_registerName("activateIgnoringOtherApps:"), YES);

	objc_msgSend_void(NSApp, sel_registerName("run:"));

	objc_msgSend_void(autoreleasePool, sel_registerName("drain"));
	return 0;
}
{% endhighlight %}

This is still Objective-C (non ARC), but we are very close to make it in plain C.

First we need to solve the memory management thing, funny said, but Apple's [Advanced Memory Management Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html) states ```If you use ARC, there is typically no need to understand the underlying implementation described in this document, although it may in some situations be helpful.```, using tools without understanding how they work usually is a bad sign IMHO.

So what's up with ARC here ? If we want to compile that code under ARC enabled Objective-C it will fail to build because we manually call autorelease. If you want to make it work under ARC just remove autorelease calls and replace ```autoreleasePool``` with ```@autorelease { ... }``` construct.

Second we still include ```Cocoa.h```, which is Objective-C only header, we need to remove that dependency to make it a plain C code. ```NSPoint```, ```NSSize``` and ```NSRect``` are simple typedefs to CoreGraphics types (which are plain C), and NSApp is just an extern variable! Should be very easy :

{% highlight objc %}
#ifdef __OBJC__
#import <Cocoa/Cocoa.h>
#else
#include <CoreGraphics/CGBase.h>
#include <CoreGraphics/CGGeometry.h>
typedef CGPoint NSPoint;
typedef CGSize NSSize;
typedef CGRect NSRect;

extern id NSApp;
#endif
{% endhighlight %}
