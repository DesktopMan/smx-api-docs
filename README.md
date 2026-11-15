# About

This repo serves as the documentation and issue tracker for my public API for the StepManiaX arcade game.

This API is free for non-commercial use and does not require authentication.

This project is in no way affiliated with Step Revolution, the developers of StepManiaX.

# Endpoints

The base URL for the API is _api.smx.573.no_. The following endpoints are available:

* scores - individual scores
* songs - song metadata
* charts - chart metadata
* players - player profiles
* extra - Step timing data
* machines - live state of a machine (players, difficulties and selected song), see below

# Request format

The request format is the same for all endpoints. You send one request parameter called _q_, which is URL encoded JSON.

Fields are typed, so use strings for string values, integers for integer values, and ISO 8601 for timestamps. Text matches are case insensitive.

* You can filter on any field in the data using dot notation: _"field.name":"value"_
* You can filter on multiple values with an array: _"field.name":["one","two"]_
* You can also use operators:  lt, lte, gt, gte, in, nin (not in), all.
* For array fields, a value matches if any element matches. Use _all_ to require every value:
  _"edit_tags":{"all":["Jumps","Groove"]}_

A request can filter on multiple fields, and you can use multiple operators for a single field.

Clickable examples:

* <https://api.smx.573.no/charts?q={"difficulty":14}>
* <https://api.smx.573.no/charts?q={"difficulty":{"gte":25}}>
* https://api.smx.573.no/players?q={"username":"Auby"}
* <https://api.smx.573.no/scores?q={"score":100000}>
* <https://api.smx.573.no/scores?q={"gamer.username":["Auby","Jokko"]}>
* <https://api.smx.573.no/scores?q={"gamer.username":"Auby","score":{"gte":99725,"lt":100000}}>
* <https://api.smx.573.no/scores?q={"gamer.username":"Auby","updated_at":{"gte":"2024-01-01"},"_order":"asc"}>
* <https://api.smx.573.no/songs?q={"genre":"Eurobeat"}>

# Special fields

You can use these special fields to modify your response in other ways than matching.

* _sort_by - sets the sort field. If you set this value to _random the API will return random results.
  Default is id.
* _order - _asc_ or _desc_. Default is _desc_
* _take - integer (default 100). How many results to return (max 100, or 1000 with `_group_by` or `_sum_by`. Use
  __skip_ to get more)
* _skip - integer (default 0). How many results to skip
* _group_by - Group by a field or multiple fields. Will return the best score for each group
* _group_keep - `best` (default), `first` or `last`. Which score `_group_by` keeps per group: the best, the oldest or
  the newest
