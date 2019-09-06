---
layout: post
category : web
author: Lucas Natraj & Chase Jenkins
tags: [REST, tutorial]
title: Creating REST APIs
---

Regardless of what underlying technologies you choose for persistence, your API is what the world sees, and getting it right should be paramount. Here are a few observations we've made about REST APIs and trying to get them right.


1. **Nouns for CRUD actions**  
Nouns work well for handling typical the typical [CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete) actions. Verb guidelines:  
    ```
    POST - Create new resource (where the server will assign the 'location').
    GET - Fetch / read a resource (idempotent).
    PUT - Create / Replace resource in entirety (idempotent) (where the client determines the 'location' of the new / updated resource).
    PATCH - Partially update a resource. This is sometimes implemented as a set of transformations to be made.
    DELETE - Remove (as in 'forever'! No longer GET-able)
    ```
Google exposes the traditional CRUD stuff for their Calendar, Contacts, etc... APIs  
    ```
    Create new calendar event -
    POST  /calendars/calendarId/events
    ```

1.  **GET requests generally should NOT include a body**
    Although the specification does not explicitly forbid it, the general consensus is that GET requests should not include any payload in the body.  
    Postman (a Chrome extension) which internally uses Chrome's http libraries do NOT even allow a body to be provided in a GET request.
    
    Elasticsearch's query api supports a body in the GET request.
    "Both HTTP GET and HTTP POST can be used to execute search with body. Since not all clients support GET with body, POST is allowed as well."
    ```bash
    $ curl -XGET 'http://localhost:9200/twitter/tweet/_search' -d '{
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }'
    ```

1. **Return something useful from POST, PATCH & PUT requests**
    Rather than a simple status code, consider returning a more useful value like the Id of the new object, version, or the updated entity itself.

1. **What about non-CRUD actions (operations)?**  
    * Option 1 — Just POST to the operation. (The most common approach)  

    ```
    Google api to send mail -
    POST https://.../gmail/v1/users/:userId/drafts/send
    {
        id: draftId
    }
    ```
    * Option 2 — Convert the action into a request (resource) for an action (i.e. noun-ify the verb)  

    ```
    Example -
    POST https://.../gmail/v1/users/:userId/drafts/sendMailRequests
    {
        id: draftId
    }
    ```

1. **Caution around DELETE**  
    Delete semantics can be complicated. Should DELETE truly remove the resource or does it merely mark it? (for example moving to trash). If there is a desire to ever undelete it note that there is no UNDELETE HTTP verb.  

    Twitter has NO DELETE endpoints and instead tends to favor POST .../destroy/:id endpoints. Unsure whether these are actually deleted. There is currently no support for un-destroy.

    Pseudo deleting (where undeleting is also required) an object can more easily be achieved using PATCH semantics (rather than DELETE)

1. **Filter / Search Endpoints**  
    Filtering is commonly supported on GET requests on a collection to return only the results that match the filter: `GET /shows?ticket_status=available`
    
    Comparison Operations can be supported using certain keywords like `eq`, `not`, `like`, `gt`: `GET /api/v1/products?price=gt:5.00`

    Full text search is commonly supported via a `'q'` parameter  
    ```
    Twitter (Tweets full text search)  
    GET https://api.twitter.com/1.1/search/tweets.json?q=%23funny
    ```
    ```
    Azure (Search Documents)      
    GET https://.../indexes/[index name]/docs?[query parameters]

    POST https://.../indexes/[index name]/docs?[query parameters]
    Content-Type: application/json
    ```

    GET vs POST
    * Filtering is commonly done as a GET request (on the desired collection) directly in the URL.  
    * Filtering (with search) within a specific collection can be achieved using a GET request and the 'q' parameter.  
    * General searching is commonly on a higher level `/search/` endpoint. (see Azure example above)
    * Queries can be done BOTH as a GET request (URL only) or as a POST (with the query in the body). There are usually (application specific?) restrictions on the size of URLs (~8KB) so for longer queries use POST + json body.

1. Singular vs Plural
    Keep it simple and always use plural for collections.  
    `GET .../entities/:entityId`

1. **Version the API**  
    Major versions are commonly found in the url.
    ```
    POST https://.../gmail/v1/users/:userId/drafts/send
    ```
    Stripe uses enables the invocation of specific minor versions via a Header.
    ```
    curl https://api.stripe.com/v1/charges -H "Stripe-Version: 2016-03-07"
    ```
    Supporting older versions while introducing new ones enables a smoother transition / migration for consuming applications. Published versions of services should not be changed, merely superseded by new versions and eventually retired. This is assisted by a microservices architecture.

1. **Differentiate API endpoint from application / web endpoints.**  
    ```
    Api:
        https://www.example.com/api/v1/...
        https://api.example.com/v1/...

    Web:
        https://www.example.com/index.html
    ```

1. **Immutability of persisted objects gives several benefits**  
    - Caching — GET requests for a specific version of an object is idempotent and can be cached at various layers
    - Reproducibility — By ensuring that data used as inputs to a computation can never be modified, the computations can be re-executed and guarantee identical results.
    - Concurrency — Reading specific versions is safe without the need for locking. Writing requires either optimistic concurrency.
    - Traceability — Modification history is never lost.
    - Performance — Many data stores are optimized to handle inserts better than updates.  

    Exposing entity versions allows clients to obtain a specific snapshot of the entity.

