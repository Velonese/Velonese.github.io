# Opening Thoughts

This request really comes across in a relatively simple way, but exploring each point reveals more and more unanswered questions. This was a real wolf in sheep's clothing. The Tickerboard itself has a whole host of potential features involved, and is ripe for feature creep if we dove right in with such a bare-bones requirement.

I could probably go on for a while about "What happens to future flights when a schedule is changed or cancelled?" "What happens around daylight savings time with scheduled flight times." - we would want to get some domain experts in and really dig these out until we had some very rigid requirements.

Overall, for the estimates and technologies, I know I am leaning on my area of knowledge. As noted below, a lot of these would be heavily influenced by our existing infrastructure and what kind of people and expertise were available for the job.


# Flight Board Exercise

## How the data should be modelled
- So the flight information looks to have two major parts - the schedule, and the flight instance. I split these into a FlightSchedule and a FlightInstance, which, as the names imply represent a schedule for a flight and the instances of that flight within the schedule.
    ```
    FlightSchedule:
        Id: Guid
        Airline: AirlineId
        FlightNumber: string
        Departure: AirfieldId // Need to know where we are leaving from as well.
        Destination: AirfieldId
        ScheduledDepartureTimeMinutes: short
        ScheduledDepartureDays: flag //['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'] 
        LastUpdated: datetime // Help us determine when to invalidate or regenerate data
    ```

    **About ScheduledDepartureTimeMinutes**

    Storing the data as minutes from midnight gives us some convenience when it comes to presenting the information accurately for the departing time zone, since the schedules don't refer to an actual date, and rather refer to a time in the day of any given date. I have more on this in my notes below.
    
    ```
    FlightInstance:
        Id: Guid
        FlightScheduleId: Guid
        EstimatedDepartureTime: datetime
        ActualDepartureTime: datetime
        FlightStatusType: string
        DepartureGate: string
        ScheduleEndDate: datetime
        LastUpdated: datetime

    Airline:
        Id: Guid
        Name: String

    Airfield:
        Id: Guid
        Code: string
        Timezone: timezone            
    ```

I think we will want to create a relational model that relates a schedule with an actual flight - as the two have some very distinct needs. We can think of the FlightSchedule as a "Template" and the Instance as the product of the application of that template.

Since we are dealing with a schedule that may have many recurrences, we need to have some kind of reasonable cutoff that allows us to limit the number of future flights we are preparing for. I would assume that different airlines might have different cutoffs. For now, I will imagine that any flight will only be available 6 months ahead of its schedule. This will prevent us from needing a record present for Wednesday, October 6th, 2962.

## Components and technology choices

So as I started writing out various ideas, I landed on 5 related but distinct deliverables. I'll run through them briefly, and then in more detail below.

**PLEASE NOTE**
In *any* of these suggestions, I would be looking much more deeply into what our existing infrastructure supports, where we have expertise, and what we could leverage that we might have delivered in the past. There are dozens of factors that would influence the below answers which might come into the picture.

1. **The tickerboard software**

    A small configuration page, which allows the user to launch a tickerboard that can be full-screened on the airport monitors.
    - I would propose we create a cloud-based react application.
1. **The datastore**

    I landed on a relational database in the interest of long-term growth, but I think it would be worth discussing alternatives. Big requirements here are data integrity and high availability, which are more or less common requirements for any datastore.
    - I am going to suggest a cloud-based SQL database for the datastore. 
1. **The API**

    This provides the tickerboards with data, and third parties with various degrees of access to the data through different tools.
    -  I would propose we present a graphQL API through dotnetcore. This would include a CDN for spike protection, and hosted on a flexible scalable technology, such as AKS / EKS
1. **Scheduling service**

    This service would handle the routine process of generating FlightInstances from FlightSchedules, and would house the Push Notification system.
    - I would propose we build the backbone of this around a message bus, with dotnet / Azure function microservices to handle the various required features. 

1. **Airline Interface App**

    This could be an enterprise-enabled application or even command line that allowed them to query, add, or make updates to schedules. There is a lot to unpack from the requirements around this one, more on that below - this is my least comfortable recommendation by far.
    - This could be built as a Xamarin application secured via SSO with the airlines themselves.


## Requirements Met, Tradeoffs & Assumptions

