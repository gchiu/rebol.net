Overview of the Rebol 3 Module System Design
============================================

Brian Hawley, 27 Jan 2013

Introduction
------------

In Rebol 3 there are no such things as system words, there are just words. Some words have been added to the runtime library `lib`, and `set` is one of those words, which happens to have a function assigned to it. Modules import words from `lib`, though what "import" means depends on the module options. That might be more tricky than you were expecting, so let me explain.

Regular Modules
---------------

For starters, I'll go over what importing means for "regular" modules, ones that don't have any options specified. Let's start with your first module:

    REBOL [Type: 'module] set 'foo "Bar"

First of all, you have a wrong assumption here: The word `foo` is *not* local to the module, it's just the same as `set`. If you want to define `foo` as a local word you have to use the same method as you do with objects, use the word as a set-word at the top level, like this:

    REBOL [Type: 'module] foo: "Bar"

The only difference between `foo` and `set` is that you hadn't exported or added the word `foo` to `lib` yet. When you reference words in a module that you haven't declared as local words, it has to get their values and/or bindings from somewhere. For regular modules, it binds the code to `lib` first, then overrides that by binding the code again to the module's local context. Any words defined in the local context will be bound to it. Any words not defined in the local context will retain their old bindings, in this case to `lib`. That is what "importing" means for regular modules.

In your first example, assuming that you haven't done so yourself, the word `foo` was not added to the runtime library ahead of time. That means that `foo` wasn't bound to `lib`, and since it wasn't declared as a local word it wasn't bound to the local context either. So as a result, `foo` wasn't bound to anything at all. In your code that was an error, but in other code it might not be.

Isolated Modules
----------------

There is an "isolate" option that changes the way that modules import stuff, making it an "isolated" module. Let's use your second example here:

    REBOL [Type: 'module Options: [isolate]] set 'foo "Bar"

When an isolated module is made, every word in the module, even in nested code, is collected into the module's local context. In this case, it means that `set` and `foo` are local words. The initial values of those words are set to whatever values they have in `lib` at the time the module is created. That is, if the words are defined in `lib` at all. If the words don't have values in `lib`, they won't initially have values in the module either.

It is important to note that this import of values is a one-time thing. After that initial import, any changes to these words made outside the module don't affect the words in the module. That is why we say the module is "isolated". In the case of your code example, it means that someone could change `lib/set` and it wouldn't affect your code.

But there's another important module type you missed...

Scripts
-------

In Rebol 3, scripts are another kind of module. Here's your code as a script:

    REBOL [] set 'foo "Bar"

Or if you like, since script headers are optional in Rebol 3:

    set 'foo "Bar"

Scripts also import their words from `lib`, and they import them into an isolated context, but with a twist: All scripts share the *same* isolated context, known as the "user" context. This means that when you change the value of a word in a script, the *next* script to use that word will see the change when it starts. So if after running the above script, you try to run this one:

    print foo

Then it will print "Bar", rather than have `foo` be undefined, even though `foo` is still not defined in `lib`. You might find it interesting to know that if you are using Rebol 3 interactively, entering commands into the console and getting results, that every command line you enter is a separate script. So if your session looks like this:

    >> x: 1
    == 1
    >> print x
    1


The `x: 1` and `print x` lines are separate scripts, the second taking advantage of the changes made to the user context by the first.

The user context is actually supposed to be task-local, but for the moment let's ignore that.

Why the difference?
-------------------

Here is where we get back to the "system function" thing, and that Rebol doesn't have them. The `set` function is just like any other function. It might be implemented differently, but it's still a normal value assigned to a normal word. An application will have to manage a *lot* of these words, so that's why we have modules and the runtime library.

In an application there will be stuff that needs to change, and other stuff that needs to *not* change, and which stuff is which depends on the application. You will want to group your stuff, to keep things organized or for access control. There will be globally defined stuff, and locally defined stuff, and you will want to have an organized way to get the global stuff to the local places, and vice-versa, and resolve any conflicts when more than one thing wants to define stuff with the same name.

In Rebol 3, we use modules to group stuff, for convenience and access control. We use the runtime library `lib` as a place to collect the exports of the modules, and resolve conflicts, in order to control what gets imported to the local places like other modules and the user context(s). If you need to override some stuff, you do this by changing the runtime library, and if necessary propagating your changes out to the user context(s). You can even upgrade modules at runtime, and have the new version of the module override the words exported by the old version.

For regular modules, when things are overridden or upgraded, your module will benefit from such changes. Assuming those changes are a benefit, this can be a good thing. A regular module cooperates with other regular modules and scripts to make a shared environment to work in.

However, sometimes you need to stay separate from these kinds of changes. Perhaps you need a particular version of some function and don't want to be upgraded. Perhaps your module will be loaded in a less trustworthy environment and you don't want your code hacked. Perhaps you just need things to be more predictable. In cases like this, you may want to isolate your module from these kinds of external changes.

The downside to being isolated is that, if there are changes to the runtime library that you might want, you're not going to get them. If your module is somehow accessible (such as by having been imported with a name), someone might be able to propagate those changes to you, but if you're not accessible then you're out of luck. Hopefully you've thought to monitor `lib` for changes you want, or reference the stuff through `lib` directly.

Still, we've missed another important issue...

Exporting
---------

The other part of managing the runtime library and all of these local contexts is exporting. You have to get your stuff out there somehow. And the most important factor is something that you wouldn't suspect: whether or not your module has a name.

Names are optional for Rebol 3's modules. At first this might seem like just a way to make it simpler to write modules (and in Carl's original proposal, that is exactly why). However, it turns out that there is a lot of stuff that you can do when you have a name that you can't when you don't, simply because of what a name is: a way to refer to something. If you don't have a name, you don't have a way to refer to something.

It might seem like a trivial thing, but here are some things that a name lets you do:

* You can tell whether a module is loaded.
* You can make sure a module is only loaded once.
* You can tell whether an older version of a module was there earlier, and maybe upgrade it.
* You can get access to a module that was loaded earlier.

When Carl decided to make names optional, he gave us a situation where it would be possible to make modules for which you couldn't do any of those things. Given that module exports were intended to be collected and organized in the runtime library, we had a situation where you could have effects on the library that you couldn't easily detect, and modules that got reloaded every time they were imported.

So for safety we decided to cut out the runtime library completely and just export words from these unnamed modules directly to the local (module or user) contexts that were importing them. This makes these modules effectively private, as if they are owned by the target contexts. We took a potentially awkward situation and made it a feature.

It was such a feature that we decided to support it explicitly with a `private` option. Making this an explicit option helps us deal with the last problem not having a name caused us: making private modules not have to reload over and over again. If you give a module a name, its exports can still be private, but it only needs one copy of what it's exporting.

However, named or not, private or not, that is 3 export types.

Regular Named Modules
---------------------

Let's take this module:

    REBOL [type: module name: foo] export bar: 1

Importing this adds a module to the loaded modules list, with the default version of 0.0.0, and exports one word `bar` to the runtime library. "Exporting" in this case means adding a word `bar` to the runtime library if it isn't there, and setting that word `lib/bar` to the value that the word `foo/bar` has after `foo` has finished executing (if it isn't set already).

It is worth noting that this automatic exporting happens only once, when the body of `foo` is finished executing. If you make a change to `foo/bar` after that, that doesn't affect `lib/bar`. If you want to change `lib/bar` too, you have to do it manually.

It is also worth noting that if `lib/bar` already exists before `foo` is imported, you won't have another word added. And if `lib/bar` is already set to a value (not unset), importing `foo` won't overwrite the existing value. First come, first served. If you want to override an existing value of `lib/bar`, you'll have to do so manually. This is how we use `lib` to manage overrides.

The main advantages that the runtime library gives us is that we can manage all of our exported words in one place, resolving conflicts and overrides. However, another advantage is that most modules and scripts don't actually have to say what they are importing. As long as the runtime library is filled in properly ahead of time with all the words you need, your script or module that you load later will be fine. This makes it easy to put a bunch of import statements and any overrides in your startup code which sets up everything the rest of your code will need. This is intended to make it easier to organize and write your application code.

Named Private Modules
---------------------

In some cases, you don't want to export your stuff to the main runtime library. Stuff in `lib` gets imported into everything, so you should only export stuff to `lib` that you want to make generally available. Sometimes you want to make modules that only export stuff for the contexts that want it. Sometimes you have some related modules, a general facility and a utility module or so. If this is the case, you might want to make a private module.

Let's take this module:

    REBOL [type: module name: foo options: [private]] export bar: 1

Importing this module doesn't affect `lib`. Instead, its exports are collected into a private runtime library that is local to the module or user context that is importing this module, along with those of any other private modules that the target is importing, then imported to the target from there. The private runtime library is used for the same conflict resolution that `lib` is used for. The main runtime library `lib` takes precedence over the private lib, so don't count on the private lib overriding global things.

This kind of thing is useful for making utility modules, advanced APIs, or other such tricks. It is also useful for making strong-modular code which requires explicit imports, if that is what you're into.

It's worth noting that if your module doesn't actually export anything, there is no difference between a named private module or a named public module, so it's basically treated as public. All that matters is that it has a name. Which brings us to...

Unnamed Modules
---------------

As explained above, if your module doesn't have a name then it pretty much has to be treated as private. More than private though, since you can't tell if it's loaded, you can't upgrade it or even keep from reloading it. But what if that's what you want?

In some cases, you really want your code run for effect. In these cases having your code rerun every time is what you want to do. Maybe it's a script that you're running with `do` but structuring as a module to avoid leaking words. Maybe you're making a mixin, some utility functions that have some local state that needs initializing. It could be just about anything.

I frequently make my `%rebol.r` file an unnamed module because I want to have more control over what it exports and how. Plus, since it's done for effect and doesn't need to be reloaded or upgraded there's no point in giving it a name.

No need for a code example, your earlier ones will act this way.