1. **Entity versioning strategies**  
    Entity versioning may seem odd to talk about in a REST API article, but the versioning strategies are incredibly relevant to the kinds of endpoints your API exposes, and they should be considered carefully.
    * **Time** — Relationships between entities do not specify particular versions but solely the id of the related object. They are effectively versioned by the entity that contains them (by for example a timestamp on the entity). This eliminates the need to bubble version changes across relationships when an entity is modified.
    A caveat of this is that traversing relationships in an eventually consistent database may lead to an incorrect old version of an entity (until the changes have fully propagated).
    * **Guids** — Relationships between entities explicitly specify the id and version of the related object.
    Version ordering becomes difficult as there is no logical sort on the version (which is a guid).
    This also requires a 'bubbling' of versions across the relationship graph as entities are modified (which may be tricky when cycles exist).
    In an eventually consistent database, there is no chance of traversing a relationship to an incorrect entity. It would purely be 'missing' until the changes have propagated.
    * **Numeric** — Similar to the Guid approach, but utilizing a monotonically increasing number (per entity). [Note that if the number was monotonically increasing across the entire database, it would effectively function as a timestamp :)]
    This also requires version bubbling, but allows a more easier way to sort the version history of entity.

    Idempotency can be tricky in conjunction with versioning because:
      * A PUT would always create a new versions - potentially breaks idempotency in 2 ways:
          - If using optimistic concurrency the first PUT will succeed while the second will fail due to not being at latest.
          - If not using optimistic concurrency each PUT will insert a new version of an object (unless the backend detects no changes)
      * GET latest version of an object can return different values if a new version has been added.  
   
1. **Return appropriate status codes**
    ```
    200 (ok) - All good
    202 (accepted) - Request acknowledged but may not yet have been performed
    400 (bad request) - arguments are invalid
    403 (forbidden) - Unauthorized to access resource / endpoint
    404 (not found) - requested resource does not exist
    409 (conflict) - if requested change conflicts with another change
    500 (internal error) - Something bad happened (Make sure to NOT leak this to the front end. Log instead)
    ```

1. **Define a consumable error payload**
    In addition to returning an appropriate status code, also provide information that indicates what went wrong with the request.
    For example:
    * A 400 (Bad Request) Response could also include which request parameters were incorrect.
    * A 401 (Unauthorized) Response could detail if the user simply failed to provide credentials or if their permissions are inadequate.
    * A 409 (Conflict) Response could indicate why the conflict occurred, such as noting that a uniqueness constraint is being violated for an entity key.
    
    An example error response body might include the status code, a user-readable message, and a reason/status type:
    ```json
    {
        "error": {
            "code": 401,
            "message": "The request does not have valid authentication credentials.",
            "reason": "unauthenticated"
        }
    }
    ```

1. **Use 'fields' to allow clients to fetch only their desired properties**  
    Also known as 'partial response' or 'projection', this enables more efficient, performant queries to be made.
    ```
    Google
    https://www.googleapis.com/demo/v1?fields=kind,items(title,characteristics/length)

    200 OK
    {
        "kind": "demo",
        "items": [
        {
            "title": "First title",
            "characteristics": {
            "length": "short"
            }
        },
        {
            "title": "Second title",
            "characteristics": {
            "length": "long"
            }
        },
    ...
    ]

    Facebook
    /joe.smith/friends?fields=id,name,picture

    LinkedIn (a little complicated)
    .../people:(id,first-name,last-name)
    ```
    It can be useful to sometimes support a 'fields_mode' parameter with possible values of 'include' (default) or 'exclude' to support the explicit exclusion of properties.

1. **Support Pagination of Results with limit and offset query parameters**
    The keywords ‘limit’ and ‘offset’ (or 'skip') are commonly supported as query parameters to allow clients to request specific portions of a query result dataset.  
    - Google use offset and limit
    - Facebook uses offset and limit
    - Twitter uses page and rpp (records per page)
    - LinkedIn uses start and count

    ```
    LinkedIn    
    GET https://api.linkedin.com/v1/companies/1337/updates?start=20&count=10&format=json

    Response
    {
        "_count": 10,
        "_start": 20,
        "_total": 613,
        "values":  [
            …
        ]
    }
    ```

1. **SSL everywhere, no exceptions**