As for the initial 11 requirements, we can meet them all with a few very large grains of salt.
This is a rough analysis of all the requirements.

- [x] We are only considering departing flights.
    - I am going to operate under the assumption that despite only being concerned about departing flights (which might be described as flights where the 'Actual Departure Time' occurs in the past), that we will actually want to be capable of providing these 'historical' flight details. For example, a customer is running 1 hour late for their flight. They look up the information, they should see that the flights Actual Departure Time is 1 hour ago, instead of just getting some future flight, or a missing record.
- [x] There is a flight schedule, which defines when the regularly scheduled flights occur.
    - This sounds like our "source of truth" and needs to be highly accessible even if the rest of the system is suffering.
- [x] The airlines keep the schedule up to date when they make changes.
    - This doesn't seem to be a requirement we can exactly fulfil ourselves - we can provide them the tools to do so, but my assumption is that if they can keep it up to date, they will.
    - One assumption I am working on here is that there aren't changes coming from other sources - eg, the airport or government, for example. I don't know if this would be more of an issue than just permissions, but if there is a requirement for this, we should find out ahead of the demand.
- [x] The flight display has a list of upcoming departures.
    - There are a whole host of details to be determined here, for example, branding, logos, overall styles and interface designs. We should be cognizant of these things as we build this solution - I would propose that we design around a "theme-able" interface, such that airports or airlines that wished to do so could customize aspects of the appearance of the boards in some way. Whether this is part of the MVP is debatable, but we will be saving ourselves some pain by simply designing with this in mind.
    - There is an assumption that the systems hosting the boards at the airport don't have a specific software requirement - so if they are limited in some way, we should explore that before we begin.
    - I am also assuming the flight boards at the airport should be always available, even if the rest of the system is under duress. If the boards at the airport are out, this could cause panic, and we should design the system to be network/fault tolerant within the limits that are allowed by the law/airports.
    - I am assuming this MVP will not be localized, but it shouldn't involve a lot of consideration to localize this aspect of the app.
- [x] Each flight has the following properties:…
    - I am assuming based on the example data in req. #2 "NZ0128 Flies to MEL at 6:30AM Mon, Wed, Fri" that we will need to differentiate status by a given instance of a flight - EG, if the Wednesday flight to MEL is delayed to 7:00AM, the Friday flights Estimated Departure will still display as "6:30 AM."
    - Another assumption here is that the airlines will be able to provide this level of detail.
    - The only significant business logic described is in Requirement h.1: A departure gate is only assigned once a flight transitions to "Boarding" - other concepts, such as "A Cancelled flight should not have an Actual Departure Time" or other rules are not stated. We will likely desire such validations, but we should gather those before we build an interface for the Airlines.
    - What is the impact to departure time with regards to Time Zones, Daylight Savings, and other globalized considerations? I am going to leave this off the table for now, but we would need to analyze the requirements here. For this design, I am going to assume we will *always* present the time relative to the departing location, in local time. I am also going to assume that a 6:30 AM flight leaves at 6:30 AM whether it is daylight savings or not.
    - I am going to assume here that we can use the standard international codes to describe the destination as well as where it is departing from. (EG, it is not landing on some secret base or private airfield, these are all going from airport->airport)
    - Departure Time and scheduling. This is a big can of worms. There are a myriad of options that are likely available to the airlines, for example, a flight may run frequently between two close locations - and thus make multiple trips per day at differing times. A flight might be scheduled every other day, meaning the cycle only repeats on a fortnight (Mon,Wed,Fri,Sun,Tue,Thu,Sat) or even more bizarre and challenging to detail programmatically. To avoid becoming mired in all these details for the exercise, I am going to assume a given flight allows for a maximum of 1 departure per day, and a "flight schedule" is limited to a single departure time and multiple days. We could easily support additional options, but I would really just be guessing here as to what the limits of that would be.
- [x] The Ticker Board in the airport will get information from a Web API.
    - I will assume these are systems are all self-contained, and will run our software.
- [x] The flight information needs to be viewable over the internet.
    - This is an interesting requirement, and very ambiguous - will we be providing this interface? What requirements are needed for querying the data? Is there any requirement for a user identity before sharing flight information?

    - I am going to operate under the ***(big)*** assumption that a given Airline, or other third party (such as a travel agency, google plugin, or the like) will be interfacing with our API, and we will NOT be providing an end-user interface for this feature. 
        
    - Building out an interface for end-users would likely require a few more people involved and some minor infrastructure tweaks (CDN placement etc...) to meet that need, as well as incorporate more localization considerations.

