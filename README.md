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

How does the server return data and pagination state info to clients? I've found that it's helpful to have a part of the response be dedicated to explicit pagination data (which helps to simplify client implemtation).

```protobuf
message ListUsersResponse {
  // One or more user records that fit the request criteria.
  repeated User users = 1;
  
  // Pagination data for the client to use in making subsequent requests.
  Pagination pagination = 2;
}

message Pagination {
  // Whether there are more items or not (and if not, why not.)
  PaginationState pagination_state = 1;
  
  // The value of the ordered field for the last record in the set. It can be provided directly to `before/after` fields.
  // (This means the client doesn't have to know how to find it within records.)
  string last = 2;
}

enum PaginationState {
  // There are more items / we're not certain we have reached the end yet.
  CONTINUE = 0;
  
  // There are no more items to load.
  END = 1;
  
  // There are more items, but they are out of range for the user.
  // (This can be used if a "free" version of the service only allows 30 days of history, for instance.)
  LIMITED = 2;
}
```

### Populating response fields

How does the service know whether there are more items in the set after the last item that's being returned? This problem can be solved by requesting `limit + 1` records from the data store and then removing the last record before returning results in the response. The presence of that extra record indicates whether there are more to come, and it's typically not too espensive to request one extra and throw it away.

## Client design considerations

Once all of your `List___` service endpoints use the same conventions for pagination, you've got less work to do on the client in order to make requests for new paginated data sets.

### Making an initial request

Typically, a component that displays data on the client will kick off the process by loading a first page of data. For example, a React component that displays a list of users might kick off a request to the `ListUsers` endpoint with the following data:

```js
listUsers({
  team_id: '12345',
  order: USERS_ORDER.NAME,
})
```

Assuming that the application is using some sort of uni-directional state container like Redux to store state, the above action might do the following:
- Create a key to store the resulting set of users based on the request fields (e.g. `order=NAME&team_id=12345`)
- (Optionally) clear that stored data set, if there's any stale copy of the same set of users already loaded in the client, so that the client displays an empty loading state
- Store a `loading: true` state at that key, to indicate that data for the first page of that set of users is loading
- Asychronously kick off an HTTP request to the endpoint, adding `limit: 10` to the fields sent to the server
- When the response arrives, store the records and the pagination state from the response at that key, also setting `loading: false` at the same time
- If the data wasn't cleared earlier at the start of loading (perhaps because the client wants to continue to display stale data while re-loading the first page), then the existing data will be replaced by this response

Note: When using a state container like Redux, you're probably storing the records for a type of entity like "users" in a single place, so the above response might do two things:
- Add or update user records to an `entities` map keyed by user ID
- Update a `user_lists` map keyed by the request fields with the ordered list of user IDs plus the pagination state.

### Making subsequent requests

Next — if a "Load more" button is clicked, for example — the React users component might need to load the next 10 users in the same set of data:

```js
loadMoreUsers({
  team_id: '12345',
  order: USERS_ORDER.NAME,
})
```

Notice that the set of fields being passed to this `loadMoreUsers` method is exactly the same as the set that was passed to the initial `loadUsers` method. You can think of this set of fields as a "query key" that uniquely identifies a set of records to load from. Any fields like `limit`, `before`, and `after` are not included, and are added behind the scenes as additional pages are requested.

Calling the above action might do the following:
- Check the existing stored pagination state for this query key
- Store a `fetching: true` state at the key, indicating that additional data for that set of users is being retrieved
- Asynchronously kick of an HTTP request to the endpoint, adding `limit: 10` and `after: paginationState.last` to the request fields
- When the response arrives, store the records and update the pagination state, setting `fetching: false` at the same time

### Reaching the end of a data set

The client can continue to call `loadMoreUsers` repeatedly until the stored pagination state indicates an `END` state, at which point it can display a "No more items" UI.

If the service supports the `LIMITED` pagination state, indicating that a free plan only allows a certain amount of history, then once the client sees the `LIMITED` pagination state, they might display a UI prompting the user to upgrade to see additional history.

### "Topping up" a time period

Some clients may want to display data in a time-bucketed UI, in which case they want to be certain that they've loaded all of the records from a particular time period.

For example, say that we're building a UI that displays users in groups by the month that they were created. How can we be sure that we've loaded all of the users for a given month before displaying the UI for that month's group?

To address this, one can build "topping up" functionality into the client-side pagination helpers. The React component might call the same action as before, but with an additional period option:

```js
listUsers({
  team_id: '12345',
  order: USERS_ORDER.NAME,
}, {
  // Options
  period: PERIOD.MONTH,
})
```

Behind the scenes, calling the above action would follow the same process as before until it received the first page of results from the server. Then, it would do the following:
- Make a subsequent request for more items in the same month as the `last` timestamp (by adding a `before` or `after` field)
- If the response from that request is full (i.e. 20 more items in July were requested, and we got 20 items in the response), then make an additional request for the next 20 items in the remaining portion of the month
- Keep repeating that process until we get a response with fewer than 20 items
- Now that we're sure we've aligned our "page boundary" with a month boundary, store all the records and pagination state, and set the loading state to false again, so the client can display the results

## Unaddressed issuses

There are some issues that I've run into which I haven't yet built good solutions for.

### Data changes during pagination

What happens if you're paginating through users sorted by name and a user changes their name from "Aaron" to "Zach"? There's potential for the same user to show up on both the first and last pages.

I believe that the concept of a "cursor" — which I've seen in other API designs — provides one possible avenue to fixing this. My understanding of this so far is that the first request in a paginated series would return a cursor ID, and the server would store info about that cursor. Subsequent requests can pass the "cursor" value to get the next records in the set based on a snapshot in time. If the data store supports getting the set of records at a particular point in time, then the cursor can store the time when the original request is made and continue to return results in subsequent responses consistent with that initial request time.

Another approach would be to have the client store the timestamp of the initial request and pass that as a `state_at` timestamp field with each subsequent request, instructing the server to return records as they were at that point in time.

### Duplicate values

What if we're requesting users ordered by name and there are many users with the same name? If the page size cuts us off in the middle of a chunk of "Alices", and then we make a subsequent request with `after: 'Alice'`, then we'll miss the rest of the "Alices"!

Spoiler: I never actually built an endpoint that paginated with data like a name field that was likely to have repeated values. It was almost always timestamps that were unlikely to have repeated values at the scale of the application I was building.

One way of solving this would be to have the `before/after` fields accept a primary key for the record instead of the value of the field used in the sort order. For example, instead of `name`, the `last` field of pagination state could return the ID of the last record in the response, and then the `before/after` fields could accept that same user ID.

The challenge that I encountered with that approach is that it was difficult to make the underlying database query efficient, since it wasn't as straightforward to write SQL that says "the next 10 rows in this order after primary key XYZ". But I'm guessing this issue is surmountable with a bit more work, or a different data store.
