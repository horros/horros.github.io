---
layout: post
title:  Local dev with Event Grid
date:   2021-01-27 00:15:19 +0200
---
[Azure Event Grid](https://azure.microsoft.com/en-us/services/event-grid/) routes events from (almost) any source to (almost) any destination. It is a fantastic and low-cost way to build an event driven application. It does have a tiny drawback though: In order to develop your app with EventGrid events, you need to be online and you need to open something like an ngrok tunnel to your local machine. In this post we'll explore how we can get around that.

Azure Event Grid allows you to create _topics_ to which you can post _events_, and you can add _subscribers_ to the topics (optionally with _event filters_), and for every incoming event, Event Grid will send the event to all the subscribers (whose filters match). It is fast, scalable, and cheap as peanuts (something like 0.50€ per million operations). The drawback is local development. If you want to develop with Event Grid, the only option is to use something like [ngrok](https://ngrok.com) to open a tunnel from the internet to your machine. If you have e.g. more than one microservice that you're developing simultaneously, you're probably going to have to make some kind of ingress-type controller that forwards traffic to the correct microservices based on URL path or so. This of course also means you need to be online at all times when developing.

Enter [AzureEventGridSimulator](https://github.com/pmcilreavy/AzureEventGridSimulator) by Paul Mcilreavy! It's a .NET Core 3.1 app that does what it says on the tin, mimics Azure Event Grid, but can be run locally. You can simulate multiple topics through a very easy configuration system (basically it opens up multiple ports, and posting to those ports delivers the events to the specified subscribers).

Before we move on, the new Azure.Messaging.EventGrid 4.0.0-beta4 API has changed quite a bit. I could not get `EventGridPublisherClient` to work nicely with AzureEventGridSimulator, so I had to write a little wrapper around it and manually handle posting the events to AzureEventGridSimulator. You can pick up the wrapper on [my GitHub page](https://github.com/horros/EventGridPublisherClientEmulator). You'll need this to continue, as we'll use the beta version of the API.

As an example I'll demonstrate how this all fits together. First things first, clone AzureEventGridSimulator somehwere. Then, let's create a new ASP.NET Core API project (I'm using .NET 5), which we'll call it AzureEventGridSimulatorExample. We also need to add the Azure EventGrid 4.0.0-beta4 package. Make sure you check the "Include prerelease" if you're using Visual Studio. If you're using `dotnet add package` use the `--prerelease`-switch.

![NuGet package install](/assets/images/azegsim/add-nuget.png)

We'll delete the default `controllers/WeatherController.cs` and `Weather.cs`, and create a new controller called `EventTestController` (I will assume you are somewhat familiar with ASP.NET Core and know how to add controllers). In order for Event Grid to validate webhook subscribers, it sends a special request with an `aeg-event-type` header set to `SubscriptionValidation`, and it also sends a validation code in the data. What subscribers must do is determine if the validation request is for an expected subscription, and reply with the same validation code they received. Fortunately this is easier than it sounds.

{% highlight csharp linenos %}
namespace AzureEventGridSimulatorExample.Controllers
{
    [Route("/EventTest")]
    public class EventTestController : Controller
    {
        private bool EventTypeNotification
                => HttpContext.Request.Headers["aeg-event-type"].FirstOrDefault() ==
                   "Notification";

        private bool EventTypeSubcriptionValidation
            => HttpContext.Request.Headers["aeg-event-type"].FirstOrDefault() ==
               "SubscriptionValidation";

        [HttpPost]
        public async Task<IActionResult> Post()
        {
            using var reader = new StreamReader(Request.Body, Encoding.UTF8);
            var jsonContent = await reader.ReadToEndAsync();

            if (EventTypeSubcriptionValidation)
            {
                return HandleValidation(jsonContent);
            } 
            else if (EventTypeNotification)
            {
                return await HandleEventGridEvents(jsonContent);
            }
            return BadRequest();
        }

        private JsonResult HandleValidation(string jsonContent)
        {
            var gridEvent = EventGridEvent.Parse(jsonContent).First();
            var data = gridEvent.GetData<SubscriptionValidationEventData>();

            return new JsonResult(new
            {
                validationResponse = data.ValidationCode
            });
        }

        private IActionResult HandleEventGridEvents(string jsonContent)
        {
            throw new NotImplementedException();
        }

    }
}
{% endhighlight %}

Quick rundown: For HTTP posts to `/EventTest`, read the request body as JSON. Line 20 checks if the request headers contain the `SubscriptionValidation` event type, and if so run the `HandleValidation` method. The method uses the Parse method from `EventGridEvent` to parse JSON into an EventGridEvent object, and gets the event data (which for subscription validation requests is a `SubscriptionValidationEventData` object). The data object contains the validation code (and also a validation URL you can use to make sure the request is valid). This validation code can just be wrapped in a JsonResult and returned.

Let's head over to AzureEventGridSimulator. Open up `appsettings.json` and configure it like this:

{% highlight json linenos %}
{
  "topics": [
    {
      "name": "Test",
      "port": 60102,
      "key": "DummyKey",
      "subscribers": [
        {
          "name": "TestSubscriber",
          "endpoint": "http://localhost:5001/EventTest",
          "filter": {
            "includedEventTypes": [
              "GuestList.PersonAdded"
            ]
          }
        }
      ]
    }
  ]
}
{% endhighlight %}

This should be fairly straightforward to understand. We name a topic, tell the simulator to listen on port 60102 for these events, and if we get an event with credential key "DummyKey", we'll forward the event to the subscribers listed in the subscribers section.

Fire up the test project (AzureEventGridSimulatorExample) and AzureEventGridSimulator after it, and you should see something like this:

![Azure Event Grid Simulator up and running](/assets/images/azegsim/azegsimrunning.png)

This means Event Grid Simulator connected to our "microservice" and was successfully validated the subscription! Now it's time to write the code that handles the events.

Let's say our fancy web app keeps track of who has been added to a guest list for some event. That is going to be the data we send, just the person's first name and last name. We create a new folder called DTO, and create a file in it that's called `Person.cs`.

{% highlight csharp linenos %}
namespace AzureEventGridSimulatorExample.DTO
namespace AzureEventGridSimulatorExample.DTO
{
    public class Person
    {
        public string Firstname { get; set; }
        public string Lastname { get; set; }
    }
}

{% endhighlight %}

Then we need a handler, so we'll make a folder called "Handlers" and create a file called `GuestListEventHandler.cs` with the following content:

{% highlight csharp linenos %}
using Azure.Messaging.EventGrid;
using AzureEventGridSimulatorExample.DTO;
using System;

namespace AzureEventGridSimulatorExample.Handlers
{
    public class GuestListEventHandler
    {
        internal static void HandlePersonAdded(EventGridEvent ev)
        {
            var person = ev.GetData<Person>();
            Console.WriteLine($"Got event {ev.EventType} with person {person.Firstname} {person.Lastname}");
        }
    }
}
{% endhighlight %}

The last thing we need is to actually handle the incoming EventGridEvents in the controller, so modify the `HandleEventGridEvents` method:

{% highlight csharp linenos %}
private IActionResult HandleEventGridEvents(string jsonContent)
{
    EventGridEvent[] events = EventGridEvent.Parse(jsonContent);

    foreach (EventGridEvent ev in events)
    {
        switch (ev.EventType)
        {
            case "GuestList.PersonAdded":
                GuestListEventHandler.HandlePersonAdded(ev);
                break;
            default:
                return BadRequest();
        }
    }
    return Ok();
}
{% endhighlight %}

We'll deserialize a list of EventGridEvents from the JSON payload, loop over them and check the event type. If the event type is "GuestList.PersonAdded", we'll pass that to the GuestListEventHandler class.

Now, to actually send messages to Event Grid Simulator, we now need to use EventGridPublisherClientEmulator. Clone the repository somewhere if you haven't already, build it, and add it as a project reference to the test project. Or you can just copy the .cs file into the project.

We'll add a new simple (and very insecure and badly written, but it'll suffice for this demo) POST-endpoint to the controller that'll take a firstname and lastname parameters, construct the EventGridPublisherClientEmulator and send the events.

{% highlight csharp linenos %}
[Route("/SendEvent")]
[HttpPost]
public async Task<IActionResult> SendEvent([FromQuery]string firstname, [FromQuery]string lastname)
{
    Uri endpoint = new Uri("https://localhost:60102/api/events?api-version=2018-01-01");
    var key = new AzureKeyCredential("DummyKey");

    ClientEmulator clientEmu = new ClientEmulator(endpoint, key);

    Person p = new Person
    {
        Firstname = firstname,
        Lastname = lastname
    };

    EventGridEvent ev = new EventGridEvent(p, Guid.NewGuid().ToString(), "GuestList.PersonAdded", "1.0");

    List<EventGridEvent> events = new List<EventGridEvent>();
    events.Add(ev);

    Response response = await clientEmu.SendEventsAsync(events);

    if (response.Status == 200)
    {
        return new OkResult();
    } 
    else
    {
        return new BadRequestResult();
    }    
}
{% endhighlight %}

Line 5 and 6 constructs the URI and Key Credential for AzureEventGridSimulator. The path in the URI must be excatly as shown, this is the same enpoint Azure Event Grid listens on too (with some modifications). We then construct our ClientEmulator. This should obviously not be done here, but it should use some sort of service class that is injected with DI for this, but for this example this'll do. Then we construct a new EventGridEvent. The constructor parameters are data, subject, event type and data version. For more information on the Event Grid schema, see <https://docs.microsoft.com/en-us/azure/event-grid/event-schema>. Lastly, we send the events, and check the response.

So let's fire up Postman and fire away a POST-request!

![Postman running a HTTP POST](/assets/images/azegsim/post.png)

Having a look at the output from AzureEventGridSimulator we see

![AZEGSIM success](/assets/images/azegsim/azegsuccess.png)

And our microservice has indeed printed what we expected!

![SUCCESS](/assets/images/azegsim/success.png)

You should now be able to just add more subscriptions, events and handlers! Thanks for reading, and I hope this post has been of some use to you!

The source code for the example controller can be found at <https://github.com/horros/AzureEventGridSimulatorExample>.
