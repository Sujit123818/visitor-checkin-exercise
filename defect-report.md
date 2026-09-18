## Defect 1: Deactivated visitors appear in the active visitor list

**Summary:** Deactivated visitors are returned by the active visitor list.

**Type:** Functional

**Description:**
The active visitor endpoint must exclude visitors whose `active` flag is `false`. The original `index` action filtered only on `checked_out_at: nil`, so a deactivated visitor who had not been checked out remained in the active list.

**Steps to Reproduce:**

1. Start the Rails API with `rails s`.
2. Create a visitor through `POST /api/visitors` with valid registration details.
3. Note the visitor ID from the response.
4. Send `GET /api/visitors` and confirm that the visitor is listed.
5. Send `PATCH /api/visitors/:id/deactivate` for that visitor.
6. Send `GET /api/visitors` again.

**Expected Result:**
The deactivated visitor is absent from the active visitor list.

**Actual Result:**
The original implementation returned the visitor because `checked_out_at` remained `nil`, even though `active` was `false`.

## Defect 2: Deactivated visitors appear in repeat-visit search

**Summary:** Deactivated visitors can be selected from the repeat-visit search results.

**Type:** Functional

**Description:**
The repeat-visit search queries visitors by name but does not restrict results to active records. This allows a deactivated visitor to be returned and selected for a new visit, contrary to the requirement that deactivated visitors must not be selectable for repeat visits.

**Steps to Reproduce:**

1. Start the Rails API with `rails s`.
2. Create or locate a visitor whose `active` value is `false`.
3. Send `GET /api/visitors/search?q=<part-of-the-deactivated-visitor-name>`.
4. Inspect the returned results.

**Expected Result:**
Deactivated visitors are excluded from the search results.

**Actual Result:**
The original implementation returned matching visitors regardless of their `active` value.

## Defect 3: Incomplete visitor records can be created

**Summary:** The API accepts registrations with missing required visitor details.

**Type:** Data

**Description:**
Visitor registration requires a full name, company name, host employee, and visit purpose. The `Visitor` model has no presence validations, and the database columns are nullable. Therefore, the API can persist incomplete records instead of rejecting them.

**Steps to Reproduce:**

1. Start the Rails API with `rails s`.
2. Send `POST /api/visitors` with an empty JSON body or omit one or more required fields.
3. Inspect the response and query the visitor list.

**Expected Result:**
The API returns `422 Unprocessable Entity` with validation errors and does not create a visitor record.

**Actual Result:**
The original implementation returned `201 Created` and persisted a record with missing fields.

## Defect 4: Visitor list performs N+1 host queries

**Summary:** The visitor list loads each visitor's host with a separate database query.

**Type:** Performance

**Description:**
The visitor serializer includes `host_name` through the `visitor.host` association. Without eager loading, Active Record performs one host query per visitor returned. A page of 20 visitors therefore performs one query for the visitor list plus up to 20 additional host queries.

**Steps to Reproduce:**

1. Start the Rails API with `rails s`.
2. Ensure there are multiple active visitors with hosts.
3. Send `GET /api/visitors?page=1`.
4. Inspect the Rails development log or SQL notifications for the request.
5. Count the visitor and host queries.

**Expected Result:**
The endpoint eager-loads hosts and uses one visitor query plus one host query for the page, rather than one host query per visitor.

**Actual Result:**
The original implementation generated a host query while serializing each visitor. The observed 20-record request generated 21 Active Record queries, including repeated `Host Load` queries.

## Defect 5: Check-in times are not displayed in the receptionist's local timezone

**Summary:** Visitor check-in times are displayed as UTC instead of Asia/Kathmandu local time.

**Type:** Functional

**Description:**
The application is intended to operate in the `Asia/Kathmandu` timezone. Rails does not configure that timezone, and the frontend formats timestamps with `toISOString()`, which converts them to UTC before displaying the hour and minute.

**Steps to Reproduce:**

1. Run the API and web application.
2. Register a visitor or use an existing active visitor.
3. Compare the `checked_in_at` value with the time shown in the active visitor list.
4. Perform the comparison at a time when Kathmandu differs from UTC, such as during normal working hours.

**Expected Result:**
The displayed check-in time matches the receptionist's local Asia/Kathmandu time.

**Actual Result:**
The frontend uses UTC time from `toISOString()`, so the displayed time is offset from Kathmandu by 5 hours and 45 minutes.

## Defect 6: Registration failures are treated as successful submissions

**Summary:** The frontend clears the registration form after a failed API response.

**Type:** Functional

**Description:**
The API helper returns `null` for non-success responses, but the registration form does not inspect that result. It always clears the form and calls the refresh callback, so users receive no error feedback and may lose their entered data when registration fails.

**Steps to Reproduce:**

1. Start the API and web application.
2. Make the registration request fail, for example by stopping the API or submitting data rejected by the API.
3. Submit the registration form.
4. Observe the form and visitor list.

**Expected Result:**
The form remains populated, an error is shown, and the visitor list is refreshed only after a successful registration.

**Actual Result:**
The form is cleared and the list refresh callback runs even though the API request returned a failure response.

## Defect 7: Checkout failures are hidden after optimistic removal

**Summary:** The frontend removes a visitor before confirming that checkout succeeded.

**Type:** Functional

**Description:**
When the receptionist clicks Check Out, the frontend immediately removes the row from the local list and does not inspect the result of the checkout request. If the API request fails, the visitor disappears from the screen even though the server still considers the visitor active.

**Steps to Reproduce:**

1. Start the web application with the API unavailable or otherwise force `PATCH /api/visitors/:id/check_out` to fail.
2. Open the active visitor list.
3. Click Check Out for a visitor.
4. Restore the API and reload the list.

**Expected Result:**
The visitor remains visible, or the UI reports the failure and restores the row when checkout does not succeed.

**Actual Result:**
The row is removed immediately, no error is shown, and a later refresh reveals that checkout did not actually occur.

## Assumptions and Unspecified Behavior

The report does not classify duplicate registrations, empty search queries, invalid page numbers, or the exact response shape for validation errors as defects because the provided requirements do not define their behavior.


Note: I used AI to write this description more accurately and precisely.