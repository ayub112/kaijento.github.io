---
title: "JSON parsing: jq google analytics api"
date:  2017-03-23 22:51:45 +0100
tags:
 - jq
 - json
---

Given a JSON response from the google analytics API we want to
extract the `rows` key and filter out any URLs that contain
the string `eyeblaster`.

`google.json`

```json
{
    "columnHeaders": [
        {
            "columnType": "DIMENSION",
            "dataType": "STRING",
            "name": "rt:pagePath"
        },
        {
            "columnType": "METRIC",
            "dataType": "INTEGER",
            "name": "rt:pageviews"
        }
    ],
    "id": "https://www.googleapis.com/analytics/v3/data/realtime?ids=ga:XXXXX",
    "kind": "analytics#realtimeData",
    "profileInfo": {
        "accountId": "XXXX",
        "internalWebPropertyId": "61947021XXXX",
        "profileId": "XXXX",
        "profileName": "home: Sites only",
        "tableId": "realtime:XXXX",
        "webPropertyId": "UA-XXXXX-X"
    },
    "query": {
        "dimensions": "rt:pagePath",
        "filters": "rt:pagePath=@404.html",
        "ids": "ga:XXXXX",
        "max-results": 1000,
        "metrics": [
            "rt:pageviews"
        ],
        "sort": [
            "-rt:pageviews"
        ]
    },
    "rows": [
        [
            "/404.html?page=/eyeblaster/addineyeV2-secure.html&from=http://www.google.com/",
            "2"
        ],
        [
            "/404.html?page=/amobee/a3d-ad-loader.html?",
            "1"
        ],
        [
            "/404.html?page=/amobee/a3d-ad-loader.html",
            "1"
        ],
        [
            "/404.html?page=/amobee/a3d-ad-loader.html?a3dWebglBanner=https",
            "1"
        ]
    ],
    "selfLink": "https://www.googleapis.com/analytics/v3/data/realtime?ids=",
    "totalResults": 4,
    "totalsForAllResults": {
        "rt:pageviews": "5"
    }
}
```

So we can use `.rows[]` to get at each item inside the rows array:

```bash
$ jq -c '.rows[]' google.json
["/404.html?page=/eyeblaster/addineyeV2-secure.html&from=http://www.google.com/","2"]
["/404.html?page=/amobee/a3d-ad-loader.html?","1"]
["/404.html?page=/amobee/a3d-ad-loader.html","1"]
["/404.html?page=/amobee/a3d-ad-loader.html?a3dWebglBanner=https","1"]
```

We want to select out certain items so we can use the aptly named `select()`. Let's start
by finding elements that contain the string `eyeblaster`.

```bash
$ jq -c '.rows[] | select(.[0] | index("eyeblaster"))' google.json
["/404.html?page=/eyeblaster/addineyeV2-secure.html&from=http://www.google.com/","2"]
```

`index()` will return null if there is no match found. So we're almost there, but 
we need the elements that do `not` match:

```bash
$ jq -c '.rows[] | select(.[0] | index("eyeblaster") | not)' google.json
["/404.html?page=/amobee/a3d-ad-loader.html?","1"]
["/404.html?page=/amobee/a3d-ad-loader.html","1"]
["/404.html?page=/amobee/a3d-ad-loader.html?a3dWebglBanner=https","1"]
```

So `not` inverts the logic giving us what we need. 

```bash
$ echo true | jq not
false
$ echo ♥ jq
♥ jq
```
