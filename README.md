# Building RESTful APIs

In the development of a project, an API is often the first code that gets written.  The success of the API can have a large influence on the success of all of the clients who consume it (iOS/Android/Web).  From that statement alone it is clear that building a great API is intrinsic to building a successful project.

We have created this document to outline both the development lifecycle and the standards we try to adhere to at Carrot when building a RESTful API.

## Sections & Summary

### The Process
1. [Discovery](#discovery) - Understanding the requirements
2. Sketch out your [Models](#models)
3. Create an [Endpoint Spec](#spec-out-restful-methods)
4. Define on an [Order of Implementation](#order-of-implementation)
5. Build it

### Best Practices
- [Base Interface](#base-interface)
- [Consistency in Models](#consistency-in-models)
- [Naming Conventions](#naming-conventions)
- [Pagination](#pagination)
- [API Versioning](#api-versioning)
- [Handling Errors](#handling-errors)

## Discovery

When starting any project, there should be an initial phase of discovery to ensure that you properly understand the requirements of the project.

This should be the first thing that occurs when getting thrown onto a project here at Carrot.  Your best bet is to get as aquainted with the project as possible -- this means you should be digging into wireframes + designs and asking for clarification on anything ambiguous.  Discovery phase isn't only limited to understanding the application, but you should also acquaint yourself with the entire ecosystem of the app.

Here's a (not fully exhaustive) list of a few questions you should be asking yourself:

- Will there be a CMS?
- Is the client responsible for hosting the servers?
- How quickly should this API be expected to scale?
- How large is the budget for hosting servers?
- Is there any web portal we are responsible for? (e.g. forgot password flow)

At the end of this phase, you should:

- Know what language you will be using
- Know what framework you will be using (if any)
- Know how you will be deploying to a staging server
- Know all components that the API team will be responsible for
- Have a solid understanding of all of the models (things) that exist within your application that would need to be managed in the API.

## Models

### Database Models

If you're using a relational database, you should spec out your tables and the columns within them. In the instance of a non-relational database, such as Redis, you should spec out all all keys and data-types (incl. mapped fields). Be sure to include keys containing data about relationship and "auto-incrementing" counters.

After you've created these model definitions, there should be a meeting between the API + client side teams to work to identify any shortcomings of the models (e.g. How are we going to show how old the pet is without a `born` column?).

### Interface Models

Interface models are the models as they will be represented in the API request.

Interface models aren't always exactly the same as the database models, so it's equally as important to spec these out.  These should be spec'ed out by the API team, and then also be brought to a meeting between the API + client side teams.  It's likely that this will be the same meeting that the database models are discussed.

## Spec Out RESTful Methods

Spec'ing out RESTful methods early will also make development easier for both the API + client side teams.

This will:

- Make splitting work easier for the API team
- Allow us to get a better idea if we are set to finish on time
- Allow the client side team to be involved in deciding the order of implementation
- Allow the API team to build clients that will consume the API early on

The API team should make an attempt at seeing what endpoints will be needed for the project, and list them out in a relatively rough manner:


```
[GET]    /users/{id}       // Gets a single user
[GET]    /users            // Gets a list of users, with parameters limit + offset
[PUT]    /users/{id}       // Updates a single user, with parameters name + age
[POST]   /users            // Creates a new user, with parameters name + age
[DELETE] /users/{id}       // Deletes a single user

[GET]    /users/{id}/homes // Get a list of user's homes
[POST]   /users/{id}/homes // Creates a new home associated to a specific user

[GET]    /pets/{id}        // Gets a single pet
[GET]    /pets             // Gets a list of pets, with parameters limit + offset
[PUT]    /pets/{id}        // Updates a single pet, with parameter name
[POST]   /pets             // Creates a new pet, with parameter name + owner_id
[DELETE] /pets/{id}        // Deletes a single pet
```

The idea behind doing this is so we get a strong idea of the endpoints we will need, without spending too much effort on each endpoint.

After this list is created by the API team, they should then again have another meeting with the client side teams to make sure we aren't missing any needed endpoints.  It's worth noting that this list could be created during the same meeting as the models meeting.  The only thing that matters is that both the API and client teams are involved in the processes.

After this list is hashed out, the team may move on to decide the order of implementation.

## Order of Implementation

In the event that the API is being written in tandem with the client applications, picking the right order of implementation for endpoints is critical for making sure the client side teams can work effectively without getting blocked.

Before any code is written, the API teams and the client side teams should work together to create a document with the order of implementation.  This document does not have to be taken 100% literally when implementation starts, as dev requirements sometimes force you to jump around a bit, but should be followed as closely as possible.

The API team should be in frequent communication with the client side teams to work together to make sure the next thing being implemented helps the client side teams from becoming blocked.

After this document has been completed, the API team may now move into implementation.

## During Implementation

As endpoints get implemented, they should be thoroughly documented so any consumers of the API can effectively understand the interface.

Ideally, as endpoints are implemented the API team should create formalized documentation or examples of how the endpoint works.

A commented CURL script is probably the easiest way to do this:

```bash
curl \
    --request POST \
    --header "Accept: application/json" \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --header "X-SESSION-TOKEN: xxxxxxxxxx" \
    --data 'post[post_type]=text&post[content]=lorem+ipsum' \
    'http://api.someapp.com/api/v1/posts'
```

If you take this approach, it's very useful because at the same time you're building out the spec, you're also building out a very simple client you can use to test the API with.

## Base Interface

The base interface of an API is usually a small decision with big implications to usability and efficiency.

The Carrot team has thought long and hard about a base API interface, and have landed on something that looks like this:

```json
{
    "success": true,
    "status_code": 200,
    "status_text": "OK",
    "error_details": [
        {
            "code": 1,
            "text": "Invalid email"
        }
    ],
    "content": {
        // ...
    }
}
```

Here is a brief explanation of each field:

**success**: This is a boolean that describes if the request was successful or not.  A `true` value for this should always be accompanied by a 2xx `status_code`.  This exists so we are able to run a quick check against the response.

**status_code**: This is a redundancy against the actual HTTP Status Code thrown for ease of client consumption.  It is much easier for some HTTP clients to parse the body for a value than it is to get the actual HTTP Status Code, so we offer both for flexibility.  The value of the `status_code` should always match the actual HTTP Status Code thrown.

**status_text**: This is a human readable redundancy for readability, and should never be checked against by code.  This should always match the `status_code` spec as defined [here](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html).  It is frequent client side developers do not know all of the HTTP Status Codes by memory, so this will help them be able to quickly identify what went wrong with a request.

**error_details[x].code**: This is intended to be a unique error code specific to the application.  If two different errors can throw the same HTTP Status Code, this is the value that will allow you to distinguish between the two.

For example, when registering for an account, the user could have input an invalid email format, or the email has been taken.  The HTTP spec tells us both of these are 400 errors.  On the client side we would probably want to distinguish between these two to prompt correct action from the user.  This is an example where you would want to use an error detail.

**error_details[x].text**: This is an `error_details[x].code` human readable redundancy, and should never be checked against by code.  The point of this is so developers don't have to look up what the `code` means when they are testing code.

**content**: This is the actual content of the response.  This could either be an array (for lists), or an object (for single object requests).  In the event there is an API endpoint in which is supposed to return multiple objects (`/v1/users` for example) and only returns one because that's all the data there is, we should still return an array with a single object inside of it.

## Consistency in Models

The content of a model should always remain the same.  In the event that a single request for a model requires additional data, it should not be stuffed in the currently existing model.

For example, if we were interested in getting a users friends, we would not throw an array in a user object called `friends` (unless it was always there).

**Don't do this:**

```json
{
    // ...
    "content": {
        "id": 1,
        "username": "Brandon",
        "friends": [
            // ...
        ]
    }
}
```

**Do this (if appropriate):**

```json
{
    // ...
    "content": {
        "user": {
            "id": 1,
            "username": "Brandon"
        },
        "friends": [
            // ...
        ]
    }
}
```

From a RESTful perspective, we should prefer adding a new method `[GET] /users/{id}/friends` which will return an array of that users friends. Sometimes it is appropriate to follow the method labeled `Do this (if appropriate)` to prevent request overhead.

## Naming Conventions

- Keys should always be in `snake_case`
  - Correct: `"number_of_cats": 4`
  - Incorrect: `"NumberOfCats": 4`
- Intended numerical values should never be strings, but as ints or floats.
  - Correct: `"number_of_cats": 4`
  - Incorrect: `"number_of_cats": "4"`
- Booleans as `true` and `false`
  - Correct: `"owns_cats": true`
  - Incorrect: `"owns_cats": 1`
  - Incorrect: `"owns_cats": True`
- Dates should follow the `ISO8601` specification, the more specific the better.
  - Correct: `"created_at": "2015-10-15"`
  - Correct: `"created_at": "2015-10-15T13:21:09+00:00"`
  - Correct: `"created_at": "2015-10-15T16:31:01.080904-04:00"`

## Pagination

### Limit + Offset

In the event that an API does not require intensive caching, use `limit` and `offset` to manage pagination.  This is the easiest of the two pagination strategies to implement.

`offset` should always default to 0.  The offset is an offset relative to the number of items, and not the page number.

`limit` should always default to a reasonably low number, and can be decided on a per-project basis.

It is worth noting that we do not make any changes to the base interface for paginated requests, as some APIs do.  Some APIs add an additional value in the base to indicate how many pages are left (or total # of pages), where as we prefer to not add something like this.

Adding this feature is actually very intensive on the API, as you have to run a `COUNT(SELECT * FROM table_name)` query, which actually reads the entire table (which is clearly not ideal).

From a client side developer perspective, there are two patterns that traditionally will need to be solved with bulk fetches:

**Unlimited Scrolling**

To handle this case, you should simply just query the API until there is a response with 0 models inside of it.

**Traditional Paged Content**

For traditionally paged content, you will actually need to get the count of elements from the API to be able to tell if there is a next page or not (or possibly even how many pages).

Here the API will need to supply either another endpoint to get the count or the API team will actually need to supply the count in every request.

If this UI pattern comes up, the API teams and client side teams should meet to come up with the best solution for the problem on a case-by-case basis.

## API Versioning

All APIs URLs should first start with /{VERSION}/.  Here are a few example API endpoints that would be valid:

```
[GET] https://api.carrot.is/v1/users
[GET] https://api.carrot.is/v1/users/{id}
[GET] https://clientwebsite.com/api/v1/users
```

As an API developer, it is your responsibility to know when you introduce a breaking change into the API.  When this happens and the application is in development, inform the client side developers.  When this happens and the application is in production, there will need to be a new version introduced.

## Handling Errors

Use appropriate [HTTP Status Codes](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) for errors.

Actually throw the HTTP error appropriately and also stuff it into the `status_code` field in the base interface.
