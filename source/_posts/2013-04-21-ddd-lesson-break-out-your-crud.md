---
title: 'DDD Lesson &#8211; Break Out Your CRUD'
author: jkodroff@gmail.com
layout: post
permalink: /2013/04/21/ddd-lesson-break-out-your-crud/
categories:
  - Domain-driven Design
---
In my view, there&#8217;s 2 main principles that distinguish Domain-driven design as an approach to solving problems:

  1. All solutions to problems are framed in terms of the value delivered to the stakeholders for the effort required.</p> 
  2. The core of the code is a model of the business process, and it [evolves with time][1] as the  developers&#8217; understanding of the business with time.

There&#8217;s other principles involved, but these two are the most relevant fo this particular post.

Sounds pretty generic, right?  So how do we implement this?  You may find, as I did, that direct advice (i.e. rules of thumb) on how to write domain-driven code is tough to find.  DDD is a very powerful technique for software development, but even though it&#8217;s has been popular for about 10 years now best practices are not yet widely understood.  There&#8217;s a long line of developers burned by DDD because they didn&#8217;t have a proper understanding of where domain models are appropriate.  They end up writing more code than they need to, and DDD gets a bad rap.  That&#8217;s a bummer because when it&#8217;s applied with wisdom, DDD really can make one&#8217;s life as a coder easier.

Let&#8217;s say we have the concept of an order in our domain. For simplicity&#8217;s sake, it takes 1 address for both billing and shipping.  All fields are required for the address.  It also has a status (InProgress, Placed, Shipped, or Cancelled) and we have some fairly intuitive rules about which status changes are valid (e.g. an order cannot be cancelled after it&#8217;s shipped).

Given the rules above, we start coding and come up with an interface like this. We&#8217;re showing the interface here for brevity &#8211; the implementation should be fairly obvious:

    public interface IOrder {
    
        void SetAddress(string streetNumber, string city, string state, string zip);
    
        // throw an exception if status == Shipped or Cancelled
        void Place();
    
        // throw an exception if status == In Progress or Cancelled
        void MarkShipped();
    
        // throw an exception if status == Shipped
        void Cancel();
    }
    

Stop. Do you see the problem here?

Think ahead a little bit. We need to both persist and edit this thing in the UI. We want to obey the [Single Responsibility Principle][2] so we don&#8217;t want to re-use our domain class for editing or persistence since those are different responsibilities. Thus, we end up with some additional classes: maybe `OrderRepository` for persisting, which has to map the attribs to a class like `OrderData` which is compatible with our ORM, and then we need to edit the thing in the UI so we make an `OrderViewModel` where we declare which fields are required in the UI. And now, we&#8217;ve got a LOT of code on our hands which in turn becomes a PITA to maintain. This is not what we hoped for when we ventured down the path of DDD. It was supposed to make our lives easier and now we have tons of code and a headache. (And possibly a heartarche depending on how passionate we are about our craft&#8230;)

So what did we do wrong? *We made the assumption that Order has to be one object.* It&#8217;s totally understandable &#8211; we&#8217;re making a model of an order, and in the paper world an order is one sheet of paper. However, if we naively carry the assumption that concepts map 1:1 to classes in our domain model we may well be making things worse. Here&#8217;s what we do instead:

The address on the order is just straight CRUD. There&#8217;s no behavior here other than validating that the required fields are there and we&#8217;ve got plenty of tools that will do this for us (e.g. DataAnnotations in the .NET world). Thus, we can use a single class to represent our persistence model all the way up to the UI. One class fills all of our needs as far as the address goes:

    public class OrderAddress {
        public Guid Id { get; set; }
    
        [Required]
        public string StreetNumber { get; set; }
    
        [Required]
        public string City { get; set; }
    
        [Required]
        public string State { get; set; }
    
        [Required]
        public string Zip { get; set; }
    }
    

No need for unit tests on this class since there&#8217;s no behavior to test. We can safely assume that validation is handled correctly by our client framework (e.g. jQuery validation in ASP.NET MVC) or at worst our database. (If we can&#8217;t safely assume this we may need to consider some new tools&#8230;)

As for the status changes, now we end up with this (same as before, but no `SetAddress()`):

    public interface IOrderStatus {
    
        // throw an exception if status == Shipped or Cancelled
        void Place();
    
        // throw an exception if status == In Progress or Cancelled
        void MarkShipped();
    
        // throw an exception if status == Shipped
        void Cancel();
    }
    

This also means we persist OrderAddress and OrderStatus as separate tables (or collections if you&#8217;re using a document database). Because they&#8217;re separate tables, we&#8217;ve also decreased our risk of concurrency issues in addition to making maintenance easier by decoupling. Now the code benefits from a true domain model where it&#8217;s actually helpful: where there&#8217;s behavior. And we can use plain old CRUD like we did before when there&#8217;s no behavior, just data.

So, we can summarize the lessons in this post in the following rules of thumb:

  1. If the current state of the data is not dependent upon the previous state of the data, then it&#8217;s simple CRUD and we do not need a domain model. We should use the same class from the DB all the way through the UI.
  2. If the current state of the data is dependent upon the previous state of the data, then it&#8217;s a state machine, has behavior, and we probably will benefit from a domain model.  Our behavior should be encapsulated in a class which has all methods and no mutators (setters).
  3. Break apart classes (and tables/document collections) accordingly in order to separate the state and behavior so we don&#8217;t write unnecessary code and create unnecessary coupling.

This is something I wish I had understood when I started doing DDD.  I hope this post clarifies things for you and saves you some trouble.

 [1]: https://en.wikipedia.org/wiki/Code_refactoring
 [2]: http://en.wikipedia.org/wiki/Single_responsibility_principle