1. **HATEOAS can be useful, but may not be worth it**  
    ["When designing a hypermedia API, you're really designing for a client that does not, and will never, exist." - Jeff Knupp](https://jeffknupp.com/blog/2014/06/03/why-i-hate-hateoas/).  
    Note this can be very useful in some cases.
    ```
    e.g. Pagination in Django
    Request:
    GET https://api.example.org/accounts/?page=4

    Response:
    HTTP 200 OK
    {
      "count": 1023
      "next": "https://api.example.org/accounts/?page=5",
      "previous": "https://api.example.org/accounts/?page=3",
      "results": [...]
    }
    ```

1. **Use JSON where possible, XML only if you must**  
    JSON is more compact and readable, and certainly more easily consumed by the ubiquitous javascript
    You *should* use camelCase with JSON, but snake_case is 20% easier to read
    Consider using JSON for POST, PUT and PATCH request bodies
    It is possible to introduce specific 'format' parameters to satisfy different consumers.

    LinkedIn  
    `https://api.linkedin.com/v1/people/~`
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <person>
        <id>1R2RtA</id>
        <first-name>Frodo</first-name>
        <last-name>Baggins</last-name>
        <headline>Jewelery Repossession in Middle Earth</headline>
        <site-standard-profile-request>
            <url>https://www.linkedin.com/profile/view?id=…</url>
        </site-standard-profile-request>
    </person>
    ```
    `https://api.linkedin.com/v1/people/~?format=json`
    ```json
    {
        "firstName": "Frodo",
        "headline": "Jewelery Repossession in Middle Earth",
        "id": "1R2RtA",
        "lastName": "Baggins",
        "siteStandardProfileRequest": {
            "url": "https://www.linkedin.com/profile/view?id=…"
        }
    }
    ```

1. **Response Envelopes**  
    Using data envelopes allows clients of the API to treat all responses (including errors) in a more uniform manner.
    It also ensures that client-specific parameters do not conflict with the required api parameters

    Sample Response
    ```json
    {
        "id": "2e64d359-853c-4f68-b4e9-bb6073e5e0bf",
        "type": "PolylineSet",
        "data": {
        	"Name": "triangle",
        	"Extensions": [{
        	    "Id" : "Petrel",
            	"Attributes" : {
                	    "TimeCreated": "2015-10-21T11:33:10.8977529-07:00",
                	    "ObjectId": "1:4b250188-7777-4942-a2c4-9699a1309b17:://1d9a2dd1-dd1d-4676-a92e-6057e64d33c2/4b250188-7777-4942-a2c4-9699a1309b17"
                	}
            	}],
        	"Polylines": [
        		{
        			"X": [
        				100.0,
        				-49.999999999999979,
        				-50.000000000000043
        			],
        			"Y": [
        				0.0,
        				86.602540378443877,
        				-86.602540378443834
        			],
        			"Z": [
        				0.0,
        				0.0,
        				0.0
        			],
        			"IsClosed": true
        		}
        	]
        }
    }
    ```

1. **Include response headers that facilitate caching**
    ETags (entity tags) should be returned in response headers to GET requests to facilitate caching by callers. A response with an ETag might look like this:
    ```
    HTTP/1.1 200 OK
    Access-Control-Allow-Origin: *
    Cache-Control: private, no-cache, no-store, must-revalidate
    Content-Type: text/javascript; charset=UTF-8
    ETag: "7776cdb01f44354af8bfa4db0c56eebcb1378975"
    ...other headers and content...
    ```
    
    A follow-up GET request to the same resource can then include that ETag in a header, such as `If-None-Match`. If the resource hasn't changed, the server should respond with `304 Not Modified`, otherwise, it should return the modified resource with a new ETag.
    
    A good way to generate an ETag for a resource might be to generate an MD5 hash of the resource. If immutable entity versioning is being used, the version number (or hash thereof) may also be a viable candidate for the ETag for a specific entity.

1. **URL-encode path and query string parameters**  
    To ensure special characters don't confuse your API handlers, ensure that you URL-encode path and query string parameters. For example, `bob@example.com` would become `bob%40example.com`, and `a/b` would become `a%2Fb`.
    
1. **Document your API**  
    Two good examples of API documentation are [Google APIs Explorer](https://developers.google.com/apis-explorer/#p/) and [Swagger](http://swagger.io). These tools:
    - list the methods your API exposes
    - document the parameters and give example request bodies
    - provide a way to interact with the API from a web-based UI
    
    Note that the Swagger Specification scheme has been donated to the Open API Initiative and renamed the [OpenAPI Specification](https://github.com/OAI/OpenAPI-Specification) as of 2016.
    
1. **Secure your API**  
    Authentication and authorization is almost certainly a must. Use OAuth2 tokens, JSON web tokens, or use API keys for service-to-service communication when service accounts are not an option.
    
    Using token-based authentication, such as JSON web tokens (JWT):
    - can help preserve server statelessness
    - faster to process (no need to hit a database)
    - allows 3rd party providers for authentication


## References

* [Architectural Styles and
the Design of Network-based Software Architectures](http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)
* [Twitter's REST API](https://dev.twitter.com/rest/public)
* [Google's Application APIs](https://developers.google.com/google-apps/products)
* [Apigee Technology Blog](http://apigee.com/about/search/gss/RESTful%20design)
* [Azure Search Service](https://msdn.microsoft.com/en-us/library/azure/dn798935.aspx)
* [Facebook Documentation](https://developers.facebook.com/docs/)
* [Elasticsearch Request Body Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html)


#### TODO

* Several Web Frameworks will support Parameter Binding from either the query or the Body (json)
* Create a Query resource!
* Actions (specifically that perform background work) can be exposed via request resources. Can help with tracking / audit / status / progress
