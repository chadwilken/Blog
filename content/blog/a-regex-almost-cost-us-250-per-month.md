---
path: a-regex-almost-cost-us-250-per-month
date: 2016-07-15T15:00:02.724Z
title: "A RegEx Almost Cost Us $250 per Month"
description: "TL;DR
Don’t use capturing groups in your RegExs unless you actually are going to use the matched content. CompanyCam is a photo app for contractors that receives about 9,000 images a day at the time…"
---

**TL;DR  
**Don’t use capturing groups in your RegExs unless you actually are going to use the matched content.

CompanyCam is a photo app for contractors that receives about 9,000 images a day at the time of writing. Believe it or not that doesn’t actually add that much load to the server in terms of processing and uploading to S3. Recently I changed the way we validated our incoming photos at a high level to make sure the base64 encoded image was in a format we accepted. I used a regular expression to perform a pretty simple check to make sure it matched the format `data:image/jpeg|png;base64,/`. Originally I was naive and used the format `/**\\A**(?:data**\\:**image**\\/**(?:jpeg|png)**\\;**base64**\\,**(?:.)+)**\\z**/`, the key part being the last `(?:.)+` acting as a catch-all after the initial **important** information. Base64 images in our system range anywhere from 10k — 200k characters long when encoded. All of these characters were being stored in a temporary piece of memory while validating the format. If you know anything about Ruby’s garbage collection it releases memory slowly. This was causing Monit to restart our Unicorn workers several times a day. Originally I thought we either had a memory leak or we were going to have to add another server into the cluster, which for us is around \$250/month. Luckily NewRelic notified me of an error and I was able to see that it was from being unable to allocate memory for the regex to run when called. Learn from me and realize that seemingly unimportant things can be your bottleneck in rare circumstances.