* _group_ties - `newest` (default) or `oldest`. When several scores in a group share the best score, which one to keep
* _sum_by - Group by a field or fields and sum the `_sum` fields over each group. Sums over every matching score, or
  with `_group_by` over the scores it keeps (e.g. each player's best score per chart). Each result only has the
  `_sum_by` fields and a `sums` object, e.g. `{"gamer_id":1,"sums":{"score":12345}}`. Add fields like
  `gamer.username` to `_sum_by` to include them. Sorted by the `_sum_by` fields unless `_sort_by` is set
* _sum - Field or fields to sum, required with `_sum_by`. Nested fields are nested in `sums` too (`song.last_ms` is
  returned as `"sums":{"song":{"last_ms":...}}`)
* _having - Filter applied after grouping and summing, using the same syntax as regular filters, e.g.
  `"_having":{"score":{"lte":99000}}` for players whose best score is at most 99000, or
  `"_having":{"sums.score":{"gte":1000000}}`
* _count - _true_ or _false_ (default). Return the result count only as an object with the _count_ field set

_group_by_ is the most powerful of these, so here are some examples to show what you can do:

* Personal bests for a player:  
<https://api.smx.573.no/scores?q={"gamer.username":"Auby","_group_by":"song_chart_id"}>
* Personal bests for two players for a single chart:  
<https://api.smx.573.no/scores?q={"gamer.username":["Auby","jokko"],"song_chart_id":3419,"_group_by":["gamer_id","song_chart_id"]}>
* First clear of each chart for a player:  
<https://api.smx.573.no/scores?q={"gamer.username":"Auby","cleared":true,"_group_by":"song_chart_id","_group_keep":"first"}>
* Total of each player's personal bests on wild charts, highest first:  
<https://api.smx.573.no/scores?q={"chart.difficulty_display":"wild","_group_by":["song_chart_id","gamer_id"],"_sum_by":"gamer_id","_sum":"score","_sort_by":"sums.score"}>
* Total play time per player, in milliseconds:  
<https://api.smx.573.no/scores?q={"gamer.username":"Auby","_sum_by":"gamer_id","_sum":"song.last_ms"}>

# Websockets

The following endpoints support websockets for real time notifications:

* charts - insert and update
* scores - insert
* songs - insert and update

Websockets support matching with the same syntax as the regular endpoints, using the _q_ parameter.
You will only get notifications for data matching your query.
The special fields listed above are not supported with websockets.

Add `heartbeat=<seconds>` (5 to 300) next to _q_ to receive `{"_heartbeat":true}` whenever nothing else has been sent
for that long. Browsers can't see websocket pings, so this is the only way for them to tell a quiet connection from a
dead one: if no message arrives within two intervals, reconnect.
Example: `wss://api.smx.573.no/scores?q={"gamer.username":"Auby"}&heartbeat=30`

**Note:** The _charts_ endpoint tracks play count and pass count.
If you watch this endpoint with no filters you will be notified every time a song is played.

# Machines

A client at a machine can post the machine's state, so that others can follow who is playing what (e.g. a score
browser on a stream showing the current players' scores). This is the only part of the API that accepts writes.

* `GET /machines/<id>` - current state
* `POST /machines/<id>` - set the state. The id is chosen by the client (letters, digits, - and _, up to 64
  characters). Body (max 10 kB):

  ```json
  {"players": [{"username": "Auby", "difficulty": "wild"}, {"username": null, "difficulty": null}],
   "song": {"id": 123, "title": "Song title"}}
  ```

  Player 1 (left) first. `song` may be `null`. Other fields are ignored.
* `wss://api.smx.573.no/machines/<id>` - sends the current state on connect, then every change. Supports `heartbeat`.

# Extra data format

The *offsets* field contains timing information for every arrow stepped by the player.
Missed arrows are not part of the data.

**Note:** Data starts at index 1. Index 0 is always 0.

The *offsets* field is an array with three values for each arrow.

* Time: 0 to 1000, where 0 is the beginning of the song and 1000 is the end.
* Offset: Offset from a perfect step, in milliseconds
* Judgement: 0 is perfect!!, 1 is perfect, 2 is early and 3 is late.

Scores submitted using a QR code and scores older than the game update that added extra data will not have any data,
and will return HTTP 404. If the StepManiaX servers can't be reached, the API returns HTTP 502; try again later.

# Changelog

## 2026-11-15

* Added `_group_keep`, `_group_ties`, `_sum`, `_sum_by` and `_having` special fields
* Added `in` and `all` operators
* `_take` can be up to 1000 for grouped results
* Added /machines endpoint
* Added opt-in websocket heartbeat
* Websocket matching now works like regular requests: missing fields match `nin`, lists of strings are case
  insensitive, and booleans and the `in` operator work
* Fixed websocket connections not being cleaned up after disconnecting
* Fixed `_take` being used as a filter with `_export_all`
* /extra retries once when StepManiaX drops the connection, and returns HTTP 502 instead of 404 if it still fails

## 2026-11-13

* Updated song, chart and player profile data on all scores
* Going forward, scores will always return the latest song, chart and player profile data

## 2026-11-10

* Fixed player lookup by id always returning an empty result

## 2025-04-21

* Added __count_ special field
* Added HTTP 400 response for bad requests
* Added HTTP 500 response for internal server errors

## 2025-02-19

* Added /extra endpoint
* Added personal bests to websocket scores

## 2025-02-05

* Fixed issue with missing websocket messages

## 2025-01-21

* Fixed issue when grouping by nested fields

## 2024-12-26

* First public release
* Added websocket support for charts and songs
* renamed _params_ parameter to _q_
* renamed __sort_ field to __sort_by_
