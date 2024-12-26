# About

This repo serves as the documentation and issue tracker for my public API for the StepManiaX arcade game.

This API is free for non-commercial use and does not require authentication.

This project is in no way affiliated with Step Revolution, the developers of StepManiaX.

# Endpoints

The base URL for the API is _api.smx.573.no_. The following endpoints are available:

* charts - chart metadata
* players - player profiles
* scores - individual scores
* songs - song metadata

# Request format

The request format is the same for all endpoints. You send one request parameter called _q_, which is URL encoded JSON.

Fields are typed, so use strings for string values, integers for integer values, and ISO 8601 for timestamps. Text matches are case insensitive.

* You can filter on any field in the data using dot notation: _"field.name":"value"_
* You can filter on multiple values with an array: _"field.name":["one","two"]_
* You can also use operators:  lt, lte, gt, gte, nin (not in).

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
* _take - how many results to return (max 100, use __skip_ to get more)
* _skip - how many results to skip
* _group_by - Group by a field or multiple fields. Will return the best score for each group

_group_by_ is the most powerful of these, so here are some examples to show what you can do:

* Personal bests for a player:  
<https://api.smx.573.no/scores?q={"gamer.username":"Auby","_group_by":"song_chart_id"}>
* Personal bests for two players for a single chart:  
<https://api.smx.573.no/scores?q={"gamer.username":["Auby","jokko"],"song_chart_id":3419,"_group_by":["gamer_id","song_chart_id"]}>

# Websockets

The following endpoints support websockets for real time notifications:

* charts - insert and update
* scores - insert
* songs - insert and update

Websockets support matching with the same syntax as the regular endpoints, using the _q_ parameter.
You will only get notifications for data matching your query.
The special fields listed above are not supported with websockets.

**Note:** The _charts_ endpoint tracks play count and pass count.
If you watch this endpoint with no filters you will be notified every time a song is played.

# Changelog

## 2024-12-26

* First public release
* Added websocket support for charts and songs
* renamed _params_ parameter to _q_
* renamed __sort_ field to __sort_by_