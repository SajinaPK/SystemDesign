# Design Twitter


**Functional Req:**  
—————————
  
1. Follow others  
2. Create Tweets  
3. View Feed (of people we follow)   
4. Feeds can be 140 char + images + videos  
  
**Non Functional Req:**  
———————————

User -> 500 M
DAU -> 200M -> each reads about 100 tweets a day -> 200M *100 =20B tweets read in a day
		size of each tweet -> 140 chars roughly 140B plus additional data like user info and other. So let’s assume the whole size to be 1KB. pLus assuming images +some short videos lets average this to 1MB.

20B users with each tweet as 1 MB is a lot of data
If each tweet is 1 byte then the total data for 20B users is 20GB so multiply by 1MB makes it 20 TB X100= 20PB

20 PB of data needed per day.

Read heavy systems

For the write lets assume about 50M as most people are not going to tweet everyday. 50M * 1Byte = 50MB * 1MB = 50 TB
Much less write than reads

Some people have massive amount of followers like 100 M



Client. ——> App servers  [we can have load balancers to scale this]

We can have relational db if there are too many relations between data. Like Followers and followeee
We can also store the data in No SQL and then have a graphDB (adjacency list graph with incoming nodes as followers and outgoing as followee)to store relations an


Because of the massive amount of reads we will need to have some kind of cache in between. So app servers will hit the cache first before hitting the DB.

For Media and videos we will have some S3 like object storage. 
When the app servers read tweets from the DB it will check if there are images and then if there are then fetch them from the object storage.
Also because the images like profile pic, other images, videos etc are static files we can also deliver them using CDN.
With CDN in place applications need not interact with the object storage, App server will give the url of the image to the user and then client will make a separate request which will then hit a CDN which is tied to the object storage. [We should use a pull based logic to pull data from the object storage to the CDN	]



High level design:
——————————

createTweet(text, media, uid)
getFeed(uid)
follow(uid, user_to_follow)

The UID and other metadata like timestamps etc can be embedded in the HTTP headers too. 


At the DB we need 2 tables 1. Tweets		2. Follow

In the follow we will index based on follower. So all the person that one person follow in one column

Tweet -> tweet_id, timestamp, uid, text, media(just the reference to the media, actual will be stored in the object storage )

If we are storing only the tweet text supposing 1KB size then for 50M tweets per day that’s 50M * 1K = 50GB of data per day
In a month that will be 50GB * 30 = 1.5TB and in a year about 18 TB of data. That’s a lots of data to store.


Since lots of reads are happening we will create read replicas are good to reduce bottlenecks. With read replicas its still ok to have one leader and other worker/slaves where the leader will update the replicas and then if the user fetches a tweet and gets a stale data or delay of 5sec its ok for a twitter design.

If 50M tweets per day then diving by 10000 will give 500 writes per second [seconds in a day 86400 approx to 100000]

Peak writes could be 10 times this amount too in a second.

So we should scale our writes too. Sharding -> based on uid . So then it’s easier to locate the shard which contains a particular users follower list and then fetch the tweet from there.
However sharing based on twitter_id will be not a good way as the user will not know which replica to fetch the data from. 


Most popular tweets will be stored in the cache for easier access.
LRU is a good way to cache the most relevant tweets in the cache. 

But it can happen that one user needs 20 tweet and 19 of them are popular and are available in the cache but the 1 tweet is to be fetched from the DB and then if there is a delay in fetching that then that’s not a good experience. So the problem here is Latency and not scalability .

We can reduce the number of shards and reduce some latency.

Another thing to reduce latency could be to introduce some asynchronous messaging using pub-sub engine which will populate another cache whenever a new tweet is created.
And now all the new tweets will be fetched from this cache and very rarely the DB will be hit.
Specially useful for the 200M out of the total 500M users who are usually active.
The feed cache we thus created will reduce latency. For scalability we can shard this cache too., 

But again for a very popular person with 100M users could put a lot of stress on the feed cache. But for such cases we can come up with other solutions with new caches.




