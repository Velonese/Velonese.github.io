
## Overview
So looking into this deliverable, I tried to break down the requirements into two categories: Requirements that might demand a specific architecture or technology, and requirements that I know can be resolved through the application of various technologies.

## Requirements and Assumptions
I am going to break down my initial analysis of each one of the 11 requirements listed.

1. We are only considering departing flights.
    - I am going to operate under the assumption that despite only being concerned about departing flights (which might be described as flights where the 'Actual Departure Time' occurs in the past), that we will actually want to be capable of providing these 'historical' flight details. For example, a customer is running 1 hour late for their flight. They look up the information, they should see that the flights Actual Departure Time is 1 hour ago, instead of just getting some future flight, or a missing record.
1. There is a flight schedule, which defines when the regularly scheduled flights occur.
    - This sounds like our "source of truth" and needs to be highly accessible even if the rest of the system is suffering.
1. The airlines keep the schedule up to date when they make changes.
    - One assumption I am working on here is that there aren't changes coming from other sources - eg, the airport or government, for example. I don't know if this would be more of an issue than just permissions, but if there is a requirement for this, we should find out ahead of the demand.
1. The flight display has a list of upcoming departures.
    - There are a whole host of details to be determined here, for example, branding, logos, overall styles and interface designs. We should be cognizant of these things as we build this solution - I would propose that we design around a "theme-able" interface, such that airports or airlines that wished to do so could customize aspects of the appearance of the boards in some way. Whether this is part of the MVP is debatable, but we will be saving ourselves some pain by simply designing with this in mind.
    - There is an assumption that the systems hosting the boards at the airport don't have a specific software requirement - so if they are limited in some way, we should explore that before we begin.
    - I am also assuming the flight boards at the airport should be always available, even if the rest of the system is under duress. If the boards at the airport are out, this could cause panic, and we should design the system to be network/fault tolerant within the limits that are allowed by the law/airports.
1. Each flight has the following properties:…
    - I am assuming based on the example data in req. #2 "NZ0128 Flies to MEL at 6:30AM Mon, Wed, Fri" that we will need to differentiate status by a given instance of a flight - EG, if the Wednesday flight to MEL is delated to 7:00AM, the Friday flights Estimated Departure will still display as "6:30 AM."
    - Another assumption here is that the airlines will be able to provide this level of detail.
    - The only significant business logic described is in Requirement h.1: A departure gate is only assigned once a flight transitions to "Boarding" - other concepts, such as "A Cancelled flight should not have an Actual Departure Time" or other rules are not stated. We will likely desire such validations, but we should gather those before we build an interface for the Airlines.
    - What is the impact to departure time with regards to Time Zones, Daylight Savings, and other globalized considerations? I am going to leave this off the table for now, but we would need to analyze the requirements here. For this design, I am going to assume we will *always* present the time relative to the departing location, in local time. I am also going to assume that a 6:30 AM flight leaves at 6:30 AM whether it is daylight savings or not.
    - I am going to assume here that we can use the standard international codes to describe the destination as well as where it is departing from. (EG, it is not landing on some secret base or private airfield, these are all going from airport->airport)
    - Departure Time and scheduling. This is a big can of worms. There are a myriad of options that are likely available to the airlines, for example, a flight may run frequently between two close locations - and thus make multiple trips per day at differing times. A flight might be scheduled every other day, meaning the cycle only repeats on a fortnight (Mon,Wed,Fri,Sun,Tue,Thu,Sat) or even more bizarre and challenging to detail programatically. To avoid becoming horrifically quagmired in these details for the exercise, I am going to assume a given flight allows for a maximum of 1 departure per day, and a "flight schedule" is limited to a single departure time and multiple days. We could easily support additional options, but I would really just be guessing here as to what the limits of that would be.
1. The Ticker Board in the airport will get information from a Web API.
    - I will assume these are systems are all self-contained, and will run our software.
1. The flight information needs to be viewable over the internet.
    - This is an interesting requirement, and very ambiguous - will we be providing this interface? What requirements are needed for querying the data? Is there any requirement for a user identity before sharing flight information?
I am going to operate under the assumption that a given Airline, or other third party (such as a travel agency, google plugin, or the like) will be interfacing with our API, and we will NOT be providing an end-user interface for this feature.
1. The internet-accessible view must deal with very large traffic spikes.
    - I'll consider high-traffic demands in general, but getting some samples would let us better understand the need here.
1. Passengers can subscribe to a particular flight and receive push notifications when its details change.
    - This sounds fun!
1. Airlines must not be able to update the flight information of other airlines
    - We will need some kind of authentication & authorization mechanism.
1. The interface to update flight information must not be accessible from the internet.
    - This is an interesting requirement. I would be very curious as to the why behind this requirement - for example, if there is a legal requirement, it likely will include some kind of guidelines or expectations beyond simply forbidding access via "the internet" which is subject to interpretation. If this requirement comes from some other source , I would want to explore what the problem that we were attempting to solve was.
    
        It feels very much like a security requirement - again, if such a requirement exists, it should be based on some tangible behavior or experience we wish to avoid, which might help us make a more sensible requirement. If, for example, the source of the concern is something like "We cannot risk usernames and passwords allowing non-airline individuals to manage flights," then in this case we could implement sign-on solutions that may require certificates to be present, in addition to a airline-managed identity, two-factor authentication, and more. These solutions may allay the concerns and allow us to generate a simpler and more maintainable solution.

        In short: I think we can meet this requirement, but I would be desperately curious to get more details about it before we invest too much into it.

