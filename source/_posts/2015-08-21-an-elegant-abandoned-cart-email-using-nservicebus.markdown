---
layout: post
title: "An Elegant Abandoned Cart Email Using NServiceBus"
date: 2015-08-21 20:30:59 -0400
comments: true
categories:
---

## Abstract

Some of the toughest requirements to model in enterprise development are business rules that involve the dimension of time.  In this post I'll show you how to fulfill those types of requirements in an elegant, testable, and scalable way using a message-driven style with [NServiceBus](http://particular.net/nservicebus).

## tl;dr

If you're familiar NServiceBus and specifically its [sagas feature](docs.particular.net/nservicebus/sagas/), you can check out [the code on GitHub](https://github.com/jkodroff/nsb-abandoned-cart).  Of particular interest would be [the saga](https://github.com/jkodroff/nsb-abandoned-cart/blob/master/src/Server/Saga.cs) and [its tests](https://github.com/jkodroff/nsb-abandoned-cart/blob/master/src/Server.Tests/SagaTests.cs).  There's also a [console app client](https://github.com/jkodroff/nsb-abandoned-cart/blob/master/src/Client/Program.cs) that will let you play around.

<!-- more -->

## A Common Requirement and a Common Implementation

A common requirement in eCommerce systems might go like this:

> If a customer adds an item to their cart and does not complete the checkout process within `$SOME_AMOUNT_OF_TIME`, send them an email reminding them that they have items in their cart.

Many systems will solve this problem using a batch job.  It'll go something like this:

```c#
var abandonedCartThreshold = TimeSpan.FromDays(1);

foreach (cart in allCarts) { // a cart disappears when it becomes an order
    if (cart.Items.Any(x => DateTime.UtcNow - x.CreatedOn <= abandonedCartThreshold) {
        // fetch customer to whom the cart belongs
        // send email
    }
}
```

This might be fine for a simple system with low numbers of customers, but for most real-world systems there's several drawbacks with this approach that we need to consider:

1. **It's slow, especially if you're sending the email synchronously:** Network operations like sending email are the slowest types of operations.  And if our business is doing well, has tons of customers, and the nightly job's time exceeds 24 hours, we're in big trouble because it will never complete.  A single process cannot handle the load, so we're going to have to get into multiple processes with batching.  That's going to get ugly.
2. **We have to keep track of what's already been sent:** Customers _really_ don't like getting duplicate emails. If we have an error in the middle of the loop, we will have to keep track of which emails have already been sent in order to avoid annoyed customers. 
3. **It doesn't deal with network partitions:** We have to handle retrying the job in the event that something is seriously wrong, e.g. the mail server is unavailable.  In many systems this will involve an administrator or developer manually running the process again.  That's not the phone call you want to receive that wakes you up.
4. **It's not as precise as it could be:** We're sending the email every night.  While an abandoned cart email probably isn't critical to be sent _exactly_ `$SOME_AMOUNT_OF_TIME` after the last item is added to the cart, there are other communications that may be, e.g. if payment could not be captured on a large order, so learning to be precise here has applications in other areas.
5. **The code doesn't adapt well for new requirements:** If we want to, e.g send several follow-up emails after 7 and 30 days, we're going to need to add yet some more fields to the DB for every customer and keep track of that state as well.
6. **It doesn't deal well with problems like invalid data:** If we make a mistake in our error handling or our logic (e.g. if we run into an unexpected bug with 1 specific customer without a valid email address, _who was undoubtedly imported directly into the database, skipping our painstakingly crafted validation in our application code_ (because that's what happens out there in the real world), our process will die.  Ok, so we can add try/catch block.  And we need to add tests for that.  And if you are inclinded towards software craftsmanship, it should strike you as wrong that we're going to retry the same broken customer every time we run the job, so now we need to add even more state via a flag to make sure we don't send the email to a customer who has failed `$SOME_NUMBER_OF_TIMES`.  That's a lot of additional logic we need to account for.
7. **It's difficult to test:**  How can we tell how this thing going to perform in production?  How can we be confident that it works correctly?  The state that the job encompasses includes all carts in the system, so it's difficult to say short of creating 10,000 carts and customers, then sending 10,000 emails.  That can run into issues like getting your employer blacklisted for bad email behavior.   Of course, we can use things like [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) for testing, but because we're not sending a real email, we won't get anything like production performance.

