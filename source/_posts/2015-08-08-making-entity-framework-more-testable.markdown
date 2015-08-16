---
layout: post
title: "Making Entity Framework More Unit-Testable"
date: 2015-08-08 09:56:22 -0400
comments: true
categories: entityframework, testing
---

## Abstract

For developers using [Entity Framework](https://entityframework.codeplex.com/), unit testing code that depends on the `DbContext` class is not the easiest thing.  If we're interested in doing unit testing, we need to be able to use an in-memory ("fake" in [Martin Fowler's terminology](http://martinfowler.com/articles/mocksArentStubs.html)) version of `DbContext` comprised of lists of objects so that our tests do not hit the database.  This series of posts takes you through my process in getting to a facade on top of Entity Framework that results in testable code and tests that are fast to write.

*Note:* The issues described are only known to apply to Entity Framework <= 6.x.  EF 7 is a complete rewrite, so it's possible that these issues have since been addressed.

## tl;dr

You can jump directly to [the code](https://github.com/jkodroff/ef-testable), or you can reference [the client code NuGet package](https://www.nuget.org/packages/EntityFrameworkTestable/) in your main assemblies and [the testing code NuGet package](https://www.nuget.org/packages/EntityFrameworkTestable.Testing/) in your testing assemblies.

## Part 1 - Getting to a Reusable Abstraction

We want to depend on interfaces instead of concrete implementations, per the "D" ([Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle)) in [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)).  So how can we do this with Entity Framework?

Let's suppose we have the following `DbContext`:
```c#
public class MyDbContext : DbContext 
{
  public DbSet<Order> Orders { get; set; }
  // a bunch of other tables...
}
```

... we might be consuming it like this:
```c#
var db = new MyDbContext();
var order = db.Orders.Single(x => x.Id == id);
```

... so if we naively extract an interface, then it will lead to something like this:
``` c#
public interface IMyDbContext
{
  DbSet<Order> Orders { get; set; }
  // a bunch of other tables
}
```

And that's not so good.  

We have to change `IMyDbContext` along with `MyDbContext` every time we add a table to the database: We would have to add `DbSet<T>` properties to both the interface and the implementation.  We also cannot reuse this code across projects because our interface is tightly coupled to the particular database schema we're using.  

Fortunately, `DbContext` exposes a `Set<T>()` method ([MSDN Docs](https://msdn.microsoft.com/en-us/library/Gg696521%28v=VS.113%29.aspx)) which can help us solve this problem.  So, if we change how we consume our `DbContext` from `db.Orders` to `db.Set<Order>()`, we can now define our interface like this:

```c#
public interface IDataSession : IDisposable
{
  IDbSet<T> Set<T>() where T : class;
  
  // We also obviously need to persist our changes, so:
  void SaveChanges();

  // I also have use cases where I need to drop down to raw SQL, so:
  int SqlCommand(string sql, params object[] parameters);
}
```

The name change on the interface from `IMyDbContext` to `IDataSession` is intentional: it reflects that this interface is no longer a representation of a _particular_ database context (i.e. `MyDbContext`), but _any_ database context.  And we can create an implementation for it like so:

```c#
public class EntityDataSession : IDataSession
{
    // This is the EF class, not ours:
    private DbContext _dbContext;

    public EntityDataSession(DbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public void Dispose()
    {
        _dbContext.Dispose();
    }

    public IDbSet<T> Set<T>() where T : class
    {
        return _dbContext.Set<T>();
    }

    public void SaveChanges()
    {
        _dbContext.SaveChanges();
    }

    public int SqlCommand(string sql, params object[] parameters)
    {
        return _dbContext.Database.ExecuteSqlCommand(sql, parameters);
    }
}
```

We can use our DI container to give us an `EntityDataSession` with whatever `DbContext` we want, and we can reuse this code across projects.  The code below is for [StructureMap](http://docs.structuremap.net/)) but your DI container should have similar constructs:

```c#
// Somewhere in your application's startup code:
For<IDataSession>().Use<EntityDataSession>();

// Because EntityDataSession takes a DbContext as a constructor param:
For<System.Data.Entity.DbContext>().Use<MyDbContext>();
```

And we might consume our code like this.  (Sending your data model directly to a view is not advisable for production-quality code, but this is just an example):

```c#
public class SomeController : Controller 
{
    private IDataSession _dataSession;

    public SomeController(IDataSession dataSession) {
        _dataSession = dataSession
    }

    public ActionResult Index() {
        return View(_dataSession.Set<Orders>())
    }
}
```

## Part 2 - Testing

We want to be able to write our tests with a minimum of incidental complexity.  In other words, we want to be able to create an in-memory implementation of `IDataSession`, specifying the tables and entities we need for the test - no more, and no less.  In order to provide a super-simple interface for our tests, we need to hide some complexity within our `IDataSession` wrapper.

Unfortunately, I was not able to accomplish this succintly without including a mocking library, albeit [a very nicely written one](http://www.moqthis.com/)).  Any suggestions for improvement here would be greatly appreciated.  The implementation follows:

```c#
// Only to be included in test assemblies:
public static class DataHelper
{
    public static IDataSession Session()
    {
        return new MockDataSession();
    }

    public static IDataSession Session<T>(params T[] set) where T : class
    {
        return Session().AddSet(set);
    }

    public static IDataSession AddSet<T>(this IDataSession session, params T[] set) where T : class
    {
        (session as MockDataSession)
            .Mock
            .Setup(x => x.Set<T>())
            .Returns(set == null ? new FakeDbSet<T>() : new FakeDbSet<T>(set));

        return mockSession;
    }

    private class MockDataSession : IDataSession
    {
        private Mock<IDataSession> _mock = new Mock<IDataSession>();

        internal Mock<IDataSession> Mock
        {
            get { return _mock; }
        }

        public void Dispose() { }

        IDbSet<T> IDataSession.Set<T>()
        {
            return _mock.Object.Set<T>();
        }

        // This doesn't seem to be something that's really testable,
        // so we'll concede this and throw an exception:
        public IEnumerable<T> SqlQuery<T>(string sql, params object[] parameters)
        {
            throw new NotImplementedException();
        }

        // There's nothing to "save", per se so it's just a NOOP
        public void SaveChanges() { }
    }
}
```

## Part 3 - Putting it All Together

Now that we have structured our client code so that it depends on interfaces instead of implementations, and we have an in-memory version of `IDataStore`, we can create tests that are fast to write, fast to execute, and test the necessary behavior of our program.  Here's an example of how we can do this:

Let's suppose we have a business rule that states "customers on credit hold cannot place new orders" (because they haven't paid their bills).  That rule might give us a controller that looks like this:

```c#
public class OrdersController : Controller 
{
    // In a real application, we probably would not have IDataSession 
    // as a direct dependency in a controller, but for our purposes 
    // this is fine.
    private IDataSession _dataSession;

    public OrdersController(IDataSession dataSession) {
        _dataSession = dataSession;
    }

    public ActionResult New(Guid customerId) {
        if (_dataSession.Set<Customer>().Single(x => x.Id == customerId).OnCreditHold)
            throw new Exception("Customer is on credit hold and cannot place new orders.");

        return View(new NewOrderViewModel {
            CustomerId = customerId;  
        });
    }
}
```

We can now write a test for this controller like this (the assertion syntax comes from [FluentAssertions](http://www.fluentassertions.com/)):

```c#
[TestFixture]
public class OrdersControllerTests
{
    [Test]
    public void New_CustomerOnCreditHold()
    {
        var customerId = Guid.NewGuid();

        var data = DataHelper
            .Session(new Customer {
                Id = customerId  
            });

        new OrdersController(data)
            .Invoking(x => x.New(customerId))
            .ShouldThrow<Exception>("*on credit hold*");
            //                      ^^^^^^^^^^^^^^^^^^
            // FluentAssertions will accept wildcard characters for
            // matching exception text.
    }
}
```

And that's it!

## Conclusion

Entity Framework can be a useful tool in the toolbelt, but it requires a little bit of work on top to keep your code testable.  Doing that work is well worth the gains in the ability to write tests that are fast to write, fast to execute, and let you focus on the business logic of your application.

Happy coding!