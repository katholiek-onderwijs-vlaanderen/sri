# Standardized ROA Interface Specification

## Introduction
This specification describes a set of requirements for implementing RESTful interfaces. An interface that adheres to this specification qualifies as an SRI interface. In short SRI defines a number of things about a RESTful interface :

- The api must support regular and list resources
- The api uses keys and permalinks in a certain way
- The api exposes meta-information
- List resource all support a set of common URL query parameters
- Batch operations are supported to to enforce atomic updates/creates of multiple resources
- All regular resources have a JSON schema associated
- etc...

RESTful web services follow the general architecture used to build the world wide web. This architectural style is described in [Roy Fielding’s dissertation][roy-fielding].

## Motivation
One of the major goals of any software architecture, be it at application- or system-level, is to achieve modularity. A system’s components should, to a certain degree, be separated and easy to recombine into new functionality.

An SRI interface should be uniform. It should, in other words, be designed without a specific use case, or client application in mind. It defines specific restrictions on how applications/modules can integrate with each other. 

## Representation of resources
We make a distinction between *list* resources and *regular* resources. *List* resources represent queries, and contain an array with many result objects. *Regular* resources correspond to single resources in a system.

All resources are be represented as [JSON documents][json-rfc]. 

URLs in resources must always be relative. By relative we mean the URL does not include protocol, host nor port. It *should* include a full path. For example a school is available on `/schools/{uuid}`, a mammal on `/mammals/{uuid}`, a person on `/persons/{uuid}`. A *list* of schools on `/schools`, a list of persons on `/persons`.

### Regular Resources
All *regular* resource MUST available on a *permalink*. *Permalinks* are of the format `/{type}/{uuid}`. Server implementations are RECOMMENDED to implement more human readable URLs (aliases). A person is OPTIONALLY also available on `/persons/john.doe`. Links between the resources MUST always use *permalinks*. Aliases MUST only provided for convenience during development.

When defining the resources in an SRI API, a designer should also consider security constraints. Security MUST be applied per URL, so the API MUST split resources along security boundaries. For example a person’s public information is a separate resources from his private information.

Clients are advised to strive to use the [“Hypermedia as the Engine of Application State”][hateoas] principle as much as possible. 

Clients can safely store *permalinks*. The server MUST, in other words, support these URLs over a long period of time. 

The `expand` parameter can be used on *regular* resources to perform inlining of referenced resources. The client can specify dot-separated paths (relative to the JSON document), that need to be included. The server MUST define which paths are supported for expansion. The server is RECOMMENDED to support a consistent schema for expansion, such as all direct references in a resource. All references to other resources MSUT embedded in a JSON object, with a key `href`. When executing expansion the server MUST add an extra key `$$expanded` to that object, including the full *regular* resource that was expanded.

Arrays in the JSON responses SHOULD BE named in plural nouns. i.e. schools, rather than school for an array of JSON objects about schools. 

Keys in the JSON responses SHOULD BE camelCasedLikeThis. Acronyms SHOULD NOT be in all capitals. For example the ZIP code is not implemented as ZIPcode, but rather zipCode.

For *regular* resources all JSON objects in the document, including the root object, MUST expose `key` that contains a UUID to uniquely identify separate sections in the JSON document. For the root object this key MUST correspond to the UUID section of the *permalink*. This also applies to arrays of objects within the document.

A regular resource should include a `$$meta` section. Such a section MUST include :

- a permalink to itself, under `permalink`.
- a link to the json schema of this resource, under `schema`

It MAY also include an array of supported aliases for this resource, under `aliases`.

Example (notice the `expand` URL parameter to include a related resource):

  GET http://api.vsko.be/schools/006613?expand=schoolLocations
  {
    $$meta: {
      permalink: ‘/schools/{UUID}’,
      schema: '/schools/schema',
      aliases: [‘/schools/006613’]
    }
    institutionNumber : “006613”
    .. ..
    schoolLocation : {
      href: ‘/schoollocations/{uuid},
      $$expanded: {
        $meta: {
          permalink: ‘/schoollocations/{uuid}’,
          expansion: ‘SUMMARY’
        }
        .. a selection of key’s from the full schoollocation ..
      }
  }

The API implementation are strongly advised to implement GZIP compression.

Responses should specify caching headers (Via HTTP headers). Both HTTP 1.1 header Cache-Control and HTTP 1.0 Last-Modified should be used to specify a sensible caching policy. It is perfectly valid to have resources that are not cacheable. In that case you should pay extra attention to the performance of your implementation, and specify no-store for Cache-Control.

Server responses for individual resources should remain smaller than 10 kilobyte (after compression), to ensure quick responsiveness in the user interface. This will require some large resource to be split into smaller resource (both resources should link to each other, as described by the connectedness principle of REST).

Server responses for list resources should remain smaller than 100 kilobyte (after compression). 

As the interfaces must be designed to support a wide variety of use-cases, they must not be designed with a specific client in mind. Therefore if resources A contains links to resource B, and the relationship makes sense to be queries in both directions, resource B should also link to resource A. Any update to one side of the relation will be reflected in the other side after a reasonable time period where all cached version are updated.

Time-date fields, URLs, address information, email, etc.. should be standardized across all API implementations as much as possible.

Date, time, email and URLs should follow the "format" definitions of json schema validation
(http://json-schema.org/latest/json-schema-validation.html):
Time-date fields: RFC 3339, section 5.6 (http://tools.ietf.org/html/rfc3339#section-5.6)
email: RFC 5322, section 3.4.1 (http://tools.ietf.org/html/rfc5322#section-3.4.1)
URLs: RFC3986 (http://tools.ietf.org/html/rfc3986)

Country codes should adhere to this widely used ISO standard : http://en.wikipedia.org/wiki/ISO_3166-1_alpha-2

All resources should be served with both HTTP headers ETag and Expires to allow for conditional gets (Expires can be set to 0, to disable caching on HTTP 1.0 proxies).

Every regular resource should have a JSON schema definition. This schema should be exposed as /{resource name}/schema.

Keys in the resources should be primarily in English (unless a term can not be translated properly from the native language).

Constants should be included in English (and capitals), they should be clear terms that at least facilitate debugging / exploring the API. Clients should translate the constant value in their view for display to the end user.

And last, but not least: extra care should be given to make sure you implementations are efficient, handling requests quickly.

List resources should respond within 100 ms. Regular resources should respond within 10 ms.

A resource should include a $$meta section for adding meta-information about the resource. Examples of meta-information for a list resource would be the link to the next page, the total number of rows, etc.. For a regular resource the $$meta section must include a permalink.

List resources
Every sensible data-access path on a resource is exposed on a specific URL. The result of requesting such a (query) URL is a list resource.

List resources can be filtered through the use of URL parameters.

Resources are allowed to support URL parameters for expanding a resource. Expansion makes a resource include related information in a single response. All expanded resource are included inside a $$expanded tag, and can be filtered out by the server. Allowing a user to put an expanded resource. If a resource is expanded, the expanded resource should include $$meta.expansion with the expansion level.

List resources should also support the expand parameter. The dot-separated paths are relative to the individual elements in the results array. So ‘href’ would perform inlining of the regular resources in the list. In the case of expansion you must take care that you check security on the documents that you include.

List resource must be found on the base URL of the corresponding regular resources. If, for example a school can be found at http://api.vsko.be/schools/128256, then a list of schools can be queried on http://api.vsko.be/schools. These list resource should be found on URLs that end in the plural form of a noun. i.e.: http://api.vsko.be/school, should not be used.

List resource could be used to retrieve a full list of resources (i.e.: all people), but become most useful if URL parameters are added to filter and order the list of resources retrieved. For example http://api.vsko.be/schools?diocese=brugge. This would only return schools belonging to the diocese brugge.

All parameter names and parameter values should be camelCasedLikeThis, but the resource names should be lower cased.

If the parameter name or the value is not recognized by the server or not valid for this resource a developer-friendly error message should be returned.

List resource must all support the following standard URL parameters by default :

modifiedSince : A field that allows the user to implement a publish-subscribe style. Periodically an application can poll such a list resource, to determine which resources were changed since it last asked the service. Allowing for local replication of some, or all of the information in the resource. 
orderBy : A user can use this to order the result list. For which values the API supports ordering, is up to the API designer.
descending : Specifies that the orderBy parameter should sort descending, if it’s value is true. (i.e.: descending=false means ascending, descending=true means descending, a missing descending parameter means ascending.
offset : Used for paging. Specifies which result-row should be returned in the response as first element. If missing, 0 is assumed.
limit : Used for paging. Specifies a maximum number of results to return. If not specified the more general guideline of responses < 100Kb (after compression) should be respected. In other words, an implicit limit value should be part of the implementation. (To prevent a single request from affecting the web and/or database/datastore). Clients must support paging.
hrefs : comma separated list of resources to retrieve (permalinks). Allows for easy cross-referencing between backends. For example, one could retrieve a list of person keys by querying /responsibilities, and allow retrieval of all /persons resources with expansion=href. This can be combined with other query parameters, to support cross-backend joining.

Other URL parameters are used to filter on a specific value. The API designer must not try to cover all possible cases, but rather a sensible and broad set of filters and orderings. As noted above all sensible data-access paths should be available as list resources.

Keywords search. Implementations of list resources are advised to implement a broad, general query mechanism that selects resources based on any strong identifying property. WIth “strong identifying property” we mean any property of the resource that restricts the result list significantly. For example, when searching for a person, the name of the person is strongly identifying (not uniquely identifying, though). Of course, any unique value is strongly identifying as well, such as login, primary email address, national identity number, etc.. But also the city of his home or office address is significantly filtering the results (although less than the previous properties).

Such a generic ‘keywords-search’ should be implemented by using this convention : The name of the parameter should be q. The value of the parameter can have multiple keywords, separated by a plus (+) sign.

Examples : http://api.vsko.be/persons?q=Mieke+Heck

Searching through this q parameter should by definition be a broad search. So at least case-insensitive matches on sub-strings should be supported. Typically this list resource will be used to provide auto-completion in a user interface, so speed is very important. Implementations should consider fuzzy matching to overcome typing mistakes by end users.

The resulting resource should be a JSON object, containing under the key ‘results’, an array of JSON objects, a $$meta section with next, previous and count values. The objects in the response array (it should always be an object) should contain a value ‘href’ to point to a resource found in the system.

As stated above, the list resource must restrict the length of the list, even if there was no limit parameter, nor any filter parameters supplied. Avoiding server overload for certain (non-restricting) requests.

The values next and previous should point to a valid list resource for the next or previous page. If no next, or previous page is available, the value should be omitted.

Examples

GET /customers
200 OK
{
  $$meta: {
    count: 25263,
    next: "/customers?offset=30&limit=30"
  },
  results: [
    { href: "/customers/8eb525d8-302c-4fef-a0c9-5deae86590a5" },
    { href: "/customers/4d1d5a7d-9817-40b1-bc3b-2f71d5e05b89" },
    … 28 more results …
  ]
}

GET /customers?expand=href
200 OK
{
  $$meta: {
    count: 25263,
    next: "/customers?offset=30&limit=30"
  },
  results: [
    {
      href: "/customers/8eb525d8-302c-4fef-a0c9-5deae86590a5”,
      $$expanded: {
        $$meta: {
          permalink: "/customers/8eb525d8-302c-4fef-a0c9-5deae86590a5”,
          expansion: “FULL”,
          type: “user”
        },
        firstName: ‘John’,
        lastName: ‘Doe’,
        … a selection of data from this person ...
      }
    }
    … 29 more results …
  ]
}



Usage of keys
In the very least a UUID value should be available to identify a resource in the system in a reliable way. Every resource can be accessed on a resource that contains this basic UUID. All UUID’s should be represented in the same way: {8 characters}-{4 characters}-{4 characters}-{4 characters}-{12 characters}. All characters are lowercase. 

/schools/393f8347-8420-11e3-b29a-0c84dce06e32

If more universally accepted / more human readable keys are available (e.g. institution number for a school is also used by many parties outside the own organisation or the login of a person), that can result in an resource alias. In such a case the resource is available on more than one URL. Aliases are intended for developers only. Programs should use permalinks.

A person resource could be made available on http://api.vsko.be/people/{UUID} and at the same time on http://api.vsko.be/people/firstname.lastname. A school could be made available on http://api.vsko.be/schools/001644 (institution number).

/schools/001566
/persons/john.doe

All links between resource, including those within one module, should be declared as ‘href’ and have a permalink. If a link to another resource is needed, this link must be a permalink (i.e. the UUID-based URL)



Resource validation
In order to allow clients to reuse all or some of the validation logic implemented on the server, the APIs will expose their validation algorithm for every resource type. As we are not exposing resources here, but rather an algorithm we will use the POST verb for this. The URL on which we expose this algorithm is a sub-URL /validate of your list resources. 

For example /schools is a list resource, /schools/{uuid} is an regular school resource, and the validation algorithm for checking a single school resource will be exposed on /schools/validate.

The response to this POST operation will return the same body as described in Error Codes. 

Additionally this resource validation algorithm can return non-blocking warnings, rather than only blocking errors (as is the case for a PUT operation).

Resource creation
When a new resource is created, this should be done via an HTTP PUT operation. If the resource is created the server should return :
An HTTP 200 (OK)

If the resource could not be created, the response should be an HTTP 409 (Conflict). The entity of the response should contain a JSON document describing the reason creation was refused. This would typically be because of business logic validation. (See below for more details on the format of errors)

The client must generate a UUID based URL for the PUT operation.

Errors
Errors must be returned in the HTTP response body.  The response must contain a set of error objects. Each error object should contain a code and an path - if possible - that points to where the error occured in the document. In order to make life easier in asynchronous languages such as javascript, the server also returns the original JSON document in the response.

The code of an error/warning should always be in lowercase, and words should be separated by dots, as shown in the example.

To make sure the same code is used for the same errors across all the API’s we define the following codes. Make sure the paths property indicates to which property/list the error code applies:
‘property.missing’: when a required property is missing in the document.
‘property.value.invalid’: when the value for a property is not valid. A more specific message is better, this is the default fallback.
‘property.type.invalid’: when the type of the property is not as expected by the schema.
‘property.value.too.long’: when the input value is too long.
‘property.value.too.short’: when the input value is too short.
‘property.list.empty’: occurs when an list is empty that is not allowed to be.
‘enddate.before.startdate’: when a period is entered with an enddate before the startdate.
‘duplicate.key’: when the resource contains two objects with the same key.
‘key.not.unique’: when the resource contains a new object with a key that is not unique.
‘invalid.permalink’: when the document contains a link is a reference to a non-existing resource. A non-existing URL would resolve as 404 or 410. It is not required, but advised that API implementations check links in documents.

Every error/warning should have a type field that has value ‘ERROR’ or ‘WARNING’, as shown in the example.

Example : creation of a new school via PUT. A school with institution number 006122 is already in the datastore.

PUT /school/{UUID-1}
{
  key : {UUID-1}
  institutionNumber: "006122",
  seatAddresses: [
    {
      key : {UUID-2}
      street: "Haantjeslei",
      houseNumber: "50-52",
      zipCode: "2018xyz",
      city: "Antwerpen",
      subCity: "Antwerpen"
    }
  ],
  details: [
    {
      key : {UUID-3}
      startDate: “1940-09-01”,
      endDate: “2011-08-31”,
      name: "Ges. Vrije Basisschool (Gemengd)",
      shortName: "Ges. Vrije Basisschool (Gemengd)",
      callName: " Sint-Ludgardis Belpaire",
      affiliation: "VSKO",
      inclination: "Confessioneel katholiek",
      schoolNet: "Gesubsidieerd vrij onderwijs"
    },
  educationLevel: "BASIS",
  educationSort: "GEWOON",
  … …
}

409 Conflict
{
  errors : [
    {
      code : “duplicate.institutionnumber”,
      paths : [“institutionNumber”]
      type : “ERROR”
      .. the API can add extra keys to this error object to provide more context if desired ...
    },
    {
      code : “zip.code.invalid”,
      paths : [“seatAddress.zipCode”]
      type : “ERROR”
      .. the API can add extra keys to this error object to provide more context if desired ...
    },
    {
      code : “invalid.xyz”,
      type : “ERROR”
      .. the API can add extra keys to this error object to provide more context if desired ...
    }
  ],
  document : {
    key : {UUID-1}
    institutionNumber: "006122",
    seatAddresses: [
      {
        key : {UUID-2}
        street: "Haantjeslei",
        houseNumber: "50-52",
        zipCode: "2018xyz",
        city: "Antwerpen",
        subCity: "Antwerpen"
      }
    ],
    details: [
      {
        key : {UUID-3}
        startDate: “1940-09-01”,
        endDate: “2011-08-31”,
        name: "Ges. Vrije Basisschool (Gemengd)",
        shortName: "Ges. Vrije Basisschool (Gemengd)",
        callName: " Sint-Ludgardis Belpaire",
        affiliation: "VSKO",
        inclination: "Confessioneel katholiek",
        schoolNet: "Gesubsidieerd vrij onderwijs"
      },
    educationLevel: "BASIS",
    educationSort: "GEWOON",
    … …
  }
}

All possible errors should be exposed on your server implementation on /{type}/errors.
The format of this resource is an array of objects describing the errors and warnings :
message : A description of what went wrong, to help the developer. Human readable.
type : ERROR or WARNING.
status : The http status code returned for this type of error or warning.
code : The programmatic code that will be returned, clients can use this string to match the error.
Optional : extra keys that clarify the error further. For example : paths

Example :
GET /persons/errors
200 OK
[
 {
  message: “There is an existent resource that overlaps with the one you are trying to create.”,
  type: “ERROR”,
  code: “overlapping.period”,
  status : 409
  .. potentially extra properties ...
 },
 {
  .. more error messages/warnings ...
 }
]

Resource modification

HTTP PUT is the only supported way to update resources. If resources are stored in a representation close to the API form (think document stores like MongoDB/CouchDB), the most convenient implementation style would be to execute all relevant validation rules, and execute a document update on the datastore (after minor modification).

Most of our datastores are relational databases, however. Therefore, the previous section defined  that every level of a document should contain a key, to allow convenient mapping. These values are also handy in case of key-value stores.

The update cycle involves getting the resource as JSON, modifying some of the content, and executing an http PUT operation to the original URL.

The server should respond, if everything went well with an HTTP 200 (OK).

If a technical error is encountered (parsing, etc..) the server should respond with an HTTP 400 (Bad Request).

If a business (validation) error is encountered, the server should respond with an HTTP 403 (Forbidden), with the same type of entity as described above for resource creation.




Resource removal
Resources may be deactivated by executing an HTTP DELETE method on the corresponding URL. Internally the applications should not remove the information from the database/datastore, but rather mark the data as deleted. This is to prevent information loss.

When the operation succeeds the server should respond with an HTTP 200 (OK).

When the server was unable to execute the request, it should send an HTTP 403 (Forbidden), within the entity a JSON document describing the reason the operation failed. The format of this document is identical as described above for resource creation.

When a client subsequently tries to DELETE, GET (unless deleted=true, see below) or PUT a resource that was already deleted, the server must respond with HTTP 410 (Gone).

Deleted resources are still available with GET operation, if the client specifies deleted=true as an URL parameter. They do not appear in any list resources and cannot be GET as regular resources, unless the client specified deleted=true as an URL parameter. 

The deleted parameter, in other words, includes deleted resource when resolving GET operations. (on both regular, and list resources)

The deleted=true parameter is only intended for debugging, and should not be use by any regular client application.


Batch operations
In order to allow batch update/create (for performance), and to allow atomic update/create of multiple resources (for consistency), APIs can implement batch operations if they choose so.

Batch operations are always atomic. This is to say, the creation/updates of the resources in the batch are all applied, or all discarded. A single error causes the entire batch operation to fail.

To make the APIs consistent the URL of a batch operation must end in /batch.

The request body of a batch operation is a composition of other operations on resources :

[
  {
   “href” : “/schools/{uuid-generated-by-client}”,
   “verb” : “PUT”,
   “body” : {
    … Identical JSON structure as for a regular PUT operation’s body …
   }
  },
  {
   “href” : “schoollocations/{uuid-generated-by-client”,
   “verb” : “PUT”,
   “body” : {
    … Identical JSON structure as for a regular PUT operation’s body …
   }
  },
  // If you want to delete in a batch, this should be marked *explicitly* :
  {
   “href” : “/otherschoolresource/{uuid}”,
   “verb” : “DELETE”
   // body can be omitted for deletes.
  }
]

If “verb” is omitted, it must be interpreted as PUT. It is advised to specify it explicitly.

The response body matches this same composition, and returns the http status and body (if any) the regular PUT/DELETE/GET/.. operations would return.

[
 {
  “href” : “/schools/{uuid-generated-by-client}”,
  “status” : 200 /* the http status that would be given for this resource in a regular PUT */
  /* “body” is ommited here, if the server normally does not return any response body for 200 OK */
 },
 {
  “href” : “/schoollocations/{uuid-generated-by-client}”,
  “status” : 409,
  “body” : {
   /* The same response object that a regular PUT would return for an error situation */
  }
 }
]

Security
APIs can be called in 3 kinds of security context. An api can be called publicly (no known security context). It may be called on behalf of a certain user. Or it may be called in behalf of some technical system.

When called on behalf of a person, OAuth authentication is required.

When calling on behalf of a technical system/process, BASIC authentication should be used.

Algorithms
Implementations can expose various algorithms as a POST operation. The input and output of such an algorithm call should be JSON documents. Besides this the implementation can choose the content of those documents.

[roa-book]: http://www.crummy.com/writing/RESTful-Web-Services/
[roy-fielding]: http://www.w3.org/TR/webarch/#information-resource
[json-rfc]: http://tools.ietf.org/html/rfc7159
[hateoas]: http://en.wikipedia.org/wiki/HATEOAS
