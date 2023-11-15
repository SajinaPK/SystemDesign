# Requirement Clarification

1. Users  
2. Scale (read/write)  
3. Performance  
4. Cost  


For ex: for a Views counting system for Youtube

Users:


1. Who will use the system?    - 1. All YouTube viewers will see the total view count for a video? OR
                                    - a Per hour statistics available for the video owner only? OR  
                                - the count will be used by some machine learning models to generate recommendations?
2. How the system will be used?   -  Data may be used by the marketing dept to generate monthly reports.   (in other words data is not retrieved not often. ).    OR 
								- Data is sent to recommendation system in real time. (Meaning data retrieval is with a very high pace)
								



Scale:

Gives a clue of how much data is coming and getting retrieved  into/from the system.

1. How many read queries per second?   
2. How much data is queries per second?
3. How many video views are processed per second?
4. Can there be spikes in the traffic?


Performance:

Questions in this category helps us in the designing different options


1. What is the expected write-to-read data delay?  -  If we can count the views several hours later than the views actually happened - Here batch data/ stream processing design options can be considered.
										- But if time matters and we need to count views on the fly or in other words real-time 

2. What is expected p99 latency for read queries? - How fast data must be retrieved from the system. If response time has to be as small as possible then it’s a hint that we should count views when we write data and we should do minimal or no counting when we read data. In other words, data must always be aggregated.


Cost:


Help us evaluate technology stack


1. Should the design minimize the cost of development? - Is cost is an issue then reputed open-source options can be explored.
2. Should the design minimize the cost of maintenance- If future maintenance cost is a primary concern then we can consider public cloud services.



========================================

Functional requirements - system behaviour, or APIs, a set of operations the system will support 
Non - Functional requirements -  System qualities, such as fast, fault-tolerant, secure, 



Functional requirements:

The system has to count video view events. Here video is the input and count view events is the action.

CountViewEvent(videoId)
If we want to also calculate other events like likes shares subscribers etc we can generalize the API a bit and introduce event type parameter.
CountEvent(videoId, eventType)
eventType can be view like share etc
Further one step to the design we can also so other functions like sum average etc
processEvent(videoId, eventType, function)
Where function can be sum or count or average. With sum may be we can calculate the “total watch time” for a video.
Further improvement can be that the system can process all events as a single batch instead of one-by-one.
processEvents(listOfEvents)
Each event is an object that contains information about a video, type of event, time when the event happened etc.


Similar thought process can be applied to data retrieval API.

The system has to return video view counts for a time period.
getViewsCount(videoId, startTime, endTime)
getCount(videoId, eventType, startTime, endTime)
getStats(videoId, eventType, function, startTime, endTime)



Non- Functional requirements:


Scalable - (tens of thousands of views per second)
Highly Performant - (few tens of milliseconds to return total views count for a video)
Highly Available - (services hardware/network failures, no single point of failure)

Those 3 can be our top priority but lets add some more
[Note: CAP theorem states I need to choose between Availability and Consistency. I chose Availability ]


Cost minimization - 



========================================
