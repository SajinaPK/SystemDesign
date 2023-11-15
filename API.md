# API

**REST = Representation State Transfer**

Other options for REST are graphQL(Facebook ) and gRPC(google)
graphQL - Instead of multiple request, just a single request (or query) with the details about all the resources you need to fetch. So you don’t over-fetch details you don’t need.
gRPC -  Considered a framework. gRPC for web - performance boost comes from protocol buffers (vs JSON in REST) where data is serialized into binary format which makes it more storage efficient and network efficient too due to less data over network. 
Websockets - In a chat like system, with HTTP we will have to implement polling to look for new messages. Websockets is a bi-directional communication. Messages are pushed to the device immediately. 

JSON is more human readable data.

Rest is not a protocol or spec just a loose set of rules that’s agreed to use for communicating between web applications.
	- Uniform Interface
	- Client-Server
	- Stateless
	- Cacheable
	- Layered System
	- Code on Demand (optional)

Ex: Twill, google maps, Stripe

URI - Uniform resource identifier

Resources should be grouped by nouns and not verbs:
	https://example.com/api/v3/products  - > correct
	https://example.com/api/v3/getAllProducts   - > incorrect


Verb -> resource -> http

POST - Create.  [Not Idempotent] GET	-  Read. [Idempotent] PUT - Update.  [Idempotent] DELETE - Delete. [Idempotent]

[CRUD]

HTTP status code: 200 level -> Success
				400 level -> Something wrong with our request 
				500 level -> Something wrong at the server level

Idempotent - In the context of REST APIs, when making multiple identical requests has the same effect as making a single request – then that REST API is called idempotent.
Ex DELETE /item/last -> calling operation N times will delete N resources – hence DELETE is not idempotent in this case. In this case, a good suggestion might be to change the above API to POST – because POST is not idempotent.
POST /item/last

A rest implementation should be stateless. Client and server do not keep any states about each other. All request response are independent of each other. This makes them well behaved and easy to scale.

Pagination
——————

When a response from the REST API would include many results, GitHub will paginate the results and return a subset of the results. For example, GET /repos/octocat/Spoon-Knife/issues will only return 30 issues from the octocat/Spoon-Knife repository even though the repository includes over 1600 open issues. This makes the response easier to handle for servers and for people.

You can use the link header from the response to request additional pages of data. If an endpoint supports the per_page query parameter, you can control how many results are returned on a page.


Versioning
——————

APIs only need to be up-versioned when a breaking change is made.
Breaking changes include:
	- a change in the format of the response data for one or more calls
	- a change in the request or response type (i.e. changing an integer to a float) 
	- removing any part of the API.

Breaking changes should always result in a change to the major version number for an API or content response type.
Non-breaking changes, such as adding new endpoints or new response parameters, do not require a change to the major version number.

