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

All resources are represented as [JSON documents][json-rfc]. 

### Regular Resources
All *regular* resource MUST be available on a *permalink*. *Permalinks* are of the format `/{type}/{uuid}`. Server implementations are RECOMMENDED to implement more human readable URLs (aliases). A person is OPTIONALLY also available on `/persons/john.doe`. Links between the resources MUST always use *permalinks*. Aliases MUST only be provided for convenience during development.

*Regular* resources SHOULD be exposed on a URL that uses a plural form for it's *type* section. (i.e. `/schools/{guid}` and not `/school/{guid}`)

When defining the resources in an SRI API, a designer should also consider security constraints. Security MUST be applied per URL, so the API MUST split resources along security boundaries. For example a person’s public information is a separate resources from his private information.

Clients are advised to strive to use the [“Hypermedia as the Engine of Application State”][hateoas] principle as much as possible. 

Clients can safely store *permalinks*. The server MUST, in other words, support these URLs over a long period of time. 

The `expand` parameter can be used on *regular* resources to perform inlining of referenced resources. The client can specify one or more dot-separated paths (relative to the JSON document), that need to be included. Multiple expansions SHOULD be requested by separating them with a comma. The server MUST define which paths are supported for expansion. The server is RECOMMENDED to support a consistent schema for expansion, such as all direct references in a resource. All references to other resources MUST be embedded in a JSON object, with a key `href`. When executing expansion the server MUST add an extra key `$$expanded` to that object, including the full *regular* resource that was expanded.

Arrays in *regular* resources SHOULD BE named in plural nouns. i.e. schools, rather than school for an array of JSON objects about schools. 

Keys in the JSON responses SHOULD BE camelCasedLikeThis. Acronyms SHOULD NOT be in all capitals. For example ZIP code is not implemented as ZIPcode, but rather zipCode.

For *regular* resources all JSON objects in the document, including the root object, MUST expose `key` that contains a UUID to uniquely identify separate sections in the JSON document. For the root object this key MUST correspond to the UUID section of the *permalink*. This also applies to arrays of objects within the document.

A regular resource should include a `$$meta` section. Such a section MUST include :

- a permalink to itself, under `permalink`.
- a link to the json schema of this resource, under `schema`.

It MAY also include an array of supported aliases for this resource, under `aliases`.
Server implementations MAY choose to add extra keys to the `$$meta` section.

Example (notice the `expand` URL parameter to include a related resource):

    GET http://api.mine.org/schools/006613?expand=director
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
Country codes MUST adhere to [ISO-3166-1 alpha-2][format-countrycodes].

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

*List* resources MUST support paging, and servers MUST impose a maximum number of items that will be returned on a single GET operation (i.e. servers MUST enforce paging if more than a reasonable number of items is requested).

*List* resources can be *filtered* through the use of URL parameters. All parameter names and parameter values MUST be camelCasedLikeThis, but the resource names MUST be in lower case. If the parameter name or the value is not recognized by the server or not valid for this *list* resource, the server MUST return with HTTP status code `404 Not Found`. The server MUST return a error message in the response body (see below for errors). 

*List* resources support URL parameter `expand` for *expanding* a resource. *Expansion* makes a *list* resource include the related *regular* resources in `results[x].$$expanded`. The `expand` parameter MUST accept one or more dot-separated paths, relative to the response object. If more than one property paths is specified they MUST be seperated with a comma. To include the regular resources from the `results` array :

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
  
Servers MAY also allow expansion of more information. For example the *regular* resource in the *list* resource could have a reference to a different *regular* resource :

    GET /persons?expand=results.href.father
    200 OK
    { 
        $$meta: { ... },
        results: [
            {
                href: '/persons/{guidA}',
                $$expanded: {
                    ...
                    father: {
                        href: '/persons/{guidB}',
                        $$expanded: {
                            ...
                        }
                    }
                    ...
                }
            }
        ]
    }

*List* resource MUST be found on the base URL of the corresponding *regular* resources. If a school can be found at `/schools/{guid}`, then a list of references of all schools can be queried on `/schools`.

