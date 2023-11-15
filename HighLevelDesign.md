# High Level Design

======================

For ex: for a Views counting system for Youtube

Database

Webservice - Process incoming video view events and stores data in the DB


To retrieve view counts from the DB, lets introduce another web service

User —> Browser. ——> Processing service  ——> Database  ——>. Query service.  ——> Browser ——> User

Now we need to think about what data we need to store and how? In professional terms,  We need to define a data model.


What we store

2 options

1. Individual events (every click) - Store each individual video view event      OR
2. Aggregate data (e.g per minute) in real time - calculate views on the fly and store aggregated data.

In case of Individual event we capture all attributes of the event:
	- video identifier
	- Timestamp
	- User related info - country, device type, OS etc

For this option the advantages are	- Fast writes
								- Can slice and dice data however we need. And then filter based on certain attributes, aggregate based on some rules
								- Can recalculate numbers if needed, for ex if there was some bug in some business report then we can recalculate from scratch.
Drawback are: 					- Slow reads - Cannot read quickly, as we need to count each individual event when total count is requested.
								- Costly for a large scale (many event)  - to store all the raw events. Raw event storage can be huge.

For the aggregate data, we calculate total count per some interval, lets say 1 minute and lose details of each individual event 

For this option the advantages are	- Fast reads - we need not calculate each individual event, we just retrieve total count value
								- Data is ready for decision making - We can use this data for decision making in real time. For ex we may send this total count value to a recommendation service or trending service for popular videos to be promoted to trends.

Drawback are: 					- We can only query data the way it was aggregated
								- Ability to filter data or aggregate it differently is very limited.
								- Requires data aggregation pipeline. - we need to pre-aggregate data in memory before storing it in the database.
