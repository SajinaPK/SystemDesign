# Design Tiny URL

**Functional Req:**  
—————————

Short a Url  
Give the long URL when  


**Non Functional Req:**  
———————————

Low latency  
High Available  


**High level design:**  
——————————

Length of the URL ??   
Fixed length is better. We can also think of agnostic where we can keep increasing the size length.  
  
If X request comes per second then the data we have to store for 10 years suppose is:  
  
X * 60 * 60 * 24 * 365 * 10 = Y  - URLs to store over 10 years  
  
A-Z a-z 0-9 = 62 char  
  
If the length of the URL is 1 then we need 62 combinations  
If length of URL is 2 then combinations are Math.pow(62,2)  
  
For n size URL its Math.pow(62,n)  
We need to find an n such that Math.pow(62,n) > Y  
  
Or n = log Y (base 62)  
  
62 to power 6 is about 58 Billion  
62 to power 7 is about 3.5 Trillion  
  
So let’s stick to 7 characters for now.  
  
**User -> Web Server -> Load balancer -> Short URL Service —> Cassandra** 
  
Suppose multiple such “Short URL Service” and then we can get collisions if one long URL is converted to 2 different tiny URLs by different such services. What we can do is we can check the DB if it already exists etc. but that’s not an efficient solution.  
  
So we need a predictable way to generate this knowing there will be no collisions.   
  
Redis can be used… And give a uniques number to the services when they ask the redis cluster. From this number we can do some hashing with base 62.   
  
But this puts a lots of load on the Redis. But this can become a single point of failure. Also scalability of Redis is also a problem during higher load.  
  
We can also have multiple Redis. If both Redis start from same index then can generate duplicates, one way to solve this is by providing different ranges to each Redis. But then we need another service to manage what series or ranges are given to each Redis.   

  
**Design:**   
  
Convert from Base 10 to Base 62 for converting Long URLs to sort URL.   
  
  
Long to short URL —> Load Balancer —> Short URL Service 1 ——> Token Service       <———> Cassandra  
								                  —> Short URL Service 2 ——>  

Token service will give ranges like 1-1000, 1000-2000 etc. then the Short URL Service 2 might generate a number sa 1001 and then convert this to base 62 and then after storing in the database it will return this to the user. 

When the ranges are exhausted Token service will generate new ranges. These ranges should be unique.
We can scale this by either increasing the range or increase the number of Token service.

If suppose after the ranges are assigned if the service dies then the ranges are lost.. There is no way to track how many of those tokens were used before dying. But tracking them could be a massive problem. So better to lose them.

**Why Cassandra?**
MySQL can be used and do some kind of sharding. But Cassandra is very scalable.  

**Design Deep Dive**  
Analytics can be built in to this, to look for geography, IP address, etc by storing extra data.  
Asynchronously can be written to Kafka.  

We can aggregate the data at the machine and then send the data to Kafka in batches.  

****************************************************************************

**Design a database for a tiny URL implementation.**

We'll need to design the data schema, database architecture, and choose a database technology to use.  
  
Do we need to specify a particular brand or instance of database engine, or can we specify the technology type, e.g. relational, key-value, document, etc?”  
  
How many users are we catering for?  
Or in other words, how many URLs would we be generating each second, and how many URLs would we be serving each second?  
I'm assuming that it would be highly asymmetric, in that there would be many more URLs served each second than URLs generated?”  
  
It would be used as the default URL shortener for a popular social network, which has    
300 million daily active users    
  
Let's say 20%, or 60 million, users are frequent original posters (assuming that most social users consume or share existing posts). If they post something with a URL, say, once per day, then we would need to create 60 million URLs per day. 60 million / (24 hours * 3600 seconds in an hour) = 695 URLs created each second, if equally spaced across the day.”   


For URLs served, I'm assuming there would be a spread of popularity. Some shortened URLs could have zero clicks, while a few may go viral and get tens or hundreds of thousands, or even millions of clicks.”  

