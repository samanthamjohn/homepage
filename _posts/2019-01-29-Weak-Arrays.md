---
layout: post
title:  "Fixing your retain cycles for fun and profit"
date:   2019-01-29 6:53am
---

I'm spending this week and next at [Apple Entrepreneur Camp](https://www.apple.com/newsroom/2019/01/apple-entrepreneur-camp-kicks-off-as-app-developer-earnings-hit-new-record/){:target='_blank'}. It's an amazing opportunity to learn from Apple engineers, so far it's basically been WWDC on steroids. So the least I can do is to spread some of the great knowledge I've gotten. Here's a tidbit I learned yesterday.

#### Retain Cycles in Arrays

When you have two objects that have a reference to one another, you become in danger of creating a [strong reference cycle](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html#ID51). It's easy to mitigate: in Swift, you would make one of the references either _weak_ or _unowned_. In Objective-C, you would make one of the references _weak_.

But, what happens if both objects have a reference to one another through an array?

Here's an example in Hopscotch:

![Interface](/assets/img/retain_cycles/interface.jpg){:class="img-responsive"}

`HSControl` refers to any block that wraps around another block.
`HSScript` refers to list of blocks inside an `HSControl`, or a rainbow block such as "Spin".
`HSControl` can have multiple `HSScript`s as you see in the `check once if/else` block. `HSScript` also needs a reference to a list of parent `HSControl`. So both `HSScript` and `HSControl` require a reference to array of the other.

You can't have an array of weak or unowned references so it may seem that we are doomed to a strong reference cycle. But never fear! There is a solution, both for Swift and Objective C.

#### Objective-C

There's a little known (to me) class called [`NSPointerArray`](https://developer.apple.com/documentation/foundation/nspointerarray){:target='_blank'}. It can hold nil values and therefore can accept weak pointers. If you switch your `NSArray` to an `NSPointerArray` you can solve your reference problems. Better yet, you can use an `NSPointerArray` internally, while publicly exposing an `NSArray`  thus keeping a clean API.

Note that you will need to bridge your object in order to add it to the pointer array.
In Xcode 10, I was able to use a fixit to get this syntax.

```
@interface HSControl()
@property (nonatomic, strong) NSPointerArray *weakScripts;
@end

@implementation HSControl
- (NSArray *)scripts
{
	if (self.weakScripts) { return [self.weakScripts allObjects]; }

	HSScript *script = [[HSScript alloc] initWithContext:self.managedObjectContext];
	script.controls = @[self];
	self.weakScripts = [NSPointerArray weakObjectsPointerArray];
	[self.weakScripts addPointer:(__bridge void * _Nullable)(script)];
	return [self.weakScripts allObjects];
}

- (void)setScripts:(NSArray *)scripts
{
	self.weakScripts = [NSPointerArray weakObjectsPointerArray];
	for (HSScript *script in scripts) {
		[self.weakScripts addPointer:(__bridge void * _Nullable)(script)];
		script.controls = [script.controls arrayByAddingObject:self];
	}
}

...

@end
```


#### Swift

In the Hopscotch code, it turned out that one side of the retain cycle happened in an Objective-C class (`HSControl`) and the other side in a Swift class `HSScript`. This meant that we got to explore solutions from both sides.

So how do you create a weakly referenced list in Swift? You *can* use an `NSMutablePointerArray`, but it requires that the items in the array be of type `UnsafeMutableRawPointer`. Yuck. Anytime I see a class called `Unsafe`, I try to stay away.

A better solution is to wrap your object in a struct with an `unowned` reference.

```
@objc public class HSScript : NSObject {
  @objc public var controls: [HSControl] {
    set {
      self.unownedControls = newValue.map { UnownedControl(control: $0) }
    }
     get {
      return self.unownedControls.compactMap { $0.control }
    }
  }

  private var unownedControls: [UnownedControl] = []

  private struct UnownedControl {
    unowned let control: HSControl
  }
}
```

In this instance, we create a private struct that keeps a reference to the HSControl. It's the same principle as creating an array of pointers, just this time we made the pointers ourselves.

#### Conclusion

A many-to-many reference cycle may seem like a rare occurrence, but if it does happen, now you know what to do. And even if you are not experiencing this memory leak, it's always nice to get a new class and a new design pattern into your tool-belt. You never know when they may come in handy.