- [x] The internet-accessible view must deal with very large traffic spikes.
    - I'll consider high-traffic demands in general, but getting some samples would let us better understand the need here.
    - It's likely that we will get two distinct request types - those from the tickerboards would be broad queries, and those from individuals would likely be specific flight numbers. The data being retrieved is very lightweight, and doesn't require significant processing to present, so we gain a ton of ground with caching. With some anticipatory caching and refresh, we could handle very significant spikes.
- [x] Passengers can subscribe to a particular flight and receive push notifications when its details change.
    - This sounds fun!
    - Depending on requirements around data storage and what we provide here, we need to consider GDPR and other privacy behaviors. Most push notification systems give users the ability to manage the requests through their devices, via STOP or other message. The feature as described is straightforward - if we assume we isolate this data and provide a means to delete our records of user phone information via our website or other legally required requests.
    - Localization isn't being considered here either - I would need to research the impact this would have and how systems like this are commonly localized.
    - This feature would be the easiest to cut and implement down the road - it builds on the other functions without being a requirement to any of them, so it is unique in that way.
- [x] Airlines must not be able to update the flight information of other airlines
    - We will need some kind of authentication & authorization mechanism, I would implement SSO with our application and allow the airlines to manage access.
- [x] The interface to update flight information must not be accessible from the internet.
    - This is an interesting requirement. I would be very curious as to the why behind this requirement - for example, if there is a legal requirement, it likely will include some kind of guidelines or expectations beyond simply forbidding access via "the internet" which is subject to interpretation. If this requirement comes from some other source , I would want to explore what the problem that we were attempting to solve was.
    
        It feels very much like a security requirement - again, if such a requirement exists, it should be based on some tangible behavior or experience we wish to avoid, which might help us make a more sensible requirement. If, for example, the source of the concern is something like "We cannot risk usernames and passwords allowing non-airline individuals to manage flights," then in this case we could implement sign-on solutions that may require certificates to be present, in addition to a airline-managed identity, two-factor authentication, and more. These solutions may allay the concerns and allow us to generate a simpler and more maintainable solution.

        In short: I think we can meet this requirement, but I would be desperately curious to get more details about it before we invest too much into it.
    - Localization! As this might be a much more interface-heavy application with a variety of error messages / conditions and validations, we might need additional time to handle localization.

## Rough estimate of how long it would take

This is one place I struggle with, expect this estimate to be *especially* rough. It's hard to predict how much time you can really devote to a project, as the work doesn't happen in a bubble. I would be seeking a lot of advice here before having any confidence in my results.

### Bare Mininmums:

With some lead time to design and plan, perhaps get some extra eyes on the process, I am confident that an MVP could be completely tested and available within approximately 6 months or less of the epics being finalized and the above requirements / debates being resolved - assuming that we were really going for **MVP**. I think a team of 5-7 could handle the workload, assuming we had the skills needed. The first thing would be modelling the database, and providing test data for the other workstreams to take advantage of.

With proper contracts defining relationships between the various deliverables could almost all be worked on concurrently, with the small exception being establishing the database models in the beginning.

This MSPaint masterpiece describes the overall way I would align the different deliverables, regardless of time scale.

![Workflow example courtesy of MSPAINT](/img/timeline_sample.png "Workflow example courtesy of MSPAINT")

### Reality is different:

In my professional experience, an MVP is rarely *minimum* - scope creeps in as demos begin and salespeople make contact with customers.

My rough estimate is assuming appropriate expertise and availability of team members, a team of 7 variously skilled engineers could handle something like this. To be honest, many choices are malleable here, and even some relatively innocuous changes could shift the deadline significantly, especially given the abilities of the team if we were already making decisions under a tight deadline.

I would steer towards a higher number of people involved, with a mix of junior and senior skills, as there is a lot of work of various different levels needed. It honestly depends on how focused the team could be on this work. In a "startup-like" environment, I have seen projects of this scope come together surprisingly quickly, but I wouldn't want anyone racing on something like this - a big miss here could cause a lot of problems for people and airlines, and having a longer deadline means we can develop without speed in mind, and potentially use fewer people.


