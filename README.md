# Pagination patterns

This repo is a place to document patterns that I've learned for client/server pagination.

## What's pagination?

When building a RESTful web service that stores many records of a given type, clients of that service will want to query for records one chunk at a time. For example, a web app might ask the server for the first 10 users to display on a user list page. Then, when a "Next" button is clicked or the page is scrolled, the web app might ask the server for the next 10 users, and so on.

Build enough RESTful endpoints, and you'll start to feel this need over and over again, so it's helpful to have some patterns (both on the server and in the clients) to make pagination easier.

## Service design considerations

When building a web service, you can set yourself up for success by using the same conventions across all endpoints that list records of a given type.

### Request fields

First, how do clients of the service query for a subset of records from a list endpoint? All of your `List___` endpoints might support a shared set of fields:

```protobuf
message ListUsersRequest {
  // Number of records to include in the response. (Default: 0 = no limit)
  uint32 limit = 1;

  // The order to return records in (one of the enum values below).
  UsersOrder order = 2;

  // One or both of these fields can be specified to define a range of records.
  // The format is dependent on the sort:
  // - CREATED_AT: These fields would accept timestamps.
  // - NAME: These fields would accept names.
  string before = 3;
  string after = 4;
  
  // If true, return records in descending order. (Default: ascending)
  bool order_desc = 5;
  
  // Other endpoint-specific request fields...
}

enum UsersOrder {
  // Ordered by the timestamp when the user was created.
  CREATED_AT = 0;

  // Ordered alphabetically by the user's name.
  NAME = 1;
}
```

### Response fields

TODO
- Pagination state
- Limited history

## Client design considerations

TODO:
- Initial request
- Storing pagination state
- Subsequent requests
- "Topping up" time-period functionality
- Limited history

## Unaddressed issuses

There are some issues that I've run into which I haven't yet built good solutions for.

### Data changes during pagination

What happens if you're paginating through users sorted by name and a user changes their name from "Aaron" to "Zach"? There's potential for the same user to show up on both the first and last pages.

I believe that the concept of a "cursor" — which I've seen in other API designs — provides one possible avenue to fixing this. My understanding of this so far is that the first request in a paginated series would return a cursor ID, and the server would store info about that cursor. Subsequent requests can pass the "cursor" value to get the next records in the set based on a snapshot in time. If the data store supports getting the set of records at a particular point in time, then the cursor can store the time when the original request is made and continue to return results in subsequent responses consistent with that initial request time.

Another approach would be to have the client store the timestamp of the initial request and pass that as a `state_at` timestamp field with each subsequent request, instructing the server to return records as they were at that point in time.
