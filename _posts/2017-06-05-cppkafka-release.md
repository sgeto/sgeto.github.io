---
layout: post
title:  "cppkafka - C++11 librdkafka wrapper"
---

[cppkafka](https://github.com/mfontanini/cppkafka) is a project I've been slowly working on for
a while. It's a _C++11_ wrapper built on top of [librdkafka](https://github.com/edenhill/librdkafka),
a high performance _C_ client library for the [Apache Kafka](https://kafka.apache.org/) protocol.

# Why another Kafka library?

A few months ago, we started using Kafka at my [current workplace](https://www.thousandeyes.com/).
I had to port some applications and implement new ones that would communicate with each other
using this protocol. The existing applications where implemented in _C++_ so re-implementing them
from scratch in another language was not an option. Plus, I'm very comfortable using it, so there
really was no reason to change it.

After looking for some alternatives on what to use, I had the impression that most of the _C++_
libraries where kind of immature and not really production ready. I did find
[librdkafka](https://github.com/edenhill/librdkafka), developed in _C_, which looked very
promising and seemed to be in active development, providing a robust and high performance
implementation of the protocol. In fact, this is the only _C/C++_ library that has full protocol
support for Kafka. _librdkafka_ contains several different language bindings like _Python_, _Go_,
_C#_, etc. Even better, there seemed to be an official _C++_ wrapper for the library, _rdkafka++_,
which should make using the library a pleasant experience.

## This doesn't really work

<p style="text-align: center;" markdown="1">
![So many pointers]({{ site.url }}/assets/no-pointers.jpg)
</p>

My first reaction was not great. Don't get me wrong, the library itself worked without any issues
and I was able to write a Kafka producer and consumer in a decent amount of time (mostly due to
Kafka's learning curve), but overall using _rdkafka++_ was slightly tedious. 

As much as I don't have issues using raw pointers and I've used them, as well as naked _news_ and
_deletes_ all over the place, that was a while ago. Nowadays I refuse to use a pointer unless it's
wrapped around some smart pointer so I can guarantee proper a cleanup of all resources even on a
failure condition. _rdkafka++_ uses raw pointers for absolutely everything, which was really not
what I wanted.

_C++_ has evolved a lot and there's really no need to perform manual memory management anymore.
There's [zero cost abstractions](http://en.cppreference.com/w/cpp/memory/unique_ptr) in the 
standard library that let you write safe code with no overhead at all.

## Wrapping the wrapper

At that point I decided to create a tiny project on our company's repository that would act as
a wrapper of _rdkafka++_. So basically this would be a wrapper of a wrapper of _librdkafka_. At the
time this seemed like a good idea as I would only need to write a tiny bit of code and then I
would be able to create applications that used Kafka in a simple way. This would allow me to
create _rdkafka++_ configuration objects, consuming and producing messages without having to
deal with any pointers.

This didn't go so well, as I soon found myself having to deal with some pretty nasty things.
_rdkafka++_ tries its best to hide away its internal implementation which is not necessarily bad,
as it allows for faster compilation times and it makes it harder to break the ABI when introducing
changes, but in this case it just made it trickier to work with. I had to end up doing things like
cloning `RdKafka::Conf` objects manually by accessing their internal representation, dealing with
implementing proxy callbacks, wrapping all pointers in `unique_ptrs`, etc.

After a while of using this wrapper and getting slightly frustrated, I realized I could do better
and chose to actually implement a wrapper for the _C_ library, _librdkafka_, that used modern
_C++_ features like smart pointers, move semantics and callbacks to provide a clean interface to
use Kafka.

# Hello [_cppkafka_](https://github.com/mfontanini/cppkafka)

I slowly implemented this wrapper and used it at work on the projects I was porting to use Kafka
as well as the new applications I implemented. All of those served as my guinea pigs, as I would
use _cppkafka_, get annoyed at something, work on it at home and then change the projects to use
the updated API. After a few months of changing things, I think at this point I'm satisfied with
the current state of the project and it's time for a release.

_cppkafka_ provides a wrapper for the high level consumer and producer interface, as well as 
some random utilities like a buffered producer (which also simplifies producer error handling)
and a compacted topic consumer, which lets you process key-value streams (like
[KTables](http://docs.confluent.io/current/streams/concepts.html#ktable)) in a simple way.

The library is licensed under BSD-2 so you can use it on both commercial and non commercial 
projects.

## Simple interface

_cppkafka_ lets you create Kafka producers/consumers with very little code. Creating a
consumer/producer's configuration is as simple as it gets:

{% highlight cpp %}
Configuration config = {
    // The list of brokers we'll use
    { "metadata.broker.list", "kafka-server1,kafka-server2" },

    // Our consumer's group
    { "group.id", "kafka-consumer-test" },

    // Disable auto commit
    { "enable.auto.commit", false },

    // Client group session timeout
    { "session.timeout.ms", 60000 }
};
{% endhighlight %}

There's no need to convert configuration values into strings, allocating anything, checking 
error codes, etc. You can actually use string, integral and bool types and _cppkafka_ will
perform the conversion automatically for you. Under the hood, this is actually creating a
`rd_kafka_conf_t` handle and setting the attributes on it. If setting any attributes
fail, this will throw an exception.

### Consuming messages

Consuming messages is straightforward using the _Consumer_ class.

{% highlight cpp %}
// Create some consumer configuration
Configuration config = ...;

// Create our consumer
Consumer consumer(config);

// Subscribe to the topic we want
consumer.subscribe({ "some_topic" });

// Poll forever
while (true) {
    Message msg = consumer.poll();
    // Poll can return nothing, so make sure to check we got something
    if (!msg) {
        continue;
    }
    // There's an error
    if (msg.get_error()) {
        // rdkafka indicates EOF (end of partition) as errors,
        // which doesn't really mean something went wrong
        if (!msg.is_eof()) {
            // This is an actual error, handle it properly
            handle_error(msg.get_error());
        }
        // Regardless, we need to keep going
        continue;
    }
    // We actually received a message, process it
    process_message(move(msg));
}
{% endhighlight %}

The `Message` class is just a wrapper over an `rd_kafka_message_t` handle. This means a message
can be empty in case the wrapped handle is null. `Messages` can't be copied but can be moved
so they can be safely stored in containers for later use.

You can read more about message consumption on the
[wiki](https://github.com/mfontanini/cppkafka/wiki/Consuming-messages), where there's examples
regarding partition assignment/revocation callbacks.

## Producing messages

Producing messages can be done using the `Producer` class. Messages are built using a 
`MessageBuilder` that allows setting the topic, partition, key, payload, etc:

{% highlight cpp %}
// Create some producer config
Configuration config = ...;

// Create the producer
Producer producer(config);

// The key and value we'll use
const string key = "some_key";
const vector<uint8_t> payload = {
    0x01, 0x02, 0x03, 0x04  
};

// Construct a builder for the Kafka topic "some_topic"
MessageBuilder builder("some_topic");

// Set the partition, key and payload. These can be easily chained.
builder.partition(10).key(key).payload(payload);

// Now produce the message
producer.produce(builder);
{% endhighlight %}

There's also an implementation of a [buffered producer](https://github.com/mfontanini/cppkafka/blob/master/include/cppkafka/utils/buffered_producer.h)
that will handle delivery reports for messages and re-send them if needed, as well as 
other non-errors in _librdkafka_ like filling the output queue, in which case it just waits
until messages can be written.

## Performance overhead

One of the things I kept in mind while implementing _cppkafka_ was performance. I didn't want
to create a wrapper which would add overhead to every operation, making it less performant than
_librdkafka_. Fortunately, most abstractions should have ended up being 100% free. For example
`Message` objects simply wrap a handle. If you fetch a message's key and payload, you'll get a
read only view of them that is actually not copying or allocating any extra data. Plus, it will
automatically release all resources when it goes out of scope, so you can store it on vectors or
other data structures and be sure you won't have any memory leaks.

I know there should be some benchmark in this section, as otherwise there's not much credibility.
Unfortunately, I don't have any yet so I can't really claim consuming/producing messages is 
as fast as when using _librdkafka_, but I'll definitely work on that at some point.

Some configuration properties use `std::function` for callbacks, which have a slight performance
penalty but that should be negligible compared to the cost of reading messages over the network.

## Conclusion

All in all, I've had a really positive experience using _librdkafka_ (through _cppkafka_), besides
a few issues I came across along the way, all of which were fixed in recent versions.

If you want to have a high performant and hopefully painless experience consuming and producing
messages using Kafka, make sure to check out _cppkafka_'s
[GitHub repository](https://github.com/mfontanini/cppkafka).

If you find any problems or have any suggestions, please don't hesitate to create an issue or pull
request!
