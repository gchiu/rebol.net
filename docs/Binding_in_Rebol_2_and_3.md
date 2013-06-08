Binding in Rebol 2 and 3
========================

Brian Hawley
11-Feb-2013
Accessed from [stackoverflow](http://stackoverflow.com/questions/14818324)

There isn't really a summary somewhere, so let's go over the basics, perhaps a little more informally than [Bindology][1]. Let Ladislav write a new version of his treatise for R3 and Red. We'll just go over the basic differences, in order of importance.

Object and Function Contexts
----------------------------

*Here's the big difference.*

In R2, there were basically two kinds of contexts: Regular object contexts and `system/words`. Both had static bindings, meaning that once the `bind` function was run, the word binding pointed to a particular object with a real pointer.

The `system/words` context was able to be expanded at runtime to include new words, but all other objects weren't. Functions used regular object contexts, with some hackery to switch out the value blocks when you call the function recursively.

The `self` word was just a regular word that happened to the first one in object contexts, with a display hack to not show the the first word in the context; function contexts didn't have that word, so they didn't display the first regular word properly.

*In R3, almost all of that is different.*

In R3 there are also two kinds of contexts: Regular and stack-local. Regular contexts are used by objects, modules, binding loops, `use`, basically everything but functions, and they are expandable like `system/words` was (yes, "was", we'll get to that). The old fixed-length objects are gone. Functions use stack-local contexts, which (barring any bugs we haven't seen yet) aren't supposed to be expandable, because that would mess up stack frames. As with the old `system/words` you can't shrink contexts, because removing words from a context would break any bindings of those words out there.

If you want to add words to a regular context, you can use `bind/new`, `bind/set`, `resolve/extend` or `append`, or the other functions that call those, depending on what behavior you need. That is new behavior for the `bind` and `append` functions in R3.

Bindings of words to regular and stack-local contexts are static, as before. Value *look-up* is another matter. For regular contexts value look-up is pretty direct, done by a simple pointer indirection to a static block of value slots. For stack-local contexts the value block is linked to by the stack frame and referenced from there, so to find the right frame you have to do a stack walk that is O(stack-depth). See [bug #1946][2] for details - we'll get into why later.

Oh, and `self` isn't a regular word anymore, it's a binding trick, a keyword. When you bind blocks of words to object or module contexts, it binds the keyword `self` which evaluates to be a reference to the context. However, there is an internal flag that can be set which says that a context is "selfless", which turns that `self` keyword off. When that keyword is turned off, you can actually use the word `self` as a field in your context. Binding loops, `use` and function contexts set the selfless flag for their contexts, and the `selfless?` function checks for that.

This model was refined and documented in a fairly involved CureCode flame war, much like R2's model was documented by a REBOL mailing list flame war back in 1999-2000. :-)

Functions vs. Closures
----------------------

When I was talking about stack-local function contexts above, I meant the contexts used by `function!` type functions. R3 has a lot of function types, but most of them are native functions in one way or another, and native functions don't use these stack-local contexts (though they do get stack frames). The only function types that are for Rebol code are `function!` and a new `closure!` type. Closures are very different from regular functions.

When you create a `function!`, you're creating a function. It constructs a stack-local context, binds the code body to it, and bundles together the code body and the spec. When you call the function it makes a stack frame with a reference to the function's context and runs the code block. If it has access words in the function context it does the stack walk to find the right frame and then gets the values from there. Fairly straight-forward.

When you create a `closure!`, on the other hand, you create a *function builder*. It sets up the spec and function body pretty much the same as `function!`, but when you *call* a closure it makes a new *regular* selfless context, then does a `bind/copy` of the body, changing all references to the function context to be references to the new regular context in the copy. Then, when it does the copied body, all closure word references are as static as those of object contexts.

Another difference between the two is in how they behave before the function is running, while the function is running, and after the function is done running.

In R2, `function!` contexts still exist when the function isn't running, but the value block of the top-level call of the function still persists too. Only recursive calls get the new value blocks, the top-level call keeps a persistent value block Like I said, hackery. Worse, the top-level value block isn't cleared when the function returns, so you better make sure that you aren't referencing anything sensitive or that you want recycled when the function returns (use the `also` function to clean up, that's what I made it for).

In R3, the `function!` contexts still exist when the function isn't running, but the value block doesn't exist at all. All function calls act like recursive calls did in R2, except better because it's designed that way all the way down, referencing the stack frame instead. The scope of that stack frame is dynamic (stalk a Lisp fan if you need the history of that), so as long as the function is running on the current stack (yes, "current", we'll get to that), you can use one of its words to get at the values of the *most recent call* of that function. Once all of the nested calls of the function return, there won't be any values in scope, and you'll just trigger an error (the wrong error, but we'll fix that). 

There's also a useless restriction on binding to an out-of-scope function word that is on my todo list to fix soon. See [bug #1893][3] for details.

For `closure!` functions, before the closure runs *the context doesn't exist at all*. Once the closure starts running, the context is created and exists persistently. If you call the closure again, or recursively, *another* persistent context is created. Any word that leaks from a closure only refers to the context created during that particular run of the closure.

You can't get a word bound to a function or closure context in R3 when the function or closure isn't running. For functions, this is a security issue. For closures, this is a definitional issue.

Closures were considered so useful that Ladislav and I both ported them to R2, independently at different times resulting in similar code, weirdly enough. I think Ladislav's version predated R3, and served as the inspiration for R3's `closure!` type; my version was based on testing the external behavior of that type and trying to replicate it in R2 for R2/Forward, so it's amusing that the solution for `closure` ended up being so similar to Ladislav's original, which I didn't see until much later. My version got included in R2 itself starting with 2.7.7, as the `closure`, `to-closure` and `closure?` functions, and the `closure!` word is assigned the same type value as `function!` in R2.

Global vs. Local Contexts
-------------------------

*Here's where things get really interesting.*

In [Bindology][4] there was a fairly large amount of the article talking about the distinction between the "global" context (which turned out to be `system/words`) and "local" contexts, a fairly important distinction for R2. In R3, *that distinction is irrelevant*.

In R3, `system/words` is *gone*. There is no one "global" context. All regular contexts are "local" in the sense that was meant in R2, which makes that meaning of "local" useless. For R3, we need a new set of terms.

For R3, the only difference that matters is whether contexts are *task-relative*, so the only useful meaning for "global" contexts are the ones that are not directly task-relative, and "local" contexts are the ones that are task-relative. A "task" in this case would be the `task!` type, which is basically an OS thread in the current model.

In R3, at the moment, the only things so far that are task-relative (barely) are the stack variables, which means that stack-relative function contexts are supposed to be *task-relative* too. This is why the [stack walk][5] is necessary, because otherwise we'd need to keep and maintain TLS pointers in *every single function context*. All *regular* contexts are global.

Another thing to consider is that according to the plan (which is mostly unimplemented so far), the user context `system/contexts/user` and `system` itself are also intended to be task-relative, so even by R3 standards they would be considered "local". And since `system/contexts/user` is basically the closest thing that R3 has to R2's `system/words`, that means that what *scripts* think of as being their "global" context is actually supposed to be *task-local* in R3.

R3 does have a couple of system global contexts called `sys` and `lib`, though they are used quite differently from R2's global context. Also, all module contexts are global.

It is possible (and common) for there to be globally defined contexts that are only referenced from task-local root references, so that would make those contexts in effect *indirectly task-local*. This is what usually happens when binding loops, `use`, closures or private modules are called from "user code", which basically means non-module scripts, which get bound to `system/contexts/user`. Technically, this is also the case for functions called from modules as well (since functions are stack-local) but those references often eventually get assigned to module words, which are global.

No, we don't yet have synchronization either. Still, that's the model that R3's design is supposed to eventually have, and partly does already. See the [module binding article][6] for more details.

As a bonus, R3 has a real symbol table now instead of using `system/words` as an ad-hoc symbol table. This means that the word limit R2 used to hit pretty quickly is effectively gone in R3. I'm not aware of any app that has reached the new limit or even determined how high the limit is, though it is apparently well over many million distinct symbols. We should check the source to figure that out, now that we have access it it.

LOAD and USE
------------

Minor details. The `use` function initializes its words with `none` instead of leaving them unset. And because there is no "global" context the way there is in R2, `load` doesn't necessarily bind words at all. Which context `load` binds to depends on circumstances mentioned in the [module binding article][6], though unless you specify otherwise it explicitly binds the words to `system/contexts/user`. And both are now mezzanine functions.

Spelling and Aliases
--------------------

R3 is basically the same as R2 in this, word bindings are by default case-insensitive. The words themselves are case-preserving, and if you compare them using case-sensitive methods you will see differences between words that differ only in case.

In object or function contexts though, when a word is mapped to a value slot, then another word is bound to that context or looked up at runtime, words that differ only by case are considered to be effectively the same word and map to the same value slot.

However, it was found that explicitly created aliases made with the `alias` function where the spelling of aliased words differed in other ways than just by case *broke object and function contexts really drastically*. In R2 they resolved these problems in `system/words`, which made `alias` merely too awkward to use in anything other than a demo, rather than actively dangerous.

Because of this, we removed the externally visible `alias` function altogether. The internal aliasing facility still works because it only aliases words that would normally be considered equivalent for context lookup, which means that contexts don't break. But we now recommend that localization and the other tricks that `alias` was used for in demos, if never in practice, be done using the old-fashioned method of assigning values to another word with the new spelling.

Word Types
----------

The `issue!` type is now a word type. You can bind it. So far, no-one has taken advantage of being able to bind issues, instead just using the increased speed of operations.

That's pretty much it, for the intentional changes. Most of the rest of the differences might be side effects of the above, or maybe even bugs or not-yet-implemented features. There might even be some R2-like behavior in R3 that is also the result of bugs or not-yet-implemented features. When in doubt, ask first.

  [1]: http://www.rebol.net/wiki/Bindology "Bindology"
  [2]: http://issue.cc/r3/1946 "Function variable access is O&#40;n&#41; depending on the stack depth"
  [3]: http://issue.cc/r3/1893 "BIND stuff function-bound-word restriction has no benefit, should be removed"
  [4]: http://www.rebol.net/wiki/Bindology "Bindology"
  [5]: http://issue.cc/r3/1946 "Function variable access is O&#40;n&#41; depending on the stack depth"
  [6]: http://stackoverflow.com/questions/14420942/how-are-words-bound-within-a-rebol-module/14552755 "How are words bound within a Rebol module?"