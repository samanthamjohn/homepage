---
layout: post
title:  "Fixing your retain cycles for fun and profit"
date:   2019-01-29 6:53am
---

I'm spending this week and next at [Apple Entrepreneur Camp](https://www.apple.com/newsroom/2019/01/apple-entrepreneur-camp-kicks-off-as-app-developer-earnings-hit-new-record/){:target='_blank'}. It’s a fantastic opportunity to learn from Apple engineers. So far, it’s been WWDC on steroids. The least I can do is to spread some of the knowledge I’ve gotten.

#### Retain Cycles in Arrays

When you have two objects that have a reference to one another, you become in danger of creating a [strong reference cycle](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html#ID51){:target='_blank'}. It's easy to mitigate: in Swift, you would make one of the references either _weak_ or _unowned_. In Objective-C, you would make one of the references _weak_.

But, what happens if both objects have a reference to one another through an array?

Here's an example in Hopscotch:

![Interface](/assets/img/retain_cycles/interface.jpg){:class="img-responsive"}

- `HSControl` refers to any block that wraps around another block.
- `HSScript` refers to a list of blocks inside an `HSControl` or a rainbow block such as "Spin".
- `HSControl` can have multiple `HSScript`s as you see in the `check once if/else` block.
- `HSScript` also needs a reference to a list of parent `HSControl`s. So both `HSScript` and `HSControl` require a reference to an array of the other.

You can't have an array of weak or unowned references so it may seem that we are doomed to a strong reference cycle. Never fear! There is a solution, both for Swift and Objective-C.

#### Objective-C

There's a little known (to me) class called [`NSPointerArray`](https://developer.apple.com/documentation/foundation/nspointerarray){:target='_blank'}. It can hold nil values and therefore can accept weak pointers. If you switch your `NSArray` to an `NSPointerArray`, you can solve your reference problems.

Note that you need to bridge your object to add it to the pointer array.
In Xcode 10, I was able to use a fixit to get this syntax.

```
@implementation HSControl
@synthesize _scripts;
- (NSPointerArray *)scripts
{
	if (_scripts) { return _scripts; }

	HSScript *script = [[HSScript alloc] init];
	script.controls = @[self];
	_scripts = [NSPointerArray weakObjectsPointerArray];
	[_scripts addPointer:(__bridge void * _Nullable)(script)];
	return _scripts;
}

- (void)setScripts:(NSPointerArray *)scripts
{
	_scripts = [NSPointerArray weakObjectsPointerArray];
	for (HSScript *script in scripts) {
		[_scripts addPointer:(__bridge void * _Nullable)(script)];
		script.controls = [script.controls arrayByAddingObject:self];
	}
}

@end
```


#### Swift

In the Hopscotch code, one side of the retain cycle was in an Objective-C class (`HSControl`) and the other side in a Swift class, `HSScript`. We decided to look for a fix in Swift as well so we could explore solutions from both sides.

So how do you create a weakly referenced list in Swift? One option is to use an `NSPointerArray`. However, this requires that the items in the array be of type `UnsafeMutableRawPointer`. Yuck. Anytime I see a class called `Unsafe` I try to stay away.

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

A many-to-many reference cycle may seem like a rare occurrence, but if it does happen, now you know what to do. Use this as an excuse to run the [Leaks tool](https://help.apple.com/instruments/mac/current/#/dev022f987b){:target='_blank'} in instruments and see what you have. Even if you are not experiencing this memory leak, I hope you enjoyed learning about a new class and a new design pattern. You never know when they may come in handy.

Lastly, if you were intrigued by that snippet of Hopscotch code, [we're hiring!](https://angel.co/hopscotch/jobs/477379-software-engineer){:target='_blank'}
