## About me 🧑🏻‍💻

I have been mostly hacking in **Python** and **Golang**, and appreciate both functional & object-oriented programming paradigms to solve real world engineering problems.

Right now I'm a three-year experienced software engineer @ **Goldman Sachs**, focusing on building the firm's distributed scheduling platform for batch workflows.

```text
What is a scheduling platform?

It is responsible for managing the dependency graphs among predefined jobs 
and their executions at the specified time and on the specified compute resources.
```

and then

```text
What is batch workflow?

It is the execution of a series of tasks in a predefined sequence without 
manual intervention, which usually processes a large volume of data
```

🤔 For example, a trading desk wants to start processing today's trading activities for some of their assets after the market close. They can set up:
* `Job_A` that triggers the post-trade application to start at 4:00pm local time to reconcile and generate transaction reports; or it can be individual job `Job_A1`, `Job_A2`, `Job_A3` for each asset class 
* `Job_B` that depends on all `Job_A1`, `Job_A2`, `Job_A3` to complete, before it starts aggregating all reports into central statistics
* `Job_C` that starts after `Job_B`'s completion to generate and submit regulatory reporting

```text
The dependency graph:

4:00pm EST ⏰
           | Job_A1 ————\
           | Job_A2 —————> Job_B ——> Job_C ——> done!
           | Job_A3 ————/
```

When realised that this workflow might be the same for tomorrow as well, you can set up recurrence for it on the platform using 
**cron** feature based on your custom schedule (e.g. trigger the above workflow every weekday, excluding bank holidays).

Of course, this is a simple example to illustrate the idea. There are more demanding scheduling requirements that the team is actively working on,
including but not limited to:

* what if the job generates huge amount of logs that put pressures on the allocated host;
* how to optimise for throughput if there are 100 thousand of such `Job_A1`...`Job_A100000`need to launch at the same time;
* noisy neighbour problem when jobs interfere with each other on the shared host resource

---

Earlier on, I used Golang & Gin HTTP framework quite extensively to build and optimise REST APIs for the Ad Network infrastructure @ **ByteDance**.

There is a lot of work involved to communicate and coordinate with different teams around the world to make Ads delivery effective:
* need **geolocation** targeting support to recommend right contents for the right region
* need to collect signals from the request context for **machine learning** model to produce more accurate personalisation
* need to infer user profile from the request context in order to make ads delivery compliant with local regulations

All happen within one HTTP round trip under 1 second organised behind the scene as a directed acyclic graph.

{{<details title="How does an OpenRTB request context (`POST payload`) look like:" >}}
```json
{
  "id": "YjqJLwAOHaAFwkGeqAj3Kg",
  "imp": [
    {
      "id": "1",
      "video": {
        "mimes": [
          "video/mp4"
        ],
        "maxduration": 600,
        "w": 360,
        "h": 640,
        "startdelay": 0,
        "playbackmethod": [
          3
        ],
        "pos": 1,
        "api": [
          3,
          6
        ],
        "protocols": [
          2,
          3
        ],
        "skip": 1,
        "placement": 5,
        "playbackend": 1
      },
      "displaymanager": "GOOGLE",
      "instl": 1,
      "bidfloor": 0.01,
      "bidfloorcur": "USD",
      "metric": [
        {
          "type": "click_through_rate",
          "value": 0.228,
          "vendor": "EXCHANGE"
        },
        {
          "type": "video_completion_rate",
          "value": 0.845,
          "vendor": "EXCHANGE"
        },
        {
          "type": "viewability",
          "value": 0.930,
          "vendor": "EXCHANGE"
        }
      ],
      "ext": {
        "billing_id": [
          "123456789"
        ],
        "open_bidding": {
          "is_open_bidding": 1,
          "adunit_mappings": [
            {
              "keyvals": [
                {
                  "key": "appid",
                  "value": "1000001"
                },
                {
                  "key": "placementid",
                  "value": "900000001"
                }
              ],
              "format": 5
            }
          ]
        }
      }
    }
  ],
  "app": {
    "name": "android app",
    "bundle": "com.android.app",
    "publisher": {
      "id": "pub-id",
      "ext": {
        "country": "JP"
      }
    },
    "content": {
      "url": "https://play.google.com/store/apps/details?id=com.android.app",
      "userrating": "3.5",
      "livestream": 0,
      "language": "en"
    },
    "storeurl": "https://play.google.com/store/apps/details?id=com.android.app",
    "ext": {
      "installed_sdk": [
        {
          "id": "com.google.ads.mediation.pangle.PangleMediationAdapter",
          "sdk_version": {
            "major": 4,
            "minor": 3,
            "micro": 4
          },
          "adapter_version": {
            "major": 4,
            "minor": 3,
            "micro": 400
          }
        }
      ]
    }
  },
  "device": {
    "ua": "Mozilla/5.0 (Linux; Android 9) Chrome/99.0.4844.58",
    "ip": "164.52.25.0",
    "geo": {
      "lat": 40,
      "lon": 116.31999999999999,
      "country": "CHN",
      "region": "CN-11",
      "city": "Beijing",
      "zip": "100190",
      "type": 2,
      "accuracy": 15009
    },
    "make": "samsung",
    "model": "a16",
    "os": "android",
    "osv": "9",
    "devicetype": 4,
    "lmt": 0,
    "w": 360,
    "h": 640,
    "pxratio": 3,
    "ext": {
      "user_agent_data": {
        "browser": {
          "brand": "Chrome",
          "version": [
            "99",
            "58"
          ]
        },
        "platform": {
          "brand": "Android",
          "version": [
            "9"
          ]
        },
        "mobile": 1,
        "model": "a16",
        "browsers": [
          {
            "brand": "Mozilla",
            "version": [
              "5"
            ]
          },
          {
            "brand": "Chrome",
            "version": [
              "99",
              "58"
            ]
          }
        ]
      }
    }
  },
  "user": {
    "id": "user-id",
    "data": [
      {
        "id": "publisher-id",
        "name": "Publisher Passed",
        "segment": [
          {
            "name": "appid",
            "value": "8000001"
          },
          {
            "name": "placementid",
            "value": "900000001"
          }
        ]
      }
    ]
  },
  "tmax": 1000,
  "cur": [
    "USD"
  ],
  "bcat": [
    "IAB7-39"
  ]
}
```
{{</details>}}
&nbsp;

---

Didn't really consider a master degree right now, because I felt I am more used to a top-down approach where I come across a real world problem
and try to break it down to fundamentals. 

Plus, I'm already surrounded by many interesting CS problems, peers and seniors in my day-to-day work that are worth spending time with.
