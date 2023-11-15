# Design Youtube  


Upload - Core functionality  
Watch - Core functionality  

Comment 
Bot prevention
Search
Recommendation
Analytics
Advertising

But with a lot of things to focus it will not be possible to go in all directions.


Lets for now for simplicity we will work only for the Upload and Watch


Functional Req:
—————————
Upload and Watch


Non Functional Req:
———————————

Reliability - Videos should not disappear
Scalability - 1 B DAU , each user watching 5/day . 1 B * 5 = 5Billion video being watched per day
		   - Upload ratio 100 watch : 1 upload (1/100)
		- 50 M video upload if we consider - because 1/100 = 1% of 5Billion videos uploaded
   		- Most of the 50M videos being uploaded not all will get a lot of views. Say about 5% videos get about 90% of views.
		- But lets assume most videos do not get that much views.

We need to favour Availability > Consistency 
I.e if your home page is populated with a bunch of videos then every time you refresh the page you should get the correct set of videos. You should get an HTTP 200 response and things should load. 

When we sacrifice consistency then this is what we are compromising: Suppose someone just uploaded a video, and when you refresh your home page, but you get a small latency or delay in getting the newly uploaded video. 
This can happen if we have multiple storage system and the web service is reading from a read replica which is not yet updated with the new content yet. So you may possibly have to wait like 1s or 5s or say 10s for a new video to be loaded on your page.

Latency: When we click a video it should immediately start playing without any delays. Possibly avoid buffering if the n/w bandwidth is good.



High level design:
——————————

Start with designing for Upload

Dealing with massive amounts of data 50M uploads per day.

							——>App Server
So User ——>. Load Balancer 	—> App Server
							—>  App Server

So in this also we can talk about few things like how the Load balancer will distribute the data among different app servers etc.
But for now if we leave that out then still there are many other things to worry about.

When User is uploading a big video about 1GB file and a small n/w disruption happens then should we retry again or start where we left off etc.. But we are leaving all those details also out of the picture for now.

So once the video is uploaded, since the data is static it’s good to store the data in some Object Storage (raw). One advantage for using google cloud storage or S3 is that they handle the replication for us so we can assume that the data will not be deleted and will be reliable stored. So Reliability is covered. 


API ->. upload(title, description, video, uid)
We can store other metadata and additional data like tags etc. but lets stick to the minimal now. 

But we need the user information of the user who uploads the data and that data we will store in the NoSQL (store metadata about the video)
In this metadata we can store the reference to the video object stored in the object storage.

We can use MongoDB which stores data in JSON format. We can have one collection (table) for Video and another for User. But since data is denormalized in MoongoDB and other No-SQL dbs then we can have duplication of data. So the video collection can also have details about the users. 

Profile picture needs to be stored also, so we will store this in the object storage also. The Profile picture should have a reference in the user info. So we have lots of duplicate user info but it’s ok because we will get a faster response as we don’t have to perform joins.

If the user updates the profile pic then the information has to be updated in multiple places because of data duplication. So if we have a user who uploaded 1000 videos then all that has to be updated with the new information about the new profile pic.

But we will accept this tradeoff as we will assume that a user does not update profile pic so often the most traffic is around upload and watch data.
We will also do the update asynchronous as it’s ok if there is a delay in updating the profile pic.

Youtube actually does encoding and compression of the videos when uploading to storage. This process can take a lot of time hence we need to do this asynchronously and for that lets introduce a message queue. From this message queue they will be put to another Encoding service which will do the encoding. 

The encoded video files can be then stored in another object storage, so 2 object storage one for raw video files and another for encoded. Users will be reading from the encoded video storage.

We can place a CDN for the encoded storage to make fast fetching. For the metadata stored in MongoDB we can make that access faster by using another in-memory cache for the metadata. This can be an LRU cache which will have the most recently uploaded videos metadata. 



Design Deep Dive


Now we can see if we can scale the encoding portion. Since encoding will take a lot of time so if we avg that time to 1min and there are 50M uploads per day.  Which is 50M / 100000 seconds in a day = 500 videos /second


So should we have 500 worker nodes for the encoding probably more so if there are 500 videos uploaded in a second then if there are 500 worker node and then each worker nodes take 1 minute to encode then while they are doing the encoding every second 500 more requests are piling up in the message queue and that will fill up the queue pretty fast. After 1 minute all the 500 worker nodes are free and they can now process next 500 but still new requests keep coming.

So ideally we should have 500 X 60 sec = 30K worker nodes… Bu may be one worker node can encode more than 1 file at a time.

Now let’s look at the Watch (watching video) part.

Ideally the videos are not streamed continuously but are loaded in chunks as HTTP request. And audio and video are fetched as separate request. 

Loading videos using smaller chunks can lower latency. 1 MB of data is easier to transfer than the whole video.
Video streaming must happen in chunks and the download can happen as a full file. The streaming in chunks happen and then that data is stored in memory so we don’t want too much of our memory to be used for storing these chunks of data.

UDP vs TCP
UDP is good for live streaming, if you miss 1 sec of data then we don’t want to go back and watch that lost part but rather get the latest info about the live match for ex. 
But for watching a video we don’t want to miss some 2sec of that data. So TCP is favoured for reliability.

So here we send HTTP request over TCP.

The upload can be rate limited - so no one user is uploading too many videos. This can be done at the load balancer.

We can do some recommendation service which analyses the data.

We can also talk about adding read replicas if we are going for SQL db and also sharding in that case.