Those are pretty significant issues, so what are we going to do?

## Enter The Saga

Luckily, there's a better way to do this: The saga pattern is the right tool for this job. It's tricky to describe a saga conscisely, so here's a list of things _about sagas_ that might help you conceptualize them:

1. **They're state machines:** You can think of them like a flow chart that represents a slow-running business process.  Where "slow-running" is defined as "at least computer time slow, and maybe even human-time slow", e.g. an ordering process that requires multiple calls to HTTP-based APIs or... an abandoned cart email that needs to be sent 1 day after the last item is added to the cart (fancy that!).
2. **They're coordinators:** They wait for things to happen (e.g. for the warehouse to tell us that an order has shipped), and then they tell the system to do things (e.g. charge the customers card, send the customer a email telling them their order has shipped).
3. **They're persistent entities:** Because the processes they model are slow-running, they have to persist the state of the process after each step is done.  We can't expect to keep this process running in memory for 3 days while a placed order is packaged at a warehouse.  Because they're loaded by a unique identifier they are [entities](http://martinfowler.com/bliki/EvansClassification.html) domain-driven design terms.  (They're also [aggregate roots](http://stackoverflow.com/questions/1958621/whats-an-aggregate-root), although I think this distinction can be confusing so feel free to ignore it.)  


We also get the following benefits with NServiceBus' implementation of sagas:

1. **Retrying failed messages:** Sagas in NServiceBus are also message handlers and NServiceBus message handlers [automatically retry failed messages](http://docs.particular.net/nservicebus/errors/automatic-retries).  This feature is very very useful when dealing with, e.g [a temporary outage on our mail service](https://twitter.com/search?q=mailgun%20%5Bstatus%5D&src=typd).
2. **Persisting state between messages:** The NServiceBus `Saga` class (from which we inherit) is already designed to handle persisting the state after each message in the saga is handled.  More code we don't have to write.
3. **The dimension of time can easily be modeled and tested:** NServiceBus sagas allow us to express a process over time through its [timeouts feature](http://docs.particular.net/nservicebus/sagas/timeouts).  This feature is going to be key to implementing our requirement as you'll see very soon.
4. **Our saga will run asynchronously:** NServiceBus message handlers (which include sagas) run asynchronously, which will allow us to [scale out](http://docs.particular.net/samples/scaleout/) by running multiple copies of our saga endpoint if we need to.

## Creating our NServiceBus Saga: First Steps

**Note:** This section assumes you have a basic understanding of NServiceBus' message types.  If you don't you should [read this](http://docs.particular.net/nservicebus/messaging/messages-events-commands).

The first thing we need to do is to create some messages which will be sent by our client application via NServiceBus' `Bus.Send()` method.  This is probably an ASP.NET MVC application, but they could just as easily be sent from a console application, or anywhere really.  If we're following the typical messaging semantics these should be events, but because there's a high likelihood they are sent from a website, we should not use `Bus.Publish()` ([detailed explanation here](http://www.make-awesome.com/2010/10/why-not-publish-nservicebus-messages-from-a-web-application/)).

Here's our messages:
```c#
// When we get one of these, we will reset the countdown to sending the email:
public class ItemAddedToCart : IMessage
{
    public string UserName { get; set; }
}

// When we get one of these, we will cancel the entire 
// process because the the cart is no longer abandoned:
public class OrderSubmitted : IMessage
{
    public string UserName { get; set; }
}
```

Now we're ready to create our saga.  When we create a saga in NServiceBus, we need to define 2 things:

1. The class which will hold the saga's state in between messages.  This is where we'll keep track of which messages were received and any state that we need to keep track of between messages.
2. The message that begins the saga.

Let's do that now:

```c#
public class AbandonedCartSagaData : IContainSagaData
{
    // Built-in saga properties:
    public Guid Id { get; set; }
    public string Originator { get; set; }
    public string OriginalMessageId { get; set; }
}

public class AbandonedCartSaga : 
  Saga<AbandonedCartSagaData>,
  IAmStartedByMessages<ItemAddedToCart>
{
    public void Handle(ItemAddedToCart message) { }
}
```

We need to tell NServiceBus how to get from an incoming message to the right saga (one saga per customer, keyed off the username).  

Add this to `AbandonedCartSagaData`:

```c#
// This needs to be unique so that we can scale out, or 
// we might get 2 Sagas with the same username, 
// per https://twitter.com/UdiDahan/status/587896128688951297
[Unique]
public string UserName { get; set; }
```

And add this to `AbandonedCartSaga`:

```c#
protected override void ConfigureHowToFindSaga(SagaPropertyMapper<AbandonedCartSagaData> mapper)
{
    mapper
        .ConfigureMapping<ItemAddedToCart>(x => x.UserName)
        .ToSaga(x => x.UserName);

    mapper
        .ConfigureMapping<OrderSubmitted>(x => x.UserName)
        .ToSaga(x => x.UserName);
}
```


## Creating our NServiceBus Saga: Business Logic

First we need to understand [timeouts in NServiceBus sagas](http://docs.particular.net/nservicebus/sagas/timeouts).  A timeout message is like a programmable boomerang: the saga sends the timeout message (which can contain whatever additional data you want), and it'll "come back" to the saga in whatever `TimeSpan` you specify when you call `RequestTimeout()` from within the saga.  

We're going to model our business process in our saga like this:

1.  Every time we receive `ItemAddedToCart`, we'll send a timeout message `AbandonedCartTimeOut` with a newly generated unique identifier.  We'll also store this ID in the saga's data.
2.  When we receive `AbandonedCartTimeOut`, if the ID of the incoming message does not match ID of the last sent message in the saga, we'll know that an item has been added to the cart since the timeout message was issued.  If the IDs match, we'll send the email and complete the saga.
4.  If we receive an `OrderSubmitted` message, we don't need to send the email because the customer has completed the checkout process and we'll complete the saga.

Let's do that now.  First, we'll need to create our timeout message.  

```c#
public class AbandonedCartTimeOut : IMessage
{
  public Guid Id { get; set; }
}
```

Add this to `AbandonedCartEmailSagaData`:

```c#
public Guid LastTimeOutId { get; set; }
```

Add this to `AbandonedCartEmailSaga`:

```c#
public void Handle(ItemAddedToCart message)
{
    Data.UserName = message.UserName;
    Data.LastTimeOutId = Guid.NewGuid();


    RequestTimeout(
        // We're using 5 seconds because it makes it easy to test,
        // but in a real system, we might want to provide this via
        // a configurable variable.
        TimeSpan.FromSeconds(5), 
        new AbandonedCartTimeout {
            Id = Data.LastTimeOutId
        }
    );
}

public void Handle(OrderSubmitted message)
{
    MarkAsComplete();
}

public void Timeout(AbandonedCartTimeout state)
{
    if (Data.LastTimeOutId != state.Id) {
        // This is not the last timeout issued, so ignore it.
        return;
      }

    Bus.SendLocal(new SendAbandonedCartEmail {
        UserName = Data.UserName
    });

    MarkAsComplete();
}
```

## How to Test Our NServiceBus Saga

There's 2 philosophies when it comes to testing NServiceBus sagas:

1. **Unit Testing:** We test each message handler in the saga in isolation, setting up the saga's internal state (i.e. `AbandonedCartSagaData` in our case) for each message, possibly with several tests for each message handler (if the handler can exhibit several different behaviors).
2. **End-to-End Testing:** We test the entire flow of a saga from start to completion.  

I think the latter approach provides some significant advantages:

1. **It more closely resembles real-world usage:** Our saga exists in the wild as a thing that handles a distinct series of messages, with its state managed internally, so it's better to test it that way.
2. **It requires less code:** NServiceBus' testing facilities provide a convenient way to do this type of testing via [method chaining](https://en.wikipedia.org/wiki/Method_chaining).

## Actually Testing Our NServiceBus Saga

We have 2 distinct scenarios.  First, let's do the "happy path" (from the view of an eCommerce retailer) where the order gets placed.  (The assertion methods pictured below are done with [FluentAssertions](http://www.fluentassertions.com/):

```c#
[Test]
public void OrderSubmitted()
{
    var userName = "sales@joshkodroff.com";
    var timeoutId1 = Guid.Empty;

    Test
        .Saga<AbandonedCartSaga>()
        .ExpectTimeoutToBeSetIn<AbandonedCartTimeout>((msg, span) => {
            timeoutId1 = msg.Id;
            span.Should().Be(TimeSpan.FromSeconds(5));
        })
        .When(saga => saga.Handle(new ItemAddedToCart {
            UserName = userName
        }))
        .ExpectTimeoutToBeSetIn<AbandonedCartTimeout>((msg, span) => {
            // just to make sure we're generating new ids
            msg.Id.Should().NotBe(timeoutId1); 
            
            span.Should().Be(TimeSpan.FromSeconds(5));
        })
        .When(saga => saga.Handle(new ItemAddedToCart {
            UserName = userName
        }))
        // just a dummy check.  no message should be sent
        .ExpectNotSend<SendAbandonedCartEmail>(x => x != null) 
        .When(saga => {
            // the first timeout comes back (which should be ignored):
            saga.Timeout(new AbandonedCartTimeout {
                Id = timeoutId1
            });

            saga.Handle(new OrderSubmitted {
                UserName = userName
            });
        })
        .AssertSagaCompletionIs(true);
}
```

And now we have the "less happy" path where the followup email needs to be sent:

```c#
[Test]
public void OrderNotSubmitted()
{
    var userName = "sales@joshkodroff.com";
    var timeoutId1 = Guid.Empty;
    var timeoutId2 = Guid.Empty;

    Test
        .Saga<AbandonedCartSaga>()
        .ExpectTimeoutToBeSetIn<AbandonedCartTimeout>((msg, span) => {
            timeoutId1 = msg.Id;
            span.Should().Be(TimeSpan.FromSeconds(5));
        })
        .When(saga => saga.Handle(new ItemAddedToCart {
            UserName = userName
        }))
        .ExpectTimeoutToBeSetIn<AbandonedCartTimeout>((msg, span) => {
            timeoutId2 = msg.Id;
            span.Should().Be(TimeSpan.FromSeconds(5));
        })
        .When(saga => {
            saga.Handle(new ItemAddedToCart {
                UserName = userName
            });

            // the first timeout comes back (which should be ignored):
            saga.Timeout(new AbandonedCartTimeout {
                Id = timeoutId1
            });
        })
        .ExpectSendLocal<SendAbandonedCartEmail>(msg => {
            msg.UserName.Should().Be(userName);
        })
        .When(saga => saga.Timeout(new AbandonedCartTimeout {
            Id = timeoutId2
        }))
        .AssertSagaCompletionIs(true);
}
```

## Possible Future Enhancements and Conclusion

What if we need to account for a new `ItemRemovedFromCart` message?  We'd need to include the total number of items in the cart with the `ItemRemovedFromCart` message, and after that it's pretty simple: If `ItemRemovedFromCart.NumberOfCartItems` is zero, complete the saga because there's no items left in the cart.  If it's non-zero, then the handler has the same logic as `ItemAddedToCart`.  For testing, we'll need to add in a few of these messages to our happy path, then add in an additional test fixture or two that aseserts that the saga completes when there's no items left in the cart.)

What if we want to send additional follow-up emails after 7 and 30 days?  Well that would be pretty simple to add:  Just add timeouts for 7 and 30 days and change when the saga completes (after the 3rd email is sent instead of a single email).  It should also be pretty easy to test: extend our happy path test with a couple more assertions for the additional emails, and add 2 more less-happy tests for the order being placed before those timeouts happen.

We can see from the above examples that the saga is very well-suited to adapt to future changes in requirements.  I've found NServiceBus sagas to be a fantastically-designed tool to handle some of the most difficult requirements I've come across.  They take a minute to grok beacause they're so powerful, but once you have a good handle on their capabilities and get comfortable with how to fit them into your projects you can really, really get things done.

Happy coding!
