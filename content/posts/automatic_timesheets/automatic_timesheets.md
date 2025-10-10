---
title: Scraping the SmarTrip Website to Automate My Timesheets
date: 2025-10-08T20:03:12-05:00
---

For my job, I need to submit timesheets at the end of every week. Sometimes I forget to submit them and by Monday the people in charge of finance at the company get angry at me. Since I moved to DC, I have been trying my best to live a car-free lifestyle. This means taking the train and bus to work every day. I recently had the idea of trying to pull the trip data from my Metro card and use it to determine what times I arrive and depart work for that day programatically. That way I don't have to sit down every Sunday and manually calculate how long I worked based on my trips.

## No API, No Problem
I have used the Metro API in the past for getting train arrival times for my station. I figured there was a similar easy-to-use API for getting SmarTrip information, but that turned out to be an incorrect assumption. Instead I had to reverse engineer how their website was fetching the usage data. I used the Developer Tools in Firefox to see the network traffic generated for fetching the usage history for the past week for my card. The request looks like the following:
```http
POST /Card/UsageReport/Index/<CARDID> HTTP/2
Host: smartrip.wmata.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:143.0) 
	Gecko/20100101 Firefox/143.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
Content-Type: application/x-www-form-urlencoded
Content-Length: 309
Origin: https://smartrip.wmata.com
Connection: keep-alive
Referer: https://smartrip.wmata.com/Card/UsageReport/Index/<CARDID>
Cookie: ...

__RequestVerificationToken=...&IsByMonth=false&StartDate=09/29/2025
	&EndDate=10/03/2025&CardId=<CARDID>&BackUrl=~/Card/Summary/Index/<CARDID>
	&MinStartDate=9/1/2023 12:00:00 AM&SubmitButton=True
```

That `__RequestVerificationToken` was just some token embedded in the HTML form. I used regex (`[a-zA-Z0-9-_]{92}`) to rip it out of the response text after GET'ing the page.  Another quirk is that the site doesn't like the Python requests User Agent string, so I had to change it to use my browser's more normal looking User Agent string.

The response produces an HTML table that I wasn't really interested in parsing out. Luckily, they provide an "Export to Excel" button which provides a CSV of the data.
```http
GET /Card/UsageReport/GetExcel?StartDate=09%2F29%2F2025%2000%3A00%3A00
	&EndDate=10%2F03%2F2025%2023%3A59%3A59&Period=R&TransactionStatus=Successful
	&cardId=<CARDID> HTTP/2
Host: smartrip.wmata.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:143.0)
	Gecko/20100101 Firefox/143.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
Connection: keep-alive
Referer: https://smartrip.wmata.com/Card/UsageReport/Report
Cookie: ...
```

I converted the CSV to JSON to make it easier to extract the fields and found all bus trips by filtering for my bus route number in the "Entry Location / Bus Route" column. I grabbed the time the trip was initiated in the "Time" column.

Next, I had to add a time delta to the morning trip to account for the time it takes the bus to arrive at the stop near my work, as well as subtracting a different time delta from the evening trip to account for the amount of time it takes to leave work and get to the bus stop.

And with that, we had a working proof of concept:
```
09/29 -- 8:57:00 hrs
09/30 -- 8:35:00 hrs
10/01 -- 8:29:00 hrs
10/02 -- 8:36:00 hrs
10/03 -- 7:49:00 hrs
```

I polished up the script and have it running as a cronjob every Sunday, where it will email me the results so I can easily input them into my timesheet application. I made a quick [Python library](https://github.com/mmaekr/pysmartrip) available on Github if anyone wants to do some projects with their own SmarTrip data.

### Disclaimer

I know there are some people who may be wondering: "But what if you miss your bus on the way home? Won't that inflate your hours? What if the bus takes longer than usual to get to your work stop? That would also inflate your hours!" Well besides the bus being pretty consistent, if there are any major delays, I make sure to keep note of them and don't blindly input the results of this script into the timesheet application. I use it as more of a guide for giving me a rough estimate of how long I worked.