## Deliverables:
1. How the data should be modelled
    - So the flight information looks to have two major parts - the schedule, and the flight instance. I am going to define some terminology to help make this make sense.
        ```
        FlightSchedule:
            Id: Guid
            Airline: AirlineId
            FlightNumber: string
            Departure: AirfieldId
            Destination: AirfieldId
            ScheduledDepartureTimeMinutes: short
            ScheduledDepartureDays: flag, consisting of ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'] 
            //LastUpdated: datetime ??

        ## About ScheduledDepartureTimeMinutes ##
        Storing the data as minutes from midnight gives us some convenience when it comes to presenting the information accurately for the departing time zone, since the schedules don't refer to an actual date, and rather refer to a time in the day of any given date.
        
        FlightInstance:
            Id: Guid
            FlightScheduleId: Guid
            EstimatedDepartureTime: datetime
            ActualDepartureTime: datetime
            FlightStatusType: string
            DepartureGate: string
            //ScheduleEndDate: datetime ??
            //LastUpdated: datetime ??

        Airline:
            Id: Guid
            Name: String

        Airfield:
            Id: Guid
            Code: string
            Timezone: timezone            
        ```

    I think we will want to create a relational model that relates a schedule with an actual flight - as the two have some very distinct needs. We can think of the FlightSchedule as a "Template" and the Instance as the product of the application of that template.

    Since we are dealing with a schedule that may have many recurrences, we need to have some kind of reasonable cutoff that allows us to limit the number of future flights we are preparing for. This will prevent us from needing a record present for Wednesday, October 6th, 2962.

1. Components and technology choices
    I think there are a few deliverables here:

    - The tickerboard software itself, which I would propose we create a cloud-based react application supplying the display information itslef as well as a small configuration page to launch the display (providing airports the ability to configure parameters for the display board).

        Security here would not need to be extreme, we could secure by certificates, oauth, or really anything that could be safely configured once during setup and be reasonably expected to stay active - even through software updates.
        Initally, the configuration page would simply allow for the configuration of the Departing airport, (so we know which flights to show) and potentially some default query parameters (say, only show Air New Zealand flights for a given terminal/gate).

    - The database/datastore. I am going to suggest a cloud-based SQL database for the datastore. We can use Azure-hosting and have geo-redundancy for our data, giving us a lot of durability.

         Cost could be a significant consideration here. To be perfectly honest, I would need some research time for this one. There are a myriad of other technologies that I think are worth exploring - for example, Mongo, or Amazon RDS, blob storage, etc etc... Our needs aren't extremely unique, and should be satisfied by a variety of options. 

    - The data API.
        I would propose we present a graphQL API through dotnetcore. This service can be designed to run on a cluster through any Kubernetes provider, and be scaled with demand.
        
        We would still want to configure a CDN or leverage some significant caching to handle extreme spikes in demand. If we so desired we could configure multiple gateways into the API, allowing for airport ticker boards to get priority data, which we use aggressive caching in other cases (when individual users are checking a flight number, for example)
    
        My area of domain expertise is in dotnetcore APIs, and so I am inclined to lean that way, but I have long wanted to leverage Azure Functions and create some serverless architecture. From what I am seeing here, this could be a good opportunity for that here, which would give us lots of rapid scalability advantages, as well as allowing us to split out various functions of the API into distinct deliverables.

        This would also be where requests to be added or removed from any push notification system are stored. We would need to consider how we make that data retrievable, so that we could meet various privacy requirements, such as GDPR.
        
    - Scheduling service. 
        This service would handle the routine process of generating FlightInstances from FlightSchedules, and would house the Push Notification system, as well as potentially reading from trusted sources hosted by the airlines themselves.

    - Airline Interface application.
        To meet the requirement that the interface for updates to the flight schedule information is not available online, we could design an application for use by the airlines that would be used locally. This could be an enterprise-enabled applciation or even command line that allowed them to query, add, or make updates to schedules. The requirement of having no *interface* accessible on the internet would be met, but it still seems as though there is more to unpack from that requirement, and this approach is not as simple as a cloud-based application from a support perspective.

1. Which requirements are met, which aren't, and tradeoffs
1. Rough estimate of how long it would take
    With some lead time to design and plan, perhaps get some extra eyes on the process, I am confident that an MVP could be completely tested and available within 6 months or less. The first thing would be modelling the database, and providing test data for the other workstreams to take advantage of.

    With proper contracts, the various deliverables could almost all be worked on concurrently, with the small exception being establishing the database models in the beginning.

    Given the appropriate expertise and availability of team members, a team of 5 variously skilled engineers could handle something like this, but they would really have to have the right skillsets. I would steer towards a higher number of people involved, with a mix of junior and senior skills.