*List* resource must all support the following standard URL parameters by default :

- `modifiedSince` : Limits `results` to references to resources that were created or modified since the given timestamp. 
- `orderBy` : Orders `results`. Servers can determine what possible ordering they support. By default the sort order is ascending.
- `descending` : Specifies that the `orderBy` parameter should sort descending, if it’s value is true.
- `offset` : Used for paging. The server MUST skip the first n references from the result set.
- `limit` : Used for paging. Specifies a maximum number of results to return. The server MUST  have a default value for `limit`, that limits the maximum transfer size of the response.
- `hrefs` : A comma separated list of permalinks. The server MUST limit the results to these items.

All URL parameters can be combined, where the filtering is combined in an `AND` fashion.

The server is RECOMMENDED to support a number of sensible extra URL parameters. 

The server MUST process all URL parameters case-insensitive.

A server MAY choose to support full-text search. If it does so it should implement this as URL parameter `q`. In case multiple search terms are submitted to the server they MUST be separated by a plus (+) sign.

    GET /persons?q=John+Doe

List resources must contain a `$$meta` section. This `$$meta` section MUST contain these keys :

- `count` : indicates the total number of references that match with the supplied URL parameters.
- `next` : If more references are available for the specified URL parameter, and the `limit` has been applied to this *list* resource, `next` must contain a relative link to the *list* resource that contains the next page. Otherwise it SHOULD be omitted.
- `previous` : If the client specified an `offset` that is larger than `limit`, then `previous`. Otherwise it SHOULD be omitted.

## Usage of keys
Every *regular* resources MUST have a UUID value to identify it in the system in a reliable, stable way. This UUID MAY NEVER change over the lifetime of the resource. All UUID’s should be represented in the same way: {8 characters}-{4 characters}-{4 characters}-{4 characters}-{12 characters}, including hyphens. All characters MUST be in lower case.

    GET /schools/393f8347-8420-11e3-b29a-0c84dce06e32

## Resource validation
In order to allow clients to reuse all or some of the validation logic implemented on the server, the server MUST expose their validation algorithm for every resource type. Clients can perform a `POST` operation to `/{type}/validate`. If `/schools` is a list resource, `/schools/{uuid}` is an regular school resource, and the validation algorithm for checking a single school resource will be exposed on `/schools/validate`. The response of the validation is almost identical to a `PUT` operation with the same resource (`200 OK`, or `409 Conflict`), except that the operation has no side-effect (i.e. the resource is not updated). Additionally this resource validation algorithm can return non-blocking warnings, rather than only blocking errors (as is the case for a PUT operation).

    POST /schools/{guid}
    {
        key: '{guid}',
        institutionNumber: '128256',
        ...
    }
    
    200 OK

The response to this POST operation will return the same body as described in section about error codes. 

## Resource creation
When a new resource is created, this should be done via an `PUT` operation. If the resource is created the server SHOULD return `200 OK`. If the resource could not be created, the response should be an `409 Conflict`. The entity of the response should contain a JSON document describing the reason creation was refused. See the section on error messages for more details on the format of errors.

The client MUST generate a UUID based URL for the PUT operation.

## Errors
Errors must be returned in the HTTP response body.  The response must contain a set of error objects. Each error object SHOULD contain a `code` and, optionally a `path` that points to where the error occured in the document (as a dot-separated property path, relative to the JSON document). The server MUST also return the original JSON document in the response, under `document`.

The `code` of an error/warning should always be in lower case, and words should be separated by dots.

Some error codes are standardized. Servers MUST use these standard error appropriately.

- `property.missing` MUST be used when a required property is missing in the document.
- `property.value.invalid` MUST be used when the value for a property is not valid.
- `property.type.invalid` MUST be used when the type of the property is not as expected by the schema. 
- `property.value.too.long` MUST be used when the input value exceeds the maximum size.
- `property.value.too.short` MUST be used when the input value is too short.
- `property.list.empty` MUST be used when an list is empty, when it is not allowed to be.
- `duplicate.key` MUST be used when the resource contains two objects with the same key.
- `key.not.unique` MUST be used when the resource contains a new object with a key that is not unique.
- `invalid.permalink` MAY be used when the document contains a link to a non-existing resource.

