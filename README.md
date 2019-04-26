# REST API v4
## For a Free API Key visit us at our [People Data Labs Website](https://www.peopledatalabs.com)

### Overview 

This document describes the People Data Labs v4 REST API. Canonical datasets used to supplement the person dataset can be found in the [data directory](https://github.com/peopledatalabs/docs/tree/master/data). Example profiles and API responses can be found in the [examples directory](https://github.com/peopledatalabs/docs/tree/master/examples), and their corresponding schemas can be found in the [schemas directory](https://github.com/peopledatalabs/docs/tree/master/schemas). 

If you have any problems or requests, please contact your account manager, or email us at <a href="mailto:support@peopledatalabs.com">support@peopledatalabs.com</a> 

1. [Overview](#overview) 
1. [Introduction](#introduction) 
1. [Endpoint](#endpoint) 
1. [Versioning](#versioning) 
1. [Authentication](#authentication) 
1. [Rate Limiting](#rate-limiting) 
1. [Requests](#requests) 
1. [Parameters](#parameters) 
1. [Required Parameter](#required-parameter) 
1. [Likelihood](#likelihood) 
1. [Response](#response) 
1. [Errors](#errors)
1. [Bulk Endpoint](#bulk-endpoint) 


### Introduction  

The API is designed to enrich information on a single person. 

The API is organized around [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) and uses HTTP response codes to indicate API errors. All API responses, including errors, are returned in [JSON](http://www.json.org). 


### Endpoint  

The API resides at `api.peopledatalabs.com`. All API requests must be made over [HTTPS](https://en.wikipedia.org/wiki/HTTPS). Calls made over plain HTTP will fail. API requests without authentication will also fail.

```curl 
https://api.peopledatalabs.com
```

The `v4` API can be used to enrich data on a person:

```curl 
curl -X GET \
  'https://api.peopledatalabs.com/v4/person?profile=linkedin.com/in/seanthorne'
```

Or data on 1-100 persons in a single request (recommended):

```curl
curl -X POST "https://api.peopledatalabs.com/v4/person/bulk" -H 'Content-Type: application/json' -d'
{
    "requests": [
    	{
    		"params": {
    			"profile": ["linkedin.com/in/seanthorne"]
    		}
    	}
    ]
}'
``` 

For high volume usage, we recommend the 2nd option. 

### Versioning 

API versions are specified in the url. Older versions of the API will continue to be supported. 
 
When we make backwards-incompatible changes to the API or dataset, we'll create a new version. The most up to date version of the API is `v4`. The next version of the API will be `v5`. We perform batch data builds on a monthly basis. We don't perform any streaming updates to the dataset between data builds. Each version of the API will use the most recent, up-to-date data build.  

As can be seen in the [Example Person Data Model](https://github.com/peopledatalabs/docs/blob/master/examples/person.json), all data fields are stored as a list of objects. Data builds are ran approximately every 2 months. After a data build is ran, barring any significant update that breaks existing functionality in the API, the newly built data will be returned in existing versions of the API. Therefore, the dataset is not completely static; the number of fields/data points a specific profile has may increase/decrease through time.    

In a single API version, the number of canonical values for `phone_numbers.type`, `locations.type`, `emails.type`, `profiles.network`, `industries.name`, and `locations` may increase, but the original values will not change. For example, in `v4`, a github profile in the `profiles` field's `network` field will always have the value `github`, yet if a social network which is not currently in the dataset, e.g. `yelp.com` is added, `yelp` would be added as a potential to the [list of canonical social network types](https://github.com/peopledatalabs/docs/blob/master/data/profiles_network.json). The most up-to-date list of canonical values for these fields can always be found in [github.com/peopledatalabs/docs/tree/master/data](https://github.com/peopledatalabs/docs/tree/master/data)    

On the other hand, the canonical `education.school` data, and the canonical `experience.company` data may change slightly within the span of a single API version. 



### Authentication 

There are two ways to authenticate requests to `v4`.  

**In URL** 

```curl  
curl -X GET \
  'https://api.peopledatalabs.com/v4/person?api_key=xxxx'
``` 

**In Header** 

```curl  
curl -X GET \
  'https://api.peopledatalabs.com/v4/person' \ 
  -H 'X-Api-Key: xxxx'
``` 

Requests made with an invalid API Key will return a `401` error. If you need any assistance here, would like to request a new API key, or delete your existing key, please contact your account manager or reach out to us at <a href="mailto:support@peopledatalabs.com">support@peopledatalabs.com</a>


### Rate Limiting

Rate limits are defined on a per-minute basis. We use a fixed-window rate limiting strategy, so if your API key's rate limit is `5000` requests per minute, those `5000` api calls can be made at any interval within the 60 second window. 


```curl 
curl -i -X GET \
  'https://api.peopledatalabs.com/v4/person' \
  -H 'X-Api-Key: xxxx'
  
HTTP/2 404 
date: Tue, 16 Jan 2018 17:38:18 GMT
content-type: application/json
content-length: 4731
server: nginx/1.11.13
x-totallimit-limit: 10000000
x-totallimit-remaining: 9723340
x-ratelimit-limit: 10000
x-ratelimit-remaining: 9999
x-ratelimit-reset: 1516124358
retry-after: 59
```  


| Header Name | Description | 
|-----------|---------| 
| `x-totallimit-limit` | The maximum number of API requests which return a `200` you're able to make. |
| `x-totallimit-remaining` | The number of API requests which return a `200` you have remaining |
| `x-ratelimit-limit` | The maximum number of requests you're permitted to make per minute. | 
| `x-ratelimit-remaining` | The number of requests remaining in the current rate limit window. |
| `x-ratelimit-reset` |     The time at which the current rate limit window resets in UTC epoch seconds. |
| `retry-after` | The number of seconds left until the current rate limit window resets. |

If your account has a limit on the number of `200` API calls you're able to make, once `x-totallimit-remaining` reaches 0, all succeeding api requests will return `403` errors, and once `x-ratelimit-remaining` reaches 0, all succeeding requests made will return `429` errors, until the current rate limit window resets. If you'd like to increase your account's `x-totallimit-limit` or `x-ratelimit-limit`, please contact your account manager or reach out to us at <a href="mailto:support@peopledatalabs.com">support@peopledatalabs.com</a>

 
### Requests 

When a matching person is returned, the HTTP Response code will be `200`, and when no matching person is found or returned, the HTTP Response code will be a `404`. 

When an API request is executed, the queried data points are preprocessed and built into a query, which is then executed against our production dataset. If the query yields 1 or more matching persons from the dataset, the person returned in the API response is the one who is most likely to be the same person as the person requested. This degree of confidence is represented by the `likelihood` field in the API response. The `likelihood` field is an integer between `0` and `10` that represents how confident we are the person returned is the same as the person requested. The minimum likelihood score a response must possess in order to return a `200` can be controlled in the api request using the `min_likelihood` param, [described in the parameters section below](#parameters). 

Query parameters should be separated by an ampersand `&`. 

```curl
GET /v4/person?api_key=xxxx&min_likelihood=6&required=emails AND experience&profile=http://linkedin.com/in/seanthorne 
```

```curl 
GET /v4/person?api_key=xxxx&company=People Data Labs&company=Hallspot&email=sean.thorne@talentiq.co 
```

### Parameters  

Data points on the queried person are added as key/value pairs to the query string of a `v4` request.  

A number of key/values can also be added to the query string of a `v4` request to control the data points a response must contain to be returned as a `200`, and also to control certain parts of the API response schema.  

All query parameters listed below are optional. 

**Data Parameters** 

The following parameters can be used to specify information on the requested person. Adding more data points to a request increases the probability of a `200` response, and further, will increase the accuracy of the response's likelihood score.

| Parameter Name | Description | Example |
|-----------|---------|---------| 
| `name` | The person's full name, at least first and last. | `Jennifer C. Jackson` |
| `first_name` | The person's first name | `Jennifer` |
| `last_name` | The person's last name | `Jackson` |
| `middle_name` | The person's middle  | `Cassandra` |
| `location` | The location in which a person lives. Can be anything from a street address to a country name | `Medford, OR USA` |
| `locality` | A locality in which the person lives | `Boise` |
| `region` | A state or region in which the person lives | `Idaho` |
| `country` | A country in which the person lives | `United States` |
| `company` | A name, website, or social url of a company where the person has worked | `Amazon Web Services` |
| `school` | A name, website, or social url of a university or college the person has attended | `university of iowa` |
| `phone` | A phone number the person has used | `+1 541-672 2910` |
| `email` | An email the person has used | `renee.c.paulsen1959@yahoo.com` |   
| `email_hash` | A sha256 email hash | `e206e6cd7fa5f9499fd6d2d943dcf7d9c1469bad351061483f5ce7181663b8d4 ` |  
| `profile` | A social profile the person has used. [List of available social profiles](https://github.com/peopledatalabs/docs/blob/master/data/profiles_network.json) | `https://linkedin.com/in/seanthorne` |  

The minimum combination of data points a request must contain in order to have a possibility of returning a `200` response are:  

```curl 
profile OR email OR phone OR ( 
    (
        (first_name AND last_name) OR name) AND 
        (locality OR region OR company OR school OR location)
    )
```


**Examples** 


```curl 
curl -X GET -G \
  'https://api.peopledatalabs.com/v4/person' \
  -H 'X-Api-Key: xxxx' \ 
  --data-urlencode 'email=sean@talentiq.co' \
  --data-urlencode 'email=sean@peopledatalabs.com' \ 
  --data-urlencode 'name=Sean Thorne'

``` 

```curl 
curl -X GET -G \
  'https://api.peopledatalabs.com/v4/person' \
  -H 'X-Api-Key: xxxx' \ 
  --data-urlencode 'company=TalentIQ Technologies' \
  --data-urlencode 'first_name=Sean' \ 
  --data-urlencode 'last_name=Thorne' \
  --data-urlencode 'school=University of Oregon' 

```

```curl 
curl -X GET -G \
  'https://api.peopledatalabs.com/v4/person' \
  -H 'X-Api-Key: xxxx' \ 
  --data-urlencode 'name=Sean Thorne' \ 
  --data-urlencode 'location=SF Bay Area' \
  --data-urlencode 'profile=www.twitter.com/seanthorne5' \
  --data-urlencode 'phone=1 503-2353497' 

```


**Response Filtering Parameters** 

These parameters can be used to describe the characteristics an API response must posess to return a `200`. 

| Parameter Name | Description | Default |
|-----------|---------|---------| 
| `min_likelihood` | The minimum `likelihood` score a response must have to return a `200` | `0` |
| `required` | Parameter specifying the fields and data points a response must have to return a `200` | |

For more information on the `min_likelihood` param, see the [likelihood](#likelihood) section, and for more information on the `required param`, see the [required param](#required-parameter) section. 


**Response Formatting Parameters** 

The following parameter can be used to control/specify certain things about the formatting of the person data in the API response. 

| Parameter Name | Description | Default |
|-----------|---------|---------| 
| `titlecase` | All text in the data of API responses is returned as lowercase by default. Setting `titlecase` to `true` will titlecase the person data in `200` responses | `false` |  

 
 
### Required Parameter  


The required parameter ensures that you only get charged for responses which have the data fields you're interested in. The value is formatted as a boolean statements

Response must contain an email

```curl 
required=emails
```
 
Response must contain a linkedin url 

```curl 
required=profiles.network:linkedin 
``` 

Response must contain experience and emails

```curl 
required=experience AND emails  
``` 

Response must contain experience or emails

```curl 
required=experience OR emails  
``` 

Response must contain education and (emails or phone_numbers)

```curl 
required=education AND (emails OR phone_numbers)
``` 

Fields which can be filtered/required are limited to:  

* `education`
* `education.school`
* `education.school.name`
* `emails`
* `emails.type`
* `experience`
* `experience.company`
* `experience.company.name`
* `locations`
* `locations.name`
* `locations.street_address`
* `names`
* `names.clean`
* `phone_numbers`
* `phone_numbers.e164`
* `profiles`
* `profiles.clean`
* `profiles.ids`
* `profiles.network`

### Likelihood 

The likelihood score gives you the ability to control match precision vs. recall. Increasing the `min_likelihood` score will result in lower match rates, yet less false positive `200` responses. For use cases which rely on a high degree of data accuracy, only records with a likelihood of approximately `8` or above should be used. By default, match recall is kept very high, so a response which returns a likelihood score of `2` will roughly have just a 10 to 30 percent chance of being the same person as the one requested. Adding more data points to your requests will increase the probability of a `200` response returning a higher likelihood score. 

Requests made with only a few data points, e.g. a name and location, will rarely return a `200` response with a likelihood score > 5, and requests made with just an email will rarely return a `200` response with a likelihood score > 7. 


### Response


**200 Response Fields**  

| Field Name | Type | Description |
|-----------|---------|---------| 
| `data` | object | The person response object |  
| `status` | integer | HTTP Status Code |  
| `likelihood` | integer | Likelihood Score |  
 
 
**Example 200 Response**  

```json 
{
  "status": 200,
  "metadata": {
    "name": "",
    "request_id": "a1ab1b49-b640-4699-a135-06a23f2ea103"
  },
  "data": {
    "id": "phewewgXRKygpg8wSUrt",
    "birth_date_fuzzy": "1990",
    "birth_date": null,
    "gender": "male",
    "primary": {
      "job": {
        "company": {
          "name": "people data labs",
          "founded": "2015",
          "industry": "information technology and services",
          "location": {
            "locality": "san francisco",
            "region": "california",
            "country": "united states"
          },
          "profiles": [
            "linkedin.com/company/peopledatalabs",
            "linkedin.com/company/1640694639"
          ],
          "website": "peopledatalabs.com",
          "size": "11-50"
        },
        "locations": [],
        "end_date": null,
        "start_date": "2015-03",
        "title": {
          "levels": [
            "owner",
            "c"
          ],
          "name": "co-founder & ceo",
          "functions": [
            "founder",
            "chief executive officer"
          ]
        },
        "last_updated": "2019-02-01"
      },
      "location": {
        "name": "san francisco, california, united states",
        "locality": "san francisco",
        "region": "california",
        "country": "united states",
        "last_updated": "2019-02-01",
        "continent": "north america"
      },
      "name": {
        "first_name": "sean",
        "middle_name": null,
        "last_name": "thorne",
        "clean": "sean thorne"
      },
      "industry": "computer software",
      "personal_emails": [
        "seanfongthorne@gmail.com"
      ],
      "linkedin": "linkedin.com/in/seanthorne",
      "work_emails": [
        "sean.thorne@talentiq.co",
        "sean@talentiq.co",
        "sean@peopledatalabs.com"
      ],
      "other_emails": [
        "sthorne@uoregon.edu",
        "sean@hallspot.com",
        "sean@datemyschool.com"
      ]
    },
    "profiles": [
      {
        "network": "linkedin",
        "ids": [
          "145991517"
        ],
        "clean": "linkedin.com/in/seanthorne",
        "aliases": [],
        "username": "seanthorne",
        "is_primary": true,
        "url": "http://www.linkedin.com/in/seanthorne"
      },
      {
        "network": "klout",
        "ids": [],
        "clean": "klout.com/seanthorne5",
        "aliases": [],
        "username": "seanthorne5",
        "url": "http://www.klout.com/seanthorne5"
      },
      {
        "network": "twitter",
        "ids": [],
        "clean": "twitter.com/seanthorne5",
        "aliases": [],
        "username": "seanthorne5",
        "url": "http://www.twitter.com/seanthorne5"
      },
      {
        "network": "aboutme",
        "ids": [],
        "clean": "about.me/sean_thorne",
        "aliases": [],
        "username": "sean_thorne",
        "url": "http://www.about.me/sean_thorne"
      },
      {
        "network": "medium",
        "ids": [],
        "clean": "medium.com/@seanthorne5",
        "aliases": [],
        "username": "@seanthorne5",
        "url": "http://www.medium.com/@seanthorne5"
      },
      {
        "network": "angellist",
        "ids": [],
        "clean": "angel.co/sean-thorne-1",
        "aliases": [],
        "username": "sean-thorne-1",
        "url": "http://www.angel.co/sean-thorne-1"
      },
      {
        "network": "gravatar",
        "ids": [],
        "clean": "gravatar.com/seanthorne5",
        "aliases": [],
        "username": "seanthorne5",
        "url": "http://www.gravatar.com/seanthorne5"
      },
      {
        "network": "linkedin",
        "ids": [],
        "clean": "linkedin.com/in/sean-thorne-9b9a854",
        "aliases": [
          "linkedin.com/pub/sean-thorne/40/a85/9b9"
        ],
        "username": "sean-thorne-9b9a854",
        "is_primary": false,
        "url": "http://www.linkedin.com/in/sean-thorne-9b9a854"
      },
      {
        "network": "angellist",
        "ids": [],
        "clean": "angel.co/475041",
        "aliases": [],
        "username": "475041",
        "url": "http://www.angel.co/475041"
      }
    ],
    "emails": [
      {
        "address": "sthorne@uoregon.edu",
        "type": null,
        "sha256": "e206e6cd7fa5f9499fd6d2d943dcf7d9c1469bad351061483f5ce7181663b8d4",
        "domain": "uoregon.edu",
        "local": "sthorne"
      },
      {
        "address": "sean.thorne@talentiq.co",
        "type": "current_professional",
        "sha256": "92e0b06495f24c1cb453b6d2a4d3374c3052fa9a87faf14ad8841ca7e92a586f",
        "domain": "talentiq.co",
        "local": "sean.thorne"
      },
      {
        "address": "sean@talentiq.co",
        "type": "current_professional",
        "sha256": "312680c9c9dacac4307197777433e39f255ce7175c6085073aedc95c1e750bd6",
        "domain": "talentiq.co",
        "local": "sean"
      },
      {
        "address": "sean@hallspot.com",
        "type": "professional",
        "sha256": "f61d233a1e3bf232f63c322a6d52ed692f69ec3a9585140302cd32a50027d704",
        "domain": "hallspot.com",
        "local": "sean"
      },
      {
        "address": "sean@datemyschool.com",
        "type": "professional",
        "sha256": "ced7fdad56974dfbb8d9ccde17b5437e776ee3ca19c283b2ff940965dfb4f570",
        "domain": "datemyschool.com",
        "local": "sean"
      },
      {
        "address": "sean@peopledatalabs.com",
        "type": "current_professional",
        "sha256": "138ea1a7076bb01889af2309de02e8b826c27f022b21ea8cf11aca9285d5a04e",
        "domain": "peopledatalabs.com",
        "local": "sean"
      },
      {
        "address": "seanfongthorne@gmail.com",
        "type": "personal",
        "sha256": "fe2892a9597b2c46939f4798263eac1cfdeff7debead9c637dd5a9b2a8da355e",
        "domain": "gmail.com",
        "local": "seanfongthorne"
      }
    ],
    "phone_numbers": [
      {
        "E164": "+14155688415",
        "number": "+14155688415",
        "type": null,
        "country_code": "1",
        "national_number": "4155688415",
        "area_code": "415"
      }
    ],
    "names": [
      {
        "first_name": "sean",
        "last_name": "thorne",
        "suffix": null,
        "middle_name": null,
        "middle_initial": null,
        "name": "sean thorne",
        "clean": "sean thorne",
        "is_primary": true
      }
    ],
    "locations": [
      {
        "name": "san francisco, california, united states",
        "locality": "san francisco",
        "region": "california",
        "subregion": "city and county of san francisco",
        "country": "united states",
        "continent": "north america",
        "type": "locality",
        "geo": "37.77,-122.41",
        "postal_code": null,
        "zip_plus_4": null,
        "street_address": null,
        "address_line_2": null,
        "most_recent": true,
        "is_primary": true,
        "last_updated": "2019-02-01"
      }
    ],
    "experience": [
      {
        "company": {
          "name": "people data labs",
          "size": "11-50",
          "founded": "2015",
          "industry": "information technology and services",
          "location": {
            "locality": "san francisco",
            "region": "california",
            "country": "united states"
          },
          "profiles": [
            "linkedin.com/company/peopledatalabs",
            "linkedin.com/company/1640694639"
          ],
          "website": "peopledatalabs.com"
        },
        "locations": [],
        "end_date": null,
        "start_date": "2015-03",
        "title": {
          "levels": [
            "owner",
            "c"
          ],
          "name": "co-founder & ceo",
          "functions": [
            "founder",
            "chief executive officer"
          ]
        },
        "type": null,
        "is_primary": true,
        "most_recent": true,
        "last_updated": "2019-02-01"
      },
      {
        "company": {
          "name": "talentiq technologies",
          "size": "1-10",
          "founded": "2015",
          "industry": "computer software",
          "location": {
            "locality": "san francisco",
            "region": "california",
            "country": "united states"
          },
          "profiles": [
            "linkedin.com/company/6404208",
            "linkedin.com/company/talentiq"
          ],
          "website": "talentiq.co"
        },
        "locations": [],
        "end_date": null,
        "start_date": "2015-03",
        "title": {
          "levels": [
            "owner"
          ],
          "name": "co-founder",
          "functions": [
            "founder"
          ]
        },
        "type": null,
        "is_primary": false,
        "most_recent": false,
        "last_updated": null
      },
      {
        "company": {
          "name": "hallspot",
          "size": "1-10",
          "founded": "2013",
          "industry": "computer software",
          "location": {
            "locality": "portland",
            "region": "oregon",
            "country": "united states"
          },
          "profiles": [
            "linkedin.com/company/hallspot",
            "linkedin.com/company/3019184"
          ],
          "website": "hallspot.com"
        },
        "locations": [],
        "end_date": "2015-02",
        "start_date": "2012-08",
        "title": {
          "levels": [
            "owner"
          ],
          "name": "co-founder",
          "functions": [
            "founder"
          ]
        },
        "type": null,
        "is_primary": false,
        "most_recent": false,
        "last_updated": null
      },
      {
        "company": {
          "name": "datemyschool",
          "size": "1-10",
          "founded": "2011",
          "industry": "internet",
          "location": {
            "locality": "brooklyn",
            "region": "new york",
            "country": "united states"
          },
          "profiles": [
            "linkedin.com/company/1977175",
            "linkedin.com/company/datemyschool"
          ],
          "website": "datemyschool.com"
        },
        "locations": [],
        "end_date": "2012-08",
        "start_date": "2012-01",
        "title": {
          "levels": [
            "mgr"
          ],
          "name": "marketing manager",
          "functions": [
            "marketing manager"
          ]
        },
        "type": null,
        "is_primary": false,
        "most_recent": false,
        "last_updated": null
      }
    ],
    "education": [
      {
        "school": {
          "name": "university of oregon",
          "type": "post-secondary institution",
          "location": "eugene, oregon, united states",
          "profiles": [
            "linkedin.com/edu/university-of-oregon-19207",
            "facebook.com/universityoforegon",
            "twitter.com/uoregon"
          ],
          "website": "uoregon.edu"
        },
        "end_date": "2014",
        "start_date": "2010",
        "gpa": null,
        "degrees": [],
        "majors": [
          "entrepreneurship"
        ],
        "minors": [],
        "locations": []
      }
    ],
    "skills": [
      {
        "name": "social media"
      },
      {
        "name": "strategic partnerships"
      },
      {
        "name": "public speaking"
      },
      {
        "name": "sales"
      },
      {
        "name": "photoshop"
      },
      {
        "name": "networking"
      },
      {
        "name": "mobile marketing"
      },
      {
        "name": "start ups"
      },
      {
        "name": "business development"
      },
      {
        "name": "fundraising"
      },
      {
        "name": "seo"
      },
      {
        "name": "strategy"
      },
      {
        "name": "idea generation"
      },
      {
        "name": "enterprise technology sales"
      },
      {
        "name": "entrepreneurship"
      },
      {
        "name": "program management"
      },
      {
        "name": "marketing"
      },
      {
        "name": "social networking"
      },
      {
        "name": "creative strategy"
      },
      {
        "name": "time management"
      },
      {
        "name": "product management"
      },
      {
        "name": "social media marketing"
      },
      {
        "name": "css"
      },
      {
        "name": "https"
      },
      {
        "name": "saas"
      },
      {
        "name": "management"
      },
      {
        "name": "business strategy"
      },
      {
        "name": "project management"
      },
      {
        "name": "public relations"
      },
      {
        "name": "marketing communications"
      },
      {
        "name": "sales/marketing and strategic partnerships"
      },
      {
        "name": "marketing strategy"
      },
      {
        "name": "mobile devices"
      },
      {
        "name": "installation"
      },
      {
        "name": "company culture"
      },
      {
        "name": "strategic vision"
      },
      {
        "name": "html5"
      },
      {
        "name": "hiring"
      }
    ],
    "industries": [
      {
        "name": "computer software",
        "is_primary": true
      }
    ],
    "interests": [
      {
        "name": "location based services"
      },
      {
        "name": "mobile"
      },
      {
        "name": "social media"
      },
      {
        "name": "colleges"
      },
      {
        "name": "university students"
      },
      {
        "name": "consumer internet"
      },
      {
        "name": "college campuses"
      }
    ]
  },
  "likelihood": 10
}
```
 



 
### Errors 

The API uses conventional HTTP response codes to indicate the success or failure of an API request. A `200` means there was a matching person returned, a `404` means that a matching person was not found or returned, any `4xx` besides `404` indicates an issue with the request, and `5xx` errors indicate an internal issue with the API. 


**4xx Response Fields**  

| Field Name | Type | Description |
|-----------|---------|---------| 
| `error` | object | Object containing the error type message |  
| `status` | integer | HTTP Status Code |  


**Example 404 Response** 

```json 
{
    "status": 404,
    "error": {
        "type": "not_found",
        "message": "No records were found matching your request"
    }
} 
```

**Example 429 Response** 

```json 
{
    "status": 429,
    "error": {
        "type": "rate_limit_error",
        "message": "An error occurred due to requests hitting the API too quick"
    }
} 
```

**Error codes** 

| Status | Error Name | Description |
|--------|-----------|---------|
| `400` | `invalid_request_error` | Request contained either missing or invalid parameters | 
| `401` | `authentication_error` | Request contained a missing or invalid key |  
| `403` | `quota_exceeded ` | Your account has reached its quota for successful API calls |  
| `404` | `not_found ` | No records were found matching your request |  
| `405` | `invalid_request_error ` | Request method is not allowed on the requested resource |  
| `429` | `rate_limit_error ` | An error occurred due to requests hitting the API too quick |  
| `5xx` | `api_error ` | The server encountered an unexpected condition which prevented it from fulfilling the request |  



### Bulk Endpoint 

Up to 100 persons can be enriched in a single HTTP request using the `/v4/person/bulk` endpoint. Enrichments executed against the bulk endpoint must be a `POST`. The request body of a bulk enrichment request must contain an array, `requests`, with 1-100 individual request objects, each containing an object `params` of request parameters. A JSON schema describing the structure of a `/v4/person/bulk` enrichment request can be found at [https://github.com/peopledatalabs/docs/blob/master/schemas/bulk_request.json](https://github.com/peopledatalabs/docs/blob/master/schemas/bulk_request.json)

```curl
curl -X POST "https://api.peopledatalabs.com/v4/person/bulk" -H 'Content-Type: application/json' -d'
{
    "requests": [
    	{
    		"params": {
    			"profile": ["linkedin.com/in/seanthorne"],
    			"location": ["SF Bay Area"],
    			"name": ["Sean F. Thorne"]
    		}
    	},
    	{
    		"params": {
    			"profile": ["https://www.linkedin.com/in/haydenconrad/"],
    			"first_name": "Hayden",
    			"last_name": "Conrad"
    		}
    	}
    ]
}'
``` 

Responses are returned as an array of [response](#response) objects.  

```json
[
	{"status": 200, "likelihood": 10, "data": ...},
	{"status": 200, "likelihood": 10, "data": ...}
]
```

Response objects are not always returned in the same order as they were defined in the `requests` array. For this reason, we strongly recommend adding an object `metadata` to each request, containing any information specific to that request. If `metadata` is defined in a request object, it will be returned, unchanged in that request's corresponding response object: 

```curl
curl -X POST "https://api.peopledatalabs.com/v4/person/bulk" -H 'Content-Type: application/json' -d'
{
    "requests": [
    	{
    		"metadata": {
    			"user_id": "123"
    		},
    		"params": {
    			"profile": ["linkedin.com/in/seanthorne"],
    			"location": ["SF Bay Area"],
    			"name": ["Sean F. Thorne"]
    		}
    	},
    	{
    		"metadata": {
    			"user_id": "345"
    		},
    		"params": {
    			"profile": ["https://www.linkedin.com/in/haydenconrad/"],
    			"first_name": "Hayden",
    			"last_name": "Conrad"
    		}
    	}
    ]
}'
``` 

```json
[
	{"metadata": {"user_id": "123"}, "status": 200, "likelihood": 10, "data": ...},
	{"metadata": {"user_id": "345"}, "status": 200, "likelihood": 10, "data": ...}
]
```

Any of the response filtering or formatting params documented in the [Parameters](#parameters) section can be defined globally for all request objects: 

```curl
curl -X POST "https://api.peopledatalabs.com/v4/person/bulk" -H 'Content-Type: application/json' -d'
{
	"required": "emails AND profiles",
	"requests": [
    	{
    		"params": {
    			"profile": ["linkedin.com/in/seanthorne"],
    			"location": ["SF Bay Area"],
    			"name": ["Sean F. Thorne"]
    		}
    	},
    	{
    		"params": {
    			"profile": ["https://www.linkedin.com/in/haydenconrad/"],
    			"first_name": "Hayden",
    			"last_name": "Conrad"
    		}
    	}
    ]
}'
``` 
Or locally in a single request object: 

```curl
curl -X POST "https://api.peopledatalabs.com/v4/person/bulk" -H 'Content-Type: application/json' -d'
{
	"required": "emails AND profiles",
	"requests": [
    	{
    		"required": "experience",
    		"params": {
    			"profile": ["linkedin.com/in/seanthorne"],
    			"location": ["SF Bay Area"],
    			"name": ["Sean F. Thorne"]
    		}
    	},
    	{
    		"min_likelihood": 8,
    		"params": {
    			"profile": ["https://www.linkedin.com/in/haydenconrad/"],
    			"first_name": "Hayden",
    			"last_name": "Conrad"
    		}
    	}
    ]
}'
```  

Response filtering/formatting params defined locally in an individual request object will override those defined in the request body root.  

Any response object in a `/v4/person/bulk` response will either have a status code of `200`, `404`, or `400`. Any valid `/v4/bulk/person` will return with a status code of `200`.  

The number of remaining enrichment matches your account is alloted will be deducted by the number of `200` responses in a bulk enrichment request.  

Any malformed, unauthenticated, or throttled request will return errors in the same format they're documented in the [errors](#errors) section.  




```curl
curl -X POST "https://api.peopledatalabs.com/v4/person/bulk" -H 'Content-Type: application/json' -d'
{
	"required": "names"
}'
```  

```json 
{
    "status": 400,
    "error": {
        "type": "invalid_request_error",
        "message": "Request object must contain `requests` field"
    }
} 
```