## Testing, maintainability, and other considerations

### The tickerboard software itself
- I would like to see built alongside tests. Cypress testing or similar could easily handle the demands of the app. The app specific requirements would be important here, with things like failed requests and other conditions needing to be handled gracefully.

- A basic react app would mean there is a lot of expertise available for maintenance - plenty of people know react. We would want to balance this when picking a stack - is it trending? Who knows it? While the app doesn't have a great deal of complexity, we would be wise to keep it up to date.

- Security here would not need to be extreme, we could secure by certificates, oauth device flow, or really anything that could be safely configured once during setup and be reasonably expected to stay active - through software updates and the like. The last thing I would want to see when racing for a flight at the airport is an error or a login screen.

- Initially, the configuration page would simply allow for the configuration of the Departing airport, (so we know which flights to show) and potentially some default query parameters (say, only show Air New Zealand flights for a given terminal/gate).

  ![Another MSPAINT masterpeice](/img/tickerboard_user_flow.png "Workflow example courtesy of MSPAINT")


### The database/datastore
- I am most familiar with cloud-based SQL databases, and similar relational systems. We can use Azure-hosting and have geo-redundancy for our data, giving us a lot of durability.

- Cost could be a significant consideration here. To be perfectly honest, I would need some research time for this one. There are a myriad of other technologies that I think are worth exploring - for example, Mongo, or Amazon RDS, blob storage, etc etc... Our needs aren't extremely unique, and should be satisfied by a variety of options. 

### The data API
- I think it's hard to go wrong with a graphQL API through dotnetcore. This service can be designed to run on a cluster through any Kubernetes provider, and be scaled with demand.
- We would still want to configure a CDN or leverage some significant caching to handle extreme spikes in demand. If we so desired we could configure multiple gateways into the API, allowing for airport ticker boards to get priority data, which we use aggressive caching in other cases (when individual users are checking a flight number, for example)
- My area of domain expertise is in dotnetcore APIs, and so I am inclined to lean that way, but I have long wanted to leverage Azure Functions and create some serverless architecture. From what I am seeing here, this could be a good opportunity for that here, which would give us lots of rapid scalability advantages, as well as allowing us to split out various functions of the API into distinct deliverables.
- This would also be where requests to be added or removed from any push notification system are stored. We would need to consider how we make that data retrievable, so that we could meet various privacy requirements, such as GDPR.
- Testing through xunit, leveraging of standard coding practices, such as a service layer pattern with dependency injection would give us enormous testability here. With unit tests running as part of the build/checkin cycle, code coverage goals, and a smattering of integration and end-to-end tests, we could feel very confident about both the functionality and performance of the API.
- This tech stack has intended future support for many years, and with minimal external needs, we would likely be able to keep this service available for many years with minimal refactoring/maintainence.
    
### Scheduling service

- Building this around a message queue system gives us a lot of future flexibility, as the technologies exist in many forms, from Service Bus to rabbitMq. As it is my area of greatest knowledge, I am most comfortable proposing dotnet / Azure function microservices to handle the various required features (Push notifications, updating the datastore, creating flight instances from flight schedules)
- These various services lend themselves well to testing, and are individually scalable.
- Durable queues prevent data loss in the case of significant disruption

### Airline Interface application.
- Ideally this would be built using Xamarin, but the details of the requirements and exacting needs would ***really*** heavily shape the results here.
- There are too many unknowns to comfortably settle in on a solution here, and I think I would need to get more details and explore more before making a proposal.
- This could be an enterprise-enabled application or even command line that allowed them to query, add, or make updates to schedules. The requirement of having no *interface* accessible on the internet would be met, but it still seems as though there is more to unpack from that requirement, and this approach is not as simple from a maintenance and support perspective as a cloud-based application from a support perspective.


# Closing

Thanks for reading through all of this and making it this far :)

That was a very interesting process, and I feel as though I am arbitrarily stopping my effort here - there are plenty of unanswered questions and unexplored technologies, but I am hoping I have provided what you are looking for in regards to the question being answered. I hope to have the opportunity to discuss and see how other people approached the same problem. 
Regardless of the outcome, I would also appreciate feedback on my submission, should you have the chance.

I really appreciate your consideration!

/*John*
