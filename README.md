# fluent-cqrs
"The Next Generation" CQRS Framework for .Net applications

### Attenzione Attenzione:
There is an **Api break**. The method `OnError` was splitted into `CatchException`
(raised by unexpected System Exceptions) and `CatchFault` (raised by Business Faults).
By default 'Do' throws all exceptions directly. Use 'Try' for catching errors.

####2.0.3.3
Now you can't use `Changes`, `History` or `MessagesOfType` outside of an aggregate.
Someone has done it in the past and it feels like a pinch.

####2.0.3.4
The `With(command)` method gets a sister, `With(new AggregateId("cool Id"))`. You can use it for providing Aggregats in
CommandHandlers by commands without Id for the Aggregate. May be you can retrieve the Id by an other way like Read Model.

---

Why fluent? Just look at this:

```csharp
    public class AwsomeCommandHandler
    {
        Aggregates _aggregates;

        public AwsomeCommandHandler(Aggregates aggregates)
        {
            _aggregates = aggregates;
        }

        public void Handle(SuperDuperCommand command)
        {
            _aggregates
                .Provide<[AnAggregateYouLike]>
                .With(command)
                // or you can use .With(new AggregateId("cool id"))
                .Do(yourAggregate => yourAggregate.DoSomethingWith(command.Data));

            // You want it with exception handling?
            // Lets do it

            _aggregates
                .Provide<[AnAggregateYouLike]>
                .With(command)
                // or you can use .With(new AggregateId("cool id")
                .Try(yourAggregate => yourAggregate.DoSomethingWith(command.Data))
                .CatchException(exception=> handleThis(exception));

            // And here a very simple way to catch business faults which might be thrown within the Aggregate

            _aggregates
                .Provide<[AnAggregateYouLike]>
                .With(command)
                // or you can use .With(new AggregateId("cool id")
                .Try(yourAggregate => yourAggregate.DoSomethingWith(command.Data))
                .CatchFault(fault=> handleThis(fault))
                .CatchException(exception => handleThis(exception));
        }
    }
```
Uhhh... this is the **complete handling** of a Domain Command.

---

Ok, but what do I have to do to **publish** the new state, aka **Domain Events**?

This is simple. You assign any Event Handler you like by chaining it by the `And` method after `PublishNewStateTo`.
For example you have three Event Handlers:
```csharp
    var _aggregates = Aggregates.CreateWith(yourExtremeGoodEventStoreInstance);
    var firstEventHandler = new SampleEventHandler();
    var secondEventHandler = new ReportingHandler();
    var thirdEventHandler = new LoggingHandler();

    _aggregates
        .PublishNewStateTo(firstEventHandler)
        .And(secondEventHandler)
        .And(thirdEventHandler);
```
now all your cool Event Handlers receiving all changes of an aggregate.

---

All right... Now let's have a look inside the **Aggregate**.
```csharp
    class CoolAggregate() : Aggregate
    {
        public CoolAggregate(string id, IEnumerable<IAmAnEventMessage> history)
          : base(id, history) { }
        ...
        public void DoSomethingHelpful()
        {
            Changes.Add(new SomethingHappend());
        }
        ...
    }
```

The very cool thing of an event store is that you can make decisions based on the history. There are two different ways doing that.
The simplest way is using `MessagesOfType<MessageType()>`.

```csharp
class CoolAggregate() : Aggregate
  {
      ...
      bool ItHappened => MessagesOfType<SomethingHappend>().Any();
      int HappeningCounter => MessagesOfType<SomethingHappend>().Count();
      AType CoolStuff => MessagesOfType<SomethingHappend>().Last().CoolStuff;
      ...
  }
```
This might be enough for a lot of use cases, though sometimes it is not that easy and multiple events are playing together. Now it's time for the CQRS Swiss knife, as the current state is just a left fold/ aggregate  of all preceding events.

In that case you set up the rules that should happen for any event you like. use it to restore a complex entity or a simple value, it is up to you.

```csharp
class CoolAggregate() : Aggregate
  {

      OtherType BetterStuff =>
            InitialzedAs<OtherType>() //InitializeWith(new BetterStuff())
              .ApplyForAny<ItHasStarted>(message => message.BetterStuff) // former state can be ignored
              .ApplyForAny<ThisHappened>(state, message => f(state, message.OtherStuff)) // the former state can be used for calculations
              .ApplyForAny<ThatWasCool>(new BetterStuff(param)) // well, or set to a constant
              .Otherwise(() => throw new BusinessFault () ) // or set it to anything you like
              .AggregateAllEvents();

  }
```

In some cases you want to save an event only once, but if you add the event into the list of Changes like above,
the event will save every time.

Bad, very bad. Here comes the hero 'Replay' ...

```csharp
    class CoolAggregate() : Aggregate
    {
        ...
        public void DoSomethingHelpful()
        {
            if(MessagesOfType<SomethingHappend>().Any)
            {
                Replay(new SomethingHappend());
            }
           else
           {
               Changes.Add(new SomethingHappend());
           }
        }
        ...
    }
```

The `Replay` method prevents you for multiple equals events. It fires to the event handlers, though it will not be recorded as event.

---

Nice, really nice... But what if you want to **replay all Events** of an Aggregate? Yes... you couldn't... still now.

To replay all Events of an Aggregate code this:
```csharp
    _aggregates
        .ReplayFor<[AnAggregateYouLike]>()
        .EventsWithAggregateId(aggrId)
        .ToAllEventHandlers();
```
This published all Events of the Aggregate with the given ID to all registered Event Handler.
If you want to publish to only one special Event Handler change your Code to:
```csharp
    _aggregates
        .ReplayFor<[AnAggregateYouLike]>()
        .EventsWithAggregateId(aggrId)
        .To([OneOfYourEventHandler]);
```
This is simple, as well.

You can also replay events of an aggregate type:
```csharp
    _aggregates
        .ReplayFor<[AnAggregateYouLike]>()
        .AllEvents()
        .ToAllEventHandlers();
```
You can also filter for certain event messages:
```csharp
    _aggregates
        .ReplayFor<[AnAggregateYouLike]>()
        .AllEvents()
        .OfType<[AnEvent]>()
        .ToAllEventHandlers();
```
---

~tbc
