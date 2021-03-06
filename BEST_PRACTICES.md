Best Practices
--------------

Rate Limiting
=============

Most APIs enforce rate limits. Taps should be written in a way that respects these rate limits.

Rate limits can take the form of quotas (X requests per day), in which case the tap should leave
room for other use of the API, or short-term limits (X requests per Y seconds). For short-term
limits, the entire quota can be used up and the tap should sleep when rate limited.

The singer-python library's utils namespace has a rate limiting decorator for use.

User Agents
===========

You should support an optional `user_agent` field in config that will be passed along in request
headers to the API. The `user_agent` should include an email address to allow the API provider to
contact you if there's an issue with the tap or your usage of their API.

Memory Constraints
==================

Taps should not rely on large volumes of RAM being available during run and should strive to keep
as little data in memory as required. Use iterators/generators when available.


Dates
=====

All dates should use the RFC3339 format (which includes timezone offsets). UTC is the preferred
timezone for all data and should be used when possible.

Good:
 - 2017-01-01T00:00:00Z (January 1, 2017 12:00:00AM UTC)
 - 2017-01-01T00:00:00-05:00 (January 1, 2017, 12:00:00AM America/New_York)

Bad:
 - 2017-01-01 00:00:00


Start Date
==========

Taps should support an optional `start_date` param that indicates how far back an integration
should sync data in the absence of a state entry.


State
=====

States that are datetimes will all be in RFC3339 format. Using ids or other identifiers for state
is possible but could cause issues if the entities are not immutable and change out of order. If
the API supports filtering by created timestamps but is not immutable (meaning a record can be
updated after creation), DO NOT use the created timestamp for saving state. States using created
timestamps or ids are a sure way to have data discrepencies.

State as early and often as possible, but no sooner and no more often than is required. When a
state is written, no data prior to that state will be synced, so do not update the state until all
possible exceptions will be thrown.

Endpoints that do not support filtering by updated timestamps or include updated timestamps do not
support saving state.

If the API supports filtering by updated timestamp, use that for filtering. If the API doesn't
support filtering but does return updated timestamps with the data, filter by the timestamp before
streaming the data.

When streaming data in, stream the data in ascending order when possible.

Keep in mind that jobs can be interrupted at any point. States should never be in an invalid
state. Interrupted jobs that save state too early will have data missing. Interrupted jobs that
save state too late will cause an increase in duplicate rows being replicated.

The tap's config file must ALWAYS have a `start_date` field indicating the default state.


Logging and Exception Handling
==============================

Log every URL + params that will be requested, but be sure to exclude any sensitive information
such as api keys and client secrets. Log the progress of the sync (e.g. Starting entity 1,
Starting entity 2, etc.) When the API returns with an error, log the status code and body.

Allow exceptions to be bubbled up and interrupt the tap. DO NOT wrap the tap code in one large
try/except block and log the exception message. The stack trace is much more useful than the error
message.

If an intermittent error is detected from the API, retry using an exponential backoff (try using
`backoff` library for Python). If an unrecoverable error is detected, exit the script with a
non-zero error code (raise an exception or use `sys.exit(1)`)


Module Structure
================

Source code should be in a module (folder with `__init__.py` file) and not just a script (`module.py`
file).


Schemas
=======

Schemas should be stored in a `schemas` folder under the module directory as JSON files rather than
as native dicts in the source code.

Always stream the entity's schema before streaming any records for that entity.

Use `singer-tools`'s `singer-infer-schema` to generate a base schema to start from.


Testing
=======

Use `singer-tools`'s `singer-check-tap` command to validate that a tap's output matches the expected
format.

Use the `target-stitch` module in dry-run mode (`tap-mytap -c myconf.json | target-stitch -n`) to
validate records against their schemas.
