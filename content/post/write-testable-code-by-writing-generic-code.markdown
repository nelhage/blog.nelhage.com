---
title: "Write testable code by writing generic code"
slug: write-testable-code-by-writing-generic-code
date: 2020-03-11T18:30:17-07:00
---
[Alex Gaynor](https://alexgaynor.net/) recently asked this question in an IRC channel I hang out in (a channel which contains several software engineers nearly as obsessed with software testing as I am):


> uhh, so I'm writing some code to handle an econnreset... how do I test this?

This is a good question! Testing `ECONNRESET` is one of those fiddly problems that exists at the interface between systems — in his case, with S3, not even a system under his control — that can be infuriatingly tricky to reproduce and test. Furthermore, it turned out that his existing tests for the system in question didn’t even hit the network, but mocked out S3 using an in-memory [`io.ReadCloser`](https://golang.org/pkg/io/#ReadCloser). This made injecting an `ECONNRESET` and testing the handling even trickier.

Another IRC member dug up some code from his own codebase where they checked for `ECONNRESET` and tested the handling. It turned out to be a bit flaky, involved multiple goroutines, a `time.Sleep` in the test, and string-comparison on error messages. But I couldn’t come up with a much more satisfying answer, either.

However, after thinking about the problem a bit, I started to wonder *why* Alex needed to test for `ECONNRESET`. Was there a simpler solution that might both solve his problem, and be easier to test?

I asked some more questions, and got the following set of facts about his situation:

- He’s working on an application that streams large immutable files out of S3.
- Occasionally, mid-way through a file, the connection to S3 drops, and he receives a `ECONNRESET`.
- His system is capable of restarting the job from the top of the file, but as an optimization he wants to handle the errors more locally and restart from where he left off.
- Since S3 supports range requests, it’s conceptually easy for him to track the byte offset he’s read so far, and restart the read at that offset, transparently to the reader.

I agreed this problem statement seemed sensible. Talking about it some more, we decided that problem statement could be abstracted from `ECONNRESET` entirely. He was focusing on retrying that error because it was the one he’d seen, but we couldn’t imagine another `Read` error that couldn’t be retried in the same way. Now his problem was slightly simpler, since it was reduced to “handling errors in `Read`" instead of handling a specific fiddly network error.

The problem, though, could be generalized even further! What Alex had, conceptually, was a way to create an `io.ReadCloser` starting at an arbitrary offset (using an S3 range request), and he wanted to wrap it into an `io.ReadCloser` that handled errors by retrying from the current offset (at least up to some limit). So his problem could be reduced, in large part, to implementing and testing a function like the following:


    // NewRetryingReader accepts a function which can create an `io.ReadCloser`
    // reading from some backend at the specified offset. For instance, this could
    // be implemented for a filesystem by combining `os.Open` and `os.File.Seek`. It
    // returns an `io.ReadCloser` which will read the entire backend sequentially
    // `from` offset 0, retrying any `Read` errors by restarting from the last-read
    // offset.
    //
    // The returned reader will retry at a given offset MaxRetriesAtOffset times. If it
    // receives that many failures in a row, it will return the last error received
    // and stop retrying.
    func NewRetryingReader(startReading func(offset int64) (io.ReadCloser, error)) io.ReadCloser

By abstracting out the details of S3, we’ve created a simple abstraction which can be implemented and tested in isolation of any application code. It’s very easy to test by creating in-memory implementations of `startReading` which back to a byte buffer and return errors periodically, or even fuzzed by randomizing the buffer sizes, read sizes, and failure rate.

What’s more, it can be quite plausibly be reused for any similar case; it’s easy to imagine it being useful for any network block storage which supports range requests, or wrapping it around an HTTP client.

If he want down this road, Alex would still need to implement the `startReading` function for his S3 backend, but since he already has code to create an `io.ReadCloser` from an S3 path, it would likely just be a question of setting the `Range` field on his [`GetObjectInput`](https://pkg.go.dev/github.com/aws/aws-sdk-go/service/s3?tab=doc#GetObjectInput) request. Testing this would be not much easier or harder than testing his existing S3 client.

# The case for generic code

I’ve found this to be a recurring result: making code more generic can sometimes make it simpler to test.

The specifics of your application and its interactions with dependencies are often quite fiddly, and replicating them faithfully in a test environment is often complex, slow, and/or unreliable. If you can abstract out the behavior you want to test from the specifics of the concrete components you are dealing with, it sometimes becomes both easier to describe, and easier to test.

One can view this as a form of [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection); instead of hardcoding S3 into the application, we’ve injected the concrete details of S3 as a parameter, allowing the code to be tested independently of S3. I think one lesson we can take away is that, when you’re considering reaching for dependency injection to test a component (either using some framework, or just via “passing arguments to functions”), it’s worth at least exploring whether your problem is a special case of a nearby abstraction which you could write instead, which might make the dependency injection more natural.

# Caveats

That said, I want to include some strong cautions about this approach!

Implementing an abstraction like the above will very likely entail more total code than the spot fix to the specific problem Alex had. More perniciously, by creating a general-purpose abstraction, other developers in his code base are likely to use it, which will stress-test it more and expose it to more use cases and edge cases than Alex might otherwise encounter in his specific case. The abstraction therefore may grow to be substantially more complex than the simple patch would entail, and might create ongoing support burden. Hopefully it will be generating value for the organization commensurate to this burden, but it will be Alex or his team taking on that support burden, which may not be desirable for them.

For this reason, even though I think this example is a case where the proposed abstraction is quite sensible and fairly simple to implement, I would think twice in Alex’s position before actually introducing it into my code base as described. It would likely have advantages in the long run, but in the short run it might take the rest of the day to write and document and test to my satisfaction, and it might be more expedient to just do a simpler, less-well-tested fix, and go back to doing actual product work.

Additionally, even if the abstraction itself is easy to test, by introduction an abstraction at all you’ve added new **integration** points between the abstraction and the application code. Even if the abstraction behaves perfectly and is well-tested on its own terms, those integration points are opportunities for bugs to sneak in and for your application code to behave differently from the test environment.

In this case, I feel quite good about the abstraction I propose above; it’s small and simple, and the types and integration points map very smoothly and straightforwardly to the code you might write anyways.

However, I’ve seen enthusiastic developers try to follow this approach and produce abstractions which require implementing a 13-method interface with subtle and unclear interactions between all of its methods in order to use the “abstraction.” In those cases, I find these components almost invariably add net complexity to the system, making it harder to test and understand. And the complexity (and, typically, poor documentation and enforcement) of the interface between the abstraction and the application makes it hard to have confidence that you’re using the abstraction right, or that the result of tests carry over to application code.


# Testing as a driver for good architecture

In this instance, I think the proposed abstraction above, or something similar, is probably actually an objectively desirable architecture for Alex’s problem, in addition to being easy to test. It keeps the “business logic” reading the files completely unchanged, with no new error-handling or awareness of the details of the object store. It isolates the S3 connection and retry logic into one place, close together and fairly generically — it’s easy to imagine slotting it in front of all uses of S3 in his application.

If I were instead pursing a “quick fix” to this problem, I might be tempted to inject *ad hoc* error handling and retries into whatever application code is calling `Read` on the `ReadCloser`, infecting the application code with additional error-handling and potentially also knowledge of the specific S3 backend. From an eye to long-term maintainability and sensible layering, I think that wrapping the retries entirely inside the `ReadCloser` makes a lot of sense. I view this case, therefore, as an instance where [designing for testability](https://blog.nelhage.com/2016/03/design-for-testability/) actually ends up leading to better code overall. We started out with a simple question of “I wrote this code, how do I test it?” and by pulling on the thread and by bringing the question of “Can we redesign this?” into scope, we ended up both with better tests, and with better system design overall.

