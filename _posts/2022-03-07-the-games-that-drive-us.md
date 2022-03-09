---
layout: post
title: "The games that drive us"
categories: gaming
author:
- Nick Bitounis
---

I might not have gotten into software development if it wasn't for [Elite](https://en.wikipedia.org/wiki/Elite_(video_game)){:target="_blank"}.

{:refdef: style="text-align: center;"}
![](/assets/imgs/posts/2022-03-07-the-games-that-drive-us/220px-BBC_Micro_Elite_screenshot.png)
<br/>
_Elite screenshot - this image is about 1/3rd of the size of the Elite game itself_
{:refdef }

I got my first PC when I started styduing for a math degree. The first program I got for the PC was a floppy disk with Elite, which I first played back in 1985 on a friend's [BBC micro](https://en.wikipedia.org/wiki/BBC_Micro){:target="_blank"}. As it so happened, I was also taking a [Pascal](https://en.wikipedia.org/wiki/Pascal_(programming_language)){:target="_blank"} introductory course during the first semester of my studies. When the teacher introduced file I/O, I cross-referenced that information with the knowledge that Elite must've been storing game save files _somewhere_. Before long, a credits loader for Elite was my first complete, usable and purposeful program. Thus I discovered computer programming and I got hooked instantly. At the end of my first semester I switched from a math to a computer science major.

In recent years I got addicted to [World of Warships](https://worldofwarships.eu/){:target="_blank"}, a naval battles game. Although WOW can be fast-paced, you don't need the reflexes required by a first person shooter where you may sprain your fingers while working the WASD keys. And you can also play only for a short while as each battle lasts up to 20 minutes max. After playing the game for some time, I found myself to be part of a clan of people with the same addiction. Part of the clan was a YouTube channel that was used to upload some of our best battles. WOW records battles in files and as it turned out these replay files can be exchanged between players. When you open a WOW game replay, the game unfolds for you from the perspective of the player that captured the replay. Since I apparently had one of the best rigs in the clan, I took it upon myself to receive replay files from the clan, turn them into videos and upload them to YouTube.

This is a tedious process. After experimenting, I ended up using [OBS Studio](https://obsproject.com/){:target="_blank"} to capture the video of replays in 2K. I also used [Shotcut](https://shotcut.org/){:target="_blank"} to blend clan promo videos and game images in the final video, which was then uploaded to YouTube. Depending on the length of each battle, the final videos can be anything from 6 to 20GBs. The whole process requires some manual steps but, worst of all, it slaves my PC for quite some time. When capturing the replay video, I can not do anything with my PC for the duration of the battle. After the battle is recorded, video blending takes most of my CPU and GPU for anything ranging from 10 to 30 minutes. And uploading the final videos with my 10Mbit upload speed can take 1.5 to 4.5 **hours**.

I created about 100 videos with that manual process, at which point I started dreading the next replay-to-video request from my clan buddies. It was at this point where I started thinking about automating the process. But, even with a fully automated process, the major drawbacks of having my PC being the game capture/upload slave would remain.

So why not construct the automation but have it run in the cloud?

This is how I ended up creating [Replay Video Creator](https://www.replayvideocreator.com/){:target="_blank"}. After a lot of incremental releases, I ended up with a site where I can upload game replays and the site pretty much does the heavy lifting: captures a video out of the game replay, blends additional media into the game video and uploads the end result to YouTube.

Sounds simple right? And it is. But as it is the case with many simple things, doing it right is not easy or straightforward. Here's how the whole system looks like.
{:refdef: style="text-align: center;"}
![](/assets/imgs/posts/2022-03-07-the-games-that-drive-us/rvc_aws.PNG)
<br/>
_High-level AWS setup of Replay Video Creator. Click [here](/assets/imgs/posts/2022-03-07-the-games-that-drive-us/rvc_aws.PNG){:target="_blank"} to enlarge._
{:refdef }

There's a lot of stuff packed in there. The primary function of the site is to accept WOW replay files and make a video out of them. The site itself is written in .Net 6.0 and uses the [Identity Framework](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-6.0&tabs=visual-studio){:target="_blank"} for user management and security. Logged in users can upload WOW replay files for video capture and when they do, a capture job is pushed to an [SQS](https://aws.amazon.com/sqs/){:target="_blank"} queue. Video capture can take a relatively long time and is compute-intensive, so backend asynchronous processes running on servers dedicated to this function are taking care of these jobs. Once a video is captured, it is uploaded to [S3](https://aws.amazon.com/s3/){:target="_blank"} and is available for download.

This scheme, queuing jobs to SQS and having them being processed asynchronously in the background, is used for other major features implemented. These are editing, indexing and YouTube uploading. Editing is the process where a user selects one of the videos created from a game replay and blends in other videos and images into it. Indexing is an internal function that is used to provide the site search engine with searchable content based on the game replay data. Finally, uploading to YouTube is the final goal of what I set out to do and this turned out to require a separate server on its own. At several points, the site provides the user with email notifications when these jobs finish. This is accomplished by a small [Lambda](https://aws.amazon.com/lambda/){:target="_blank"} function that uses [SES](https://aws.amazon.com/ses/){:target="_blank"} to send outgoing email.

I will probably write up several posts about each of the major features and how they were implemented. But I will finish this post here by sharing a couple of points about my experience so far. Needless to say that people that are developers at heart can get an enormous amount of satisfaction by creating applications that turn out to be useful even when these are not understood by the majority of the sane people that share the world with them. This was the case for me as well and I must say that over the course of the weeks that where required to implement Replay Video Creator, I put in most of my limited free time. I didn't mind in the least because I am pleased with the final result. It scratched my personal itch, I learned stuff along the way and the end result seems to be useful to a small number of people as well, so hurray. I am a hardcode backend developer, so a lot of the cogs and gears that were created and work in the background, unseen by the end users, were pure pleasure to code. But boy, oh boy, did I have a hard time with the frontend. I suck at the frontend. I have no sense of style or design and every single line of HTML, CSS and Javascript that I had to assemble left a permanent mark on my psyche. During this process, I have rediscovered and renewed my respect for the frontend people and what they have to go through in order to create usable stuff. Sometimes backend developers have a nagging feeling that the gruesome, important and serious work is done in the backend of the stack, while the frontend is all fun and games. Nothing could be further from the truth.