# Standardized ROA Interface Specification

## Introduction
This specification describes a set of requirements for implementing RESTful interfaces. An interface that adheres to this specification qualifies as an SRI interface. In short SRI defines a number of things about a RESTful interface :

- The api must support *regular* resources.
- The api must expose a sensible set of queries on *regular* resources as *list* resources.
- The api uses keys and permalinks in a certain way.
- *List* resource all support a set of common URL query parameters, pagination, ordering, expansion, etc..
- Batch operations are supported to to enforce atomic updates/creates of multiple resources.
- All resource types must have a JSON schema associated.
- etc...

RESTful web services follow the general architecture used to build the world wide web. This architectural style is described in [Roy Fielding’s dissertation][roy-fielding].

## Representation of resources
We make a distinction between *list* resources and *regular* resources. *List* resources represent queries, and contain an array with many result objects. *Regular* resources correspond to single resources in a system.

All resources are be represented as [JSON documents][json-rfc]. 

URLs in resources must always be relative. By relative we mean the URL does not include protocol, host nor port. It *should* include a full path. For example a school is available on `/schools/{uuid}`, a mammal on `/mammals/{uuid}`, a person on `/persons/{uuid}`. A *list* of schools on `/schools`, a list of persons on `/persons`.

### Regular Resources
All *regular* resource MUST available on a *permalink*. *Permalinks* are of the format `/{type}/{uuid}`. Server implementations are RECOMMENDED to implement more human readable URLs (aliases). A person is OPTIONALLY also available on `/persons/john.doe`. Links between the resources MUST always use *permalinks*. Aliases MUST only provided for convenience during development.

*Regular* resources SHOULD be exposed on a URL that uses a plural form for it's *type* section. (i.e. `/schools/{guid}` and not `/school/{guid}`)

When defining the resources in an SRI API, a designer should also consider security constraints. Security MUST be applied per URL, so the API MUST split resources along security boundaries. For example a person’s public information is a separate resources from his private information.

Clients are advised to strive to use the [“Hypermedia as the Engine of Application State”][hateoas] principle as much as possible. 

Clients can safely store *permalinks*. The server MUST, in other words, support these URLs over a long period of time. 

The `expand` parameter can be used on *regular* resources to perform inlining of referenced resources. The client can specify dot-separated paths (relative to the JSON document), that need to be included. The server MUST define which paths are supported for expansion. The server is RECOMMENDED to support a consistent schema for expansion, such as all direct references in a resource. All references to other resources MSUT embedded in a JSON object, with a key `href`. When executing expansion the server MUST add an extra key `$$expanded` to that object, including the full *regular* resource that was expanded.

Arrays in the JSON responses SHOULD BE named in plural nouns. i.e. schools, rather than school for an array of JSON objects about schools. 

Keys in the JSON responses SHOULD BE camelCasedLikeThis. Acronyms SHOULD NOT be in all capitals. For example the ZIP code is not implemented as ZIPcode, but rather zipCode.

For *regular* resources all JSON objects in the document, including the root object, MUST expose `key` that contains a UUID to uniquely identify separate sections in the JSON document. For the root object this key MUST correspond to the UUID section of the *permalink*. This also applies to arrays of objects within the document.

A regular resource should include a `$$meta` section. Such a section MUST include :

- a permalink to itself, under `permalink`.
- a link to the json schema of this resource, under `schema`.

It MAY also include an array of supported aliases for this resource, under `aliases`.
Server implementations MAY choose to add extra keys to the `$$meta` section.

Example (notice the `expand` URL parameter to include a related resource):

    GET http://api.vsko.be/schools/006613?expand=schoolLocations
    {
      $$meta: {
        permalink: ‘/schools/{UUID}’,
        schema: '/schools/schema',
        aliases: [‘/schools/006613’]
      }
      institutionNumber : “006613”
      ...
      director : {
        href: ‘/persons/{uuid},
        $$expanded: {
          $meta: {
            permalink: ‘/persons/{uuid}’,
          }
          ...
        }
      }
    }

Timestamps MUST be formatted according to [RFC 3339, section 5.6][format-timestamps].
Emails MUST be formatted according to [RFC 5322, section 3.4.1][format-emails].
URLs MUST be formatted according to [RFC3986][format-urls].
Country codes MUST adhere to [ISO-3166-1][format-countrycodes].

Every *regular* resource MUST have a JSON schema definition. This schema should be exposed on `/{resource name}/schema`.

Constants SHOULD BE included in capitals, they should be clear terms that at least facilitate debugging / exploring the API.

### List resources
A *list* resource contains a set of references to *regular* resources. When requesting a *list* resource references to all items of a certain type are returned, by default. The references are returned in an array under key `results`. Inside the `results` array objects appear with an `href` that contains the *permalink* to a regular resource.

    GET /persons
    200 OK
    {
      $$meta: { ... }
      results: [
        { href: '/persons/{guid1}' },
        { href: '/persons/{guid2}' },
        ...
      ]
    }

*List* resources MUST provides paging through, and servers MUST impose a maximum number of items that will be returned on a single GET operation.

*List* resources can be *filtered* through the use of URL parameters. All parameter names and parameter values should be camelCasedLikeThis, but the resource names should be lower cased. If the parameter name or the value is not recognized by the server or not valid for this resource a developer-friendly error message SHOULD be returned with HTTP status code `404 Not Found`.

*List* resources support URL parameter `expand` for *expanding* a resource. *Expansion* makes a *list* resource include the related *regular* resources in `results[x].$$expanded`. The `expand` parameter MUST accept dot-separated paths, relative to the response object. To include the regular resources from the `results` array :

    GET /customers?expand=results.href
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
  
Servers MAY also allow expansion of more levels :

    GET /persons?expand=results.href.father
    200 OK
    { ... }

*List* resource MUST be found on the base URL of the corresponding *regular* resources. If a school can be found at `/schools/{guid}`, then a list of references of all schools can be queried on `/schools`.

*List* resource must all support the following standard URL parameters by default :

- `modifiedSince` : Limits `results` to references to resources that were created or modified since the given timestamp. 
- `orderBy` : Orders `results`. Servers can determine what possible ordering they support. By default the sort order is ascending.
- `descending` : Specifies that the `orderBy` parameter should sort descending, if it’s value is true.
- `offset` : Used for paging. The server MUST skip the first n references from the result set.
- `limit` : Used for paging. Specifies a maximum number of results to return. The server is RECOMMENDED to have a default value for `limit`, that limits the maximum transfer size of the response.
- `hrefs` : A comma separated list of permalinks. The server MUST limit the results to these items.

All URL parameters can be combined, where the filtering is combined in an `AND` fashion.

The server is RECOMMENDED to support a number of sensible extra URL parameters. 

The server MUST process all URL parameters case-insensitive.

A server MAY choose to support full-text search. If it does so it should implement this as URL parameter `q`. In case multiple search terms are submitted to the server they MUST be separated by a plus (+) sign.

    GET /persons?q=John+Doe

List resources must contain a `$$meta` section. This `$$meta` section MUST contain these keys :

- `count` : indicates the total number of references that match with the supplied URL parameters.
- `next` : If more references are available for the specified URL paramter, and the `limit` has been applied to this *list* resource, `next` must contain a relative link to the *list* resource that contains the next page. Otherwise it SHOULD be omitted.
- `previous` : If the client specified an `offset` that is larger than `limit`, then `previous`. Otherwise is SHOULD be omitted.

## Usage of keys
Every *regular* resources MUST have a UUID value to identify it in the system in a reliable, stable way. This UUID MAY NEVER change over the lifetime of the resource. All UUID’s should be represented in the same way: {8 characters}-{4 characters}-{4 characters}-{4 characters}-{12 characters}, including hyphens. All characters MUST be lowercase.

    GET /schools/393f8347-8420-11e3-b29a-0c84dce06e32

## Resource validation
In order to allow clients to reuse all or some of the validation logic implemented on the server, the APIs will expose their validation algorithm for every resource type. As we are not exposing resources here, but rather an algorithm we will use the POST verb for this. The URL on which we expose this algorithm is a sub-URL /validate of your list resources. 

For example /schools is a list resource, /schools/{uuid} is an regular school resource, and the validation algorithm for checking a single school resource will be exposed on /schools/validate.

The response to this POST operation will return the same body as described in Error Codes. 

Additionally this resource validation algorithm can return non-blocking warnings, rather than only blocking errors (as is the case for a PUT operation).

## Resource creation
When a new resource is created, this should be done via an HTTP PUT operation. If the resource is created the server should return :
An HTTP 200 (OK)

If the resource could not be created, the response should be an HTTP 409 (Conflict). The entity of the response should contain a JSON document describing the reason creation was refused. This would typically be because of business logic validation. (See below for more details on the format of errors)

The client must generate a UUID based URL for the PUT operation.

## Errors
Errors must be returned in the HTTP response body.  The response must contain a set of error objects. Each error object should contain a code and an path - if possible - that points to where the error occured in the document. In order to make life easier in asynchronous languages such as javascript, the server also returns the original JSON document in the response.

The code of an error/warning should always be in lowercase, and words should be separated by dots, as shown in the example.

To make sure the same code is used for the same errors across all the API’s we define the following codes. Make sure the paths property indicates to which property/list the error code applies:

- `property.missing` : when a required property is missing in the document.
- ‘property.value.invalid’: when the value for a property is not valid. A more specific message is better, this is the default fallback.
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
      ...
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
        ...
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

## Resource modification
HTTP PUT is the only supported way to update resources. If resources are stored in a representation close to the API form (think document stores like MongoDB/CouchDB), the most convenient implementation style would be to execute all relevant validation rules, and execute a document update on the datastore (after minor modification).

Most of our datastores are relational databases, however. Therefore, the previous section defined  that every level of a document should contain a key, to allow convenient mapping. These values are also handy in case of key-value stores.

The update cycle involves getting the resource as JSON, modifying some of the content, and executing an http PUT operation to the original URL.

The server should respond, if everything went well with an HTTP 200 (OK).

If a technical error is encountered (parsing, etc..) the server should respond with an HTTP 400 (Bad Request).

If a business (validation) error is encountered, the server should respond with an HTTP 403 (Forbidden), with the same type of entity as described above for resource creation.

## Resource removal
Resources may be deactivated by executing an HTTP DELETE method on the corresponding URL. Internally the applications should not remove the information from the database/datastore, but rather mark the data as deleted. This is to prevent information loss.

When the operation succeeds the server should respond with an HTTP 200 (OK).

When the server was unable to execute the request, it should send an HTTP 403 (Forbidden), within the entity a JSON document describing the reason the operation failed. The format of this document is identical as described above for resource creation.

When a client subsequently tries to DELETE, GET (unless deleted=true, see below) or PUT a resource that was already deleted, the server must respond with HTTP 410 (Gone).

Deleted resources are still available with GET operation, if the client specifies deleted=true as an URL parameter. They do not appear in any list resources and cannot be GET as regular resources, unless the client specified deleted=true as an URL parameter. 

The deleted parameter, in other words, includes deleted resource when resolving GET operations. (on both regular, and list resources)

The `deleted=true` parameter is only intended for debugging, and should not be use by any regular client application.

## Batch operations
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

## Security
APIs can be called in 3 kinds of security context. An api can be called publicly (no known security context). It may be called on behalf of a certain user. Or it may be called in behalf of some technical system.

When called on behalf of a person, OAuth authentication is required.

When calling on behalf of a technical system/process, BASIC authentication should be used.

## Algorithms
Implementations can expose various algorithms as a POST operation. The input and output of such an algorithm call should be JSON documents. Besides this the implementation can choose the content of those documents.

[roa-book]: http://www.crummy.com/writing/RESTful-Web-Services/
[roy-fielding]: http://www.w3.org/TR/webarch/#information-resource
[json-rfc]: http://tools.ietf.org/html/rfc7159
[hateoas]: http://en.wikipedia.org/wiki/HATEOAS
[format-timestamps]: http://tools.ietf.org/html/rfc3339#section-5.6
[format-countrycodes]: http://en.wikipedia.org/wiki/ISO_3166-1_alpha-2
[format-urls]: http://tools.ietf.org/html/rfc3986
[format-emails]: http://tools.ietf.org/html/rfc5322#section-3.4.1

