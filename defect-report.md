## Defect 1: Deactivated visitors appear in the active visitor list

**Summary:** Deactivated visitors are still returned in the active visitor list.

**Type:** Functional

**Description:**
When a visitor is deactivated, their `active` status is changed to `false`. However, the active visitor list still returns the deactivated visitor. The `index` action in `api/app/controllers/api/visitors_controller.rb` filters visitors only by `checked_out_at: nil` and does not filter by `active: true`. Therefore, a visitor who has been deactivated but has not been checked out is still included in the active visitor list.

**Steps to Reproduce:**

1. Start the Rails API using `rails s`.
2. Create a visitor through `POST /api/visitors` with valid registration details.
3. Note the visitor's `id` from the API response.
4. Send a `GET` request to `/api/visitors` and confirm that the newly created visitor appears in the active visitor list.
5. Send a `PATCH` request to `/api/visitors/:id/deactivate`, replacing `:id` with the visitor's ID.
6. Confirm from the response that the visitor's `active` value is now `false`.
7. Send another `GET` request to `/api/visitors`.
8. Check the returned visitor records.

**Expected Result:**
The deactivated visitor should not be returned by the active visitor list because deactivated visitors must not appear in the active list.

**Actual Result:**
The deactivated visitor is still returned by `GET /api/visitors` even though its `active` value is `false` and its `checked_out_at` value remains `nil`.

## Defect 2: Visitor list performs N+1 host queries

**Summary:** The visitor list loads each visitor's host with a separate database query.

**Type:** Performance

**Description:**
The `index` action in `api/app/controllers/api/visitors_controller.rb` serializes each visitor's `host_name` through the `visitor.host` association. Without eager loading, Active Record performs one additional host query for each visitor returned. A page containing 20 visitors therefore performs one query for the visitor list plus up to 20 additional host queries, instead of loading the hosts in a single query.

**Steps to Reproduce:**

1. Start the Rails API using `rails s`.
2. Ensure there are multiple active visitors with hosts.
3. Send a `GET` request to `/api/visitors?page=1`.
4. Inspect the Rails development log or SQL notifications for the request.
5. Count the database queries used to load the visitors and their hosts.

**Expected Result:**
The visitor list should eager-load the host association and use one query for visitors plus one query for the required hosts, regardless of the number of visitors on the page.

**Actual Result:**
The visitor list performs one host query per visitor while serializing `host_name`. In the observed 20-record page, the request generated 21 Active Record queries, including repeated `Host Load` queries.
