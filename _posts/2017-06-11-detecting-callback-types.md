---
layout: post
title:  "Detecting and classifying template callback types in C++"
---

# Introduction

While implementing [cppkafka](https://github.com/mfontanini/cppkafka), I realized every time
I wrote a consumer application, the consumption code would look like this:

{% highlight cpp %}

while (true) {
    // Try getting a message
    Message msg = consumer.poll();

    // We may not have got anything (timeout)
    if (!msg) {
        continue;
    }

    // Check for errors
    if (msg.get_error()) {
        // EOF is propagated as an error but it really isn't one
        // Most of the times, you don't care about them
        if (!msg.is_eof()) {
            // This is an actual error, handle it some how
            // ...
        }
        continue;
    }

    // We found an actual message. Process it
    process_message(move(msg));
}

{% endhighlight %}

This started getting repetitive after a while. Most of the times, you only really care about
getting a message so the majority of that code will usually do the same, except for the one bit
that actually does the message processing.

Moreover, the rest of the "events" (_EOF_/error notification, etc) you get can easily be assigned
a default behavior. For example, when an error that's not just an _EOF_ notification is
encountered, it's safe to default to throwing an exception and let someone handle it properly,
whereas for some others just ignoring them is enough (e.g. for timeouts).

So I wondered if there could be a way to wrap consumption in such a way that I could just express
interest in the cases I wanted (e.g. actually getting a message) and not having to remember to
check for all of those conditions.

Note that this is a C++11 library and as much I'd love to use some C++14/17 features that would
simplify some of the code in here, I preferred sticking to the standard the library was aimed for.

# Using callbacks

The first thing that came to my mind was using callbacks via `std::functions`. You would be able
to configure callbacks so that whenever some event occurred, you would get a notification. For
example, something like:

{% highlight cpp %}

// Build some dispatcher instance that will consume for us using the
// underlying consumer to do all the polling related work
ConsumerDispatcher dispatcher(consumer);

// Configure the callback that will handle actual messages
dispatcher.on_message([&](Message msg) {
    // Process it
    // ... 
});

// This would start the consumption loop, dispatching every event as
// needed. In this case we would only get valid messages through 
// the lambda used above that aren't timeouts nor errors
dispatcher.run();

{% endhighlight %}

The rest of the optional callbacks (e.g. handling an error) would default to some implementation
which you can easily override. The entire list of events you could be interested in would be:

* A new message is polled.
* An error is found.
* _EOF_ is reached on a partition.
* We get no messages out of a poll. We'll call this timeout, as it's one of the cases which
will cause this.

This implementation actually works fine and could be a solution to the problem. The only potential
downside about this is that `std::functions` have some overhead as they use virtual calls to
perform the function dispatching. If I was going to add some helper class to aid in making
consumption simpler, I would want it to be as lightweight as possible so that it wouldn't make
consuming messages any slower.

# Templates to the rescue

If we changed that `ConsumerDispatcher` class to use template functor types rather than
`std::functions`, the compiler would likely be able to get rid of most of the overhead and 
end up generating code that looks as close to the initial loop as possible.

For example, we could do something like:

{% highlight cpp %}

template <typename MessageFunctor, typename ErrorFunctor> // etc
void ConsumerDispatcher::run(MessageFunctor on_message,
                             ErrorFunctor on_error) {
    // Same loop, dispatch to parameters
}

{% endhighlight %}

The issue is that in order to use templates, we would have to either:

* Require the user to provide callbacks for all types of events that can occur. This is not a
great idea as we'd be forcing the user to go back to the initial loop implementation, only now
using callbacks instead.
* Provide multiple overloads so that users can specify anywhere from one to four callbacks to 
handle each type of event. This doesn't work well as you would need to handle all combinations
of them which becomes a mess for both implementing and using the class.

Both of these don't really work for the reasons stated above. But there's another way to do this:

* Use a single function and variadic templates, detect which callback the user is trying to
specify based on its ignature and then dispatch to each of them when needed. This way, the user
can provide one to four callbacks and we would be able to find which one should handle each
case and call them accordingly.

This would require some template metaprogramming to detect signatures and sounded fun so I gave
it a try. It had been a while since I had done anything using _TMP_ so this added some extra fun
to it.

## Detecting signatures

Detecting the signature of a template functor type is simple. You can implement this by
using `decltype` and `std::declval` using the commonly used yes/no overload technique:

{% highlight cpp %}

// Indicates whether the functor type T accepts parameters Args...
template <typename T, typename... Args>
struct takes_arguments {
    using yes = double;
    using no = bool;

    // Declare a value of type Functor and try to call it using the arguments
    // types provided
    template <typename Functor>
    static yes test(decltype(declval<Functor&>()(declval<Args>()...))*);

    template <typename Functor>
    static no test(...);

    static constexpr bool value = sizeof(test<T>(nullptr)) == sizeof(yes);
};

{% endhighlight %}

Then I figured that given all the template parameter packs that I would need to use, it would be 
a good idea to implement a specialization that actually took a tuple of types:

{% highlight cpp %}

template <typename T, typename... Args>
struct takes_arguments<T, tuple<Args...>> : takes_arguments<T, Args...> {

};

{% endhighlight %}

So now using this we can check if we can call a function using some parameters. For example:

{% highlight cpp %}

void foo(int);
void bar(char, void*);

// All of these would succeed
static_assert(takes_arguments<decltype(foo), int>::value);
static_assert(takes_arguments<decltype(bar), char, void*>::value);
static_assert(takes_arguments<decltype(bar), tuple<char, void*>>::value);

{% endhighlight %}

## Finding a specific callback's type

Now that we know whether a callback can be executed using the list of types we expect,
we can write something that given a list of callbacks and a signature, would find the
type of one of those callbacks that matched that signature.

Since we may not actually find the callback we want because the user used an incompatible
signature, I used a `type_not_found` type to indicate so:

{% highlight cpp %}

// Simple identity trait
template <typename T>
struct identity {
    using type = T;
};

// Placeholder to indicate a type wasn't found
struct type_not_found {

};

// Recursive helper to iterate through all functor types until 
// we find the one we want
template <typename Tuple, typename Functor, typename... Functors>
struct find_type_helper {
    // If it accepts the arguments, then this is the type we want.
    // Otherwise keep iterating over the callback types
    using type = typename conditional<takes_arguments<Functor, Tuple>::value,
                                      identity<Functor>,
                                      find_type_helper<Tuple, Functors...>
                                     >::type::type;
};

// Base case. If we reach the end of the list, we couldn't find it
template <typename Tuple>
struct find_type_helper<Tuple, type_not_found> {
    using type = type_not_found;
};

// Find the functor that matches the given std::tuple of types.
// Note that we append type_not_found at the end so that recursion
// will always end on it in case we can't find the type.
template <typename Tuple, typename... Functors>
struct find_type {
    using type = typename find_type_helper<Tuple, Functors...,
                                           type_not_found>::type;
};

{% endhighlight %}

This can be instantiated using an `std::tuple` containing the types we expect the functor
to accept and the list of functors:

{% highlight cpp %}

void foo(int);
void bar(char, void*);

// decltype(foo)
find_type<tuple<int>, decltype(foo), decltype(bar)>::type x;
// decltype(bar)
find_type<tuple<char, void*>, decltype(foo), decltype(bar)>::type y;
// type_not_found, as nothing can be called using a string
find_type<tuple<string>, decltype(foo), decltype(bar)>::type z;

{% endhighlight %}

This gives us the ability to find a specific type within the list of functors. Now we
need a way to actually find the callback we want rather than just its type.

## Finding a callback

Finding a callback is simple if we use the `find_type` helper we wrote above. We basically
need to find the type we want out of the list of functor types (e.g. the one that matches the
signature) and then go through all functors until we find the one that matches that type. In
pseudo code, this would mean something like:

{% highlight cpp %}

// First find the type of the functor that is callable
// with the list of arguments we provide. The type returned
// would be the type of one of the functors
type = find_type(arguments, functors);

// Now find the functor that matches the given type
functor = find_functor(type, functors);

{% endhighlight %}

After actually implementing it, it looks like the following:

{% highlight cpp %}

// Helper the iterates through all functors until it finds the right one
template <typename Functor>
struct find_functor_helper {
    // Base case where we found it
    template <typename... Functors>
    static const Functor& find(const Functor& arg, Functors&&...) {
        return arg;
    }

    // Recursive case, only instantiated if the head of the template
    // parameter pack is not the type we want
    template <typename Head, typename... Functors>
    static typename enable_if<!is_same<Head, Functor>::value,
                              const Functor&>::type
    find(const Head&, const Functors&... functors) {
        return find(functors...);
    }
};

// The function that actually finds it just instantiates and calls
// the helper
template <typename Functor, typename... Args>
const Functor& find_functor(const Args&... args) {
    return find_functor_helper<Functor>::find(args...);
}

{% endhighlight %}

Note that this will fail if we provide a type that can't actually be matched. However, this
cant't happen, as we are going to extract the actual type out of the functor list using
`find_type` first so we are guaranteed that the type we are looking for is actually in the
functor parameter pack. 

Now as a last helper, I wrote this function that finds the type and if it can't find it,
it fails with a static assertion:

{% highlight cpp %}

// Finds the callback and returns it or fails with a static assertion
// if lookup fails
template <typename Tuple, typename... Functors>
const typename find_type<Tuple, Functors...>::type&
find_matching_functor(const Functors&... functors) {
    // Find the specific type
    using type = typename find_type<Tuple, Functors...>::type;

    // Fail if we couldn't find it
    static_assert(!is_same<type_not_found, type>::value,
                  "Valid functor not found");

    // Now find the functor that matches the type
    return find_functor<type>(functors...);
}

{% endhighlight %}

This function is then used as follows:

{% highlight cpp %}

template <typename... Args>
void ConsumerDispatcher::run(const Args&... args) {
    // Define the types we expect on our message processing callback
    using OnMessageArgs = tuple<Message>;

    // Will fail with a static assertion if we can't find it
    auto callback = find_matching_functor<OnMessageArgs>(args...);

    // Iterate, poll for messages, error events, etc
}
{% endhighlight %}

## Optional callbacks

But what about optional callbacks? Initially we wanted to allow the user to specify some callbacks
but not necessarily all of them as this is kind of tedious. Right now if we call
`find_matching_functor` for an optional one, the static assertion will fail, so we need
a way around that. One way could be to implement some "find or fallback" mechanism so that
we provide our own default in case we can't find it. 

However, there's a trivial way around this without any code additions. We can just append our
own custom handler at the end of the functor list. If we find the user's first, then we'll use
that one. Otherwise we fall back to using ours:

{% highlight cpp %}

static void ConsumerDispatcher::handle_error(Error err) {
    // Throw!
}

template <typename... Args>
void ConsumerDispatcher::run(const Args&... args) {
    // Define the types we expect on our message processing callback
    using OnErrorArgs = tuple<Error>;

    // Our default handler in case the user doesn't specify one
    auto default_handler = &ConsumerDispatcher::handle_error;

    // Find the callback, falling back to our default as a last resort
    auto callback = find_matching_functor<OnErrorArgs>(args...,
                                                       default_handler);
}

{% endhighlight %}

## Detecting invalid signatures on callbacks

This actually happened to me while writing tests. I wrote a test for this which worked fine and
then I changed the required signature for two of the events and my tests failed on runtime. My code
didn't detect if the user provided an invalid signature so it would silently ignore it and use
its own default instead.

This can be easily solved making sure every provided callback matches one of the signatures:

{% highlight cpp %}

// Check that a given functor matches at least one of the
// expected signatures
template <typename Functor>
void check_callback_matches(const Functor& functor) {
    static_assert(
        !is_same<type_not_found,
                 typename find_type<OnMessageArgs, Functor>::type>::value ||
        !is_same<type_not_found,
                 typename find_type<OnEofArgs, Functor>::type>::value ||
        !is_same<type_not_found,
                 typename find_type<OnTimeoutArgs, Functor>::type>::value ||
        !is_same<type_not_found,
                 typename find_type<OnErrorArgs, Functor>::type>::value,
        "Callback doesn't match any of the expected signatures"
    );
}

// Base case for recursion
void check_callbacks_match() {
}

// Check that all given functors match at least one of the
// expected signatures
template <typename Functor, typename... Functors>
void check_callbacks_match(const Functor& functor,
                           const Functors&... functors) {
    // Check this one
    check_callback_matches(functor);

    // Then check the rest
    check_callbacks_match(functors...);
}

{% endhighlight %}

## Putting it all together

Now, after putting all of this together, we can just provide our callbacks and have
the class detect which tries to handle what. As a result, we can use this dispatcher class
in the following way:

{% highlight cpp %}

// Construct the dispatcher
ConsumerDispatcher dispatcher(consumer);

// Run and dispatch
dispatcher.run(
    [&](Message msg) {
        // Do something with this message
    },
    [&](Error error) {
        // Handle the error
    }
);

{% endhighlight %}

Internally this consumes a message and performs all the checks shown in the loop at the beginning
of this post, dispatching to each callback when needed.

# Conclusion

All of this code was used to create the
[ConsumerDispatcher](https://github.com/mfontanini/cppkafka/blob/master/include/cppkafka/utils/consumer_dispatcher.h)
class in _cppkafka_.

I think this is a great improvement over having to write the same loop over an over again. Using 
callbacks allows easily spotting what happens on each case and let's you only express interest
in the specific type of events you want rather than having to handle all of them.

One could argue that this reduces readability, as now you need to rely on the documentation to 
be sure what is the signature you're supposed to use. Moreover, compilation errors will now be a
bit less specific as they will be a static assertion claiming that no valid functor could be
found but knowing which is the one that's wrong is slightly trickier. Still, I'm happy with the
way this works and I'll surely be moving my applications to use it.