Let's assume that on average a URL would be read maybe 100 times, heavily weighted to the most popular URL. So we can say there are roughly 100 times more reads per second than writes.”  

“Let's increase it by an order of magnitude, say, 1000 times?”

We will be generating a lot of URLs! Would we need to keep them forever, or should they expire after some time?”

We'll keep track of when a URL is created, so we can remove it

We’ll need multiple API or web servers, and we’ll also need multiple database servers, to create a distributed system. This way we’ll be able to meet the high demand and availability requirements.

We’ve established that a few of the links will get the most traffic. This seems like a good candidate for a caching layer, so we can reduce the load on our databases and make the most-used links quicker to access.

**Data schema:**

At the minimum, we’ll be storing the full URL, so we can redirect to it, and the short code, which must be unique per URL. Since we also look up links by the short code, it could also be the primary key or identifier for the record.”

Earlier we also mentioned that links should expire, so we’d need to store a timestamp. Perhaps either the date the link should expire, or the date it was created.”

If we store the date created, we’ll be able to change expiry policy parameters more easily, and enable more flexibility there. It would also be simple to report on tiny URLs created per time frame, without back calculating. If we store the expiry date though, it means we won’t need to calculate it from the created date, and that could speed up purging old records. We could store both as well.

We could also store the user that created the link, if the service records that. Do we have a user ID to store, or would it be anonymous.

LinkRecord:
shortCode
Full URL
createdDate
ExpiryDate
userID

Collisions - we will be running multiple servers, or partitions, and if each server is creating an ID, we will be bound to get collisions.

Ee'll need some other scheme to come up with an ID for the ‘shortCode’. We could make a central, shared ID generator that each server can call when creating a new short code. That way we can guarantee that the generated short code will be unique.”



Load Balancer —>. App servers 1 , 2, 3 —>Cache for reads ——> db server shard 1,2,3  
					|  
				ID generator  

Problem could be that since there is only one ID generator, the speed and throughput of the system is limited by it. It would have to be very fast if it were to manage the amount of traffic, particularly in peak times. Also it’s a single point of failure.  
  
We could make a few more ID generators, and the API servers could ‘round robin’ between them to get a new number. Each of the servers would be able to serve from an assigned range of numbers, so that they don't return the same numbers. So ID generator 1 would serve from say 0-50 billion, generator 2 from (50 billion +1) to 100 billion, and so on. Then if one ID generator goes down, the servers can still create short codes, just from a different range.  

**Design Deep Dive**  
——————————
  
If we extend the concept of **predefined ranges**, instead of returning a single ID, we could return a list of numbers to the application servers. That way things are sped up, as the application servers do not need to call the ID generators on each call, making it even more reliable. They can assign a short code from the list of numbers, and only when they run out do they need to call one of the ID generator servers again.”
  
Systems like **ZooKeeper** that I understand can manage this distributed key-generating scheme. Then we can use existing, reliable software for this part, instead of writing it from scratch, especially since it is a core function of the system.”
  
We’ve discussed that the **ID generators** would generate numbers within ranges, in other words generate integers. If we simply return the integer number, it soon could become quite a large number. Our tiny URL would soon become a very long URL, the more the service is used. If we re-encoded that ID as a number in a number system that uses letters and numbers and is also case sensitive, which could be a base62 representation, then we would be able to constrain the length of the short code to be, well, short
  
So after a year of generating short codes, we'd be at ID ‘60million*365 = 21 900 000 000’, which is 11 characters long. But if we represented it in a higher base, like base62, it would be something like ‘nU6aqs’, only 6 characters, making it a lot shorter.
  
Since the schema is quite simple, and the link records are independent (i.e. they’re not related to each other), we are not constrained to traditional relational databases. Various NoSQL solutions have good support for sharding and horizontal scaling. I'm wondering whether a key-value store or a document-oriented database would be the better option.
  
It depends on how much information we are storing for each link, and how that information is used. In a **key-value store**, it is not usually straightforward to query on anything other than the key. We have a few other fields that might be useful to query on, especially the userId and expiry dates. Then we should go with a **document store**.