Every warning SHOULD have a `type` field that has value `WARNING`. Error messages MAY contain a `type` field with value `ERROR`.

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

All possible errors should be exposed on your server implementation on `/{type}/errors`.
The format of this resource is an array of objects describing the errors and warnings. The objects have these keys :

- `code` : The programmatic code that will be returned, clients can use this string to match the error.
- `type` : ERROR or WARNING.
- `status` : The http status code returned for this type of error or warning.
- `message` : A description of what went wrong, to help the developer. Human readable.

Extra keys that clarify the error further MAY be added by the server.

Example :

    GET /persons/errors
    200 OK
    [
     {
      code: “overlapping.period”,
      type: “ERROR”,
      status : 409,
      message: “There is an existent resource that overlaps with the one you are trying to create.”
      .. potentially extra properties ...
     },
     {
      .. more error messages/warnings ...
     }
    ]

## Resource modification
A `PUT` is the only supported operation to update resources. The server MUST respond `200 OK` if the update was succesful. If a technical error is encountered (parsing, etc..) the server SHOULD respond with an `400 Bad Request`. If a validation error is encountered, the server should respond with an `409 Conflict`, with the same type of entity as described for resource creation.

## Resource removal
A server SHOULD deactivate a resource when receiving a `DELETE` operation on the corresponding URL. Internally the server SHOULD NOT remove the information, but rather mark the data as deleted. The server SHOULD implement an archiving strategy. This is beyond the scope of this document.

When the operation succeeds the server SHOULD respond with an `200 OK`.

When the server was unable to execute the request, it SHOULD send `409 Conflict`. The response body must be of the same format as described in the errors section.

When a client subsequently tries to DELETE, GET (unless he is using URL parameter `deleted=true`, see below) or PUT a resource, the server MUST respond with `410 Gone`.

If the client specifies `deleted=true` as an URL parameter (when requesting either a *regular* resource, or a *list* resource), the resource SHOULD be returned as if it was not deleted. A deleted resource MUST include, on the top level `deleted` with value `true`.

## Batch operations
In order to allow multiple operations to happen in an atomic way servers MUST implement a `/batch` endpoint.

Batch operations are always atomic. This is to say, the operations (creation/updates/deletes) of the resources in the batch are all applied, or all discarded. A single error MUST cause the entire batch operation to fail.

The request body of a batch operation is a composition of other operations on resources as such :

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

If `verb` is omitted, it MUST be interpreted as PUT.

The response body matches this same composition, and returns the http status and body (if any) the regular operations would return. A client MAY combine all types of operations (GET/PUT/POST/DELETE) in a single batch operation.

    [
     {
      “href” : “/schools/{uuid-generated-by-client}”,
      “status” : 200
     },
     {
      “href” : “/schoollocations/{uuid-generated-by-client}”,
      “status” : 409,
      “body” : {
       ...
      }
     }
    ]

## Algorithms
Implementations can expose various algorithms as a POST operation. The input and output of such an algorithm call SHOULD be JSON documents. Besides this the server can choose the content of those documents.

[roa-book]: http://www.crummy.com/writing/RESTful-Web-Services/
[roy-fielding]: http://www.w3.org/TR/webarch/#information-resource
[json-rfc]: http://tools.ietf.org/html/rfc7159
[hateoas]: http://en.wikipedia.org/wiki/HATEOAS
[format-timestamps]: http://tools.ietf.org/html/rfc3339#section-5.6
[format-countrycodes]: http://en.wikipedia.org/wiki/ISO_3166-1_alpha-2
[format-urls]: http://tools.ietf.org/html/rfc3986
[format-emails]: http://tools.ietf.org/html/rfc5322#section-3.4.1

