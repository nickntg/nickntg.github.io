---
layout: post
title: "YouTube integration"
categories: gaming
author:
- Nick Bitounis
---

This is a continuing post about how I got to create [Replay Video Creator](https://www.replayvideocreator.com/){:target="-blank"}. In my last post I described how I [captured video](https://nickntg.github.io/gaming/2022/03/18/capturing-video.html) from replay files of [World of Warships](https://worldofwarships.eu/){:target="_blank"}. I
In this post I'll go over the integration of the Replay Video Creator web application with YouTube.

## YouTube
After managing to capture video out of game replays, I wanted to allow my users to upload their captured video to their YouTube channel. In theory, this seemed to be straightforward to achieve based on the YouTube's [Data API sample](https://developers.google.com/youtube/v3/code_samples/dotnet#upload_a_video){:target="-blank"}. To start, I created an account in [GCP](https://cloud.google.com/gcp){:target="-blank"}. Then I created a new application and went through the short process of generating OAuth2 client IDs. 

When a user asks for a video to be uploaded to YouTube, the web application goes through the Google OAuth authorization process and asks for youtube.upload scope. Google categorizes scopes to non-sensitive, sensitive and restricted. Depending on the scopes your application requires, you need to undergo a verification process [that can be quite different](https://support.google.com/cloud/answer/9110914){:target="-blank"}. The youtube.upload scope I needed is a sensitive scope, meaning that my web application would not need to undergo an independant third-party security assessment, which was good.

After I got my client IDs, the web application moved to the testing phase and I was able to implement the integration within a couple of days. While in testing phase, the integration works exactly as it would in production with one major difference: I had to explicitly state the Google email accounts that can access my application and they cannot be more than 100. This allowed me to test without other limitations until I was happy with the implementation. The web application basically acquired the required YouTube scopes through the Google authentication process and passed these to a background YouTube uploaded worker that did the job. Once upload of the video to YouTube was complete, the worker erased the authentication token received - if the user wanted to upload another video, the web application asked for permission again.

Although this is an oversimplification and there is some code involved in implementing this process, it's not overly complex. Google APIs are thorough and there is enough documentation to guide you through this. What surprised me was the amount of time and work I had to put in, in order to take the YouTube integration live once I was happy with the testing. Google asked me to describe the web application in detail and they were particularly insistent in understanding how I used the scopes obtained through authorization. They were also very particular in making sure that the Terms of Service and Privacy Statement in the site reflected exactly what the application did the scopes, how long they could be used and how a user can revoke the provided authorization at any time.

There were a series of email exchanges with Google that took almost three weeks. Looking back, it is perfectly understandable that Google took me through these steps. They need clarity into what I do with their APIs and most importantly with the user data.

Here are the steps I took to get the web application integrated with YouTube after implementation was complete:
* First I had to create an OAuth verification request to take the integration live and out of development.
* I then had to verify to Google that I was the owner of replayvideocreator.com.
* After that, I had to create a video to show what I was doing with the web application and YouTube.
* I had to comply with the YouTube branding guidelines more closely and use the appropriate YouTube icon at the appropriate places. I made some corrections and recreated a video showing the web application, which was resubmitted to Google.
* At that point the web application got verified to use the youtube.upload sensitive scope. However, the quotas I received meant that my users could upload a maximum of 5 videos per day, a limit that I hit pretty quickly. I then submitted a request for quota increase.
* YouTube then asked me to verify that I was not using multiple project IDs for my API client, which I did.
* I was then asked to embed a link to YouTube's Terms of Service into my own Terms of Service page.
* After that, I was asked to explicitly state in my own privacy policy that my web application used the YouTube API services.
* Then I was asked to explicitly refer to Google's privacy policy in my own privacy policy page.
* I was also asked to refer to the security settings page of Google in my own privacy policy.

After all that, and a short wait of a few days, I was granted the quota extension.

## Next
In the future, I will write a little bit about what ended up taking way more effort that I expected: adding "little" features.