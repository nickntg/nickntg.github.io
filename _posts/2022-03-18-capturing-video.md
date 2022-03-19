---
layout: post
title: "Capturing video"
categories: gaming
author:
- Nick Bitounis
---

I [recently wrote](https://nickntg.github.io/gaming/2022/03/07/the-games-that-drive-us.html) about how I got to create [Replay Video Creator](https://www.replayvideocreator.com/){:target="-blank"}. In this post I'll go over the implementation of the basic feature of the site, which is capturing a video.

## What was required
The [World of Warships](https://worldofwarships.eu/){:target="_blank"} game has a feature: once you play a battle, the game is recorded on your local PC on a game replay file that has the extesion wowsreplay. If you open this file, the game replays the battle from the perspective of the user that played it. Fans of the game record their games and upload them to YouTube, which is what Replay Video Creator set out to facilitate.

To automate the process of capturing the video, I needed to do the following:
* Allow users to upload their replay files.
* Run the replays and record them.
* Upload the created video artifact to S3 so that it becomes available to download.

## Orchestrating the process
Accepting a replay file is easy enough. Registered users are allowed to upload them and their replay files are saved to S3. Similarly, once a video is created it is trivial to upload it to S3 and make it available for download. Each uploaded replay gets its own id in the form of a GUID and this is used to identify all artifacts of the game (replay file and generated video). The tricky part is capturing the video.

This obviously cannot be done synchronously. When WOWs replays a battle, it does so in realtime. This means that for a battle that lasted 20 minutes, the game will need 20 minutes to replay it plus the time required for the game to load. Therefore, once a replay is uploaded and saved to S3, the site queues a capture job to an SQS queue.

{:refdef: style="text-align: center;"}
![](/assets/imgs/posts/2022-03-07-the-games-that-drive-us/rvc_aws.PNG)
<br/>
_High-level AWS setup of Replay Video Creator. Click [here](/assets/imgs/posts/2022-03-07-the-games-that-drive-us/rvc_aws.PNG){:target="_blank"} to enlarge._
{:refdef }

The job is picked up from a capture server. The capture server setup is the following:
* Windows OS.
* WOW installed.
* [OBS Studio](https://obsproject.com/){:target="_blank"} installed.
* My capture orchestrator.

The capture orchestrator waits for capture jobs to arrive. Once it picks up a job, it downloads the replay file and opens it with WOW, which results in the game playing the battle. Once the battle starts, the capture orchestrator instructs OBS Studio to start recording the battle. When the battle ends, orchestrator instructs OBS Studio to stop recording and then uploads the created video to S3.

## Damn details
The devil is always in the details. In order to achieve what I said in the previous couple of paragraphs, I had to go through a few hoops.

### The Windows server
Setting up a Windows server as an [EC2](https://aws.amazon.com/ec2/){:target="_blank"} in AWS and installing WOW in it is easy enough. But the game needs to (a) run and (b) perform sufficiently enough so that videos can be recorded.

The first hurrdle was that WOW refused to run on a remote desktop connection. You have to run the game through a console connection. That is achieved in AWS by using [NICE](https://aws.amazon.com/hpc/dcv/){:target="_blank"}. Installing NICE is straightforward, and the only tricky part is that you need to assign a specific role to the Windows EC2 so that NICE can license itself.

If you try to run WOW on a [C5](https://aws.amazon.com/ec2/instance-types/c5/){:target="_blank"}-class server, you will quickly find that it performs miserably. My goal was to record videos at 60FPS - with C5 instances I couldn't even get the game to run decently, let alone record high-quality video. C5 instances are labeled by AWS as compute-optimized, which means they are good candidates for applications that require CPU. Since WOW is a game, I needed a server class with CPU <b>and</b> GPU, otherwise I would get nowhere.

The [G4dn](https://aws.amazon.com/ec2/instance-types/g4/){:target="_blank"} instance family seemed to do what I wanted so I tried that after C5. There are [some steps involved](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/install-nvidia-driver.html){:target="_blank"} to setup NVIDIA drives for Windows on G4dn in order to get use of the GPU. Once I got over those and tried WOW on the smallest G4DN instance (g4dn.xlarge has 1 GPU, 4vCPUs and 16GB of RAM) I was satisfied with the performance I got. The game played without hiccups with the highest video quality settings and I could capture videos with OBS Studio.

### The windows
To capture video with OBS Studio, you need to specify the capture video and audio sources. For the video source, you can specify an application window or a screen that OBS will capture. Unfortunately, to specify an application window you need to have the application running which I could not achieve automatically through the capture orchestrator. Therefore, I configured OBS to capture the screen where WOW would play the game to be recorded. That was easy enough. Unfortunately, when you specify a screen then OBS captures everything that is displayed on the screen. This includes the window task bar, OBS Studio and the window of the capture orchestrator application itself if it happens to be visible. OBS studio can be configured to go directly to the tray when started, which was good. My own application could also be instructed to run minimized.

So when the capture orchestrator started WOW, it had only to  hide the taskbar to ensure that it was not captured. In Windows you cannot completely hide the taskbar - the best you can do is instruct it to auto-hide but that still leaves a slice of it visible at the bottom of the screen.

I hadn't used Win API calls in a really long time but I had to use them here.
{%highlight c#%}
    public class WindowsApis
    {
        [DllImport("user32.dll")]
        private static extern int FindWindow(string className, string windowText);

        [DllImport("User32")]
        public static extern int ShowWindow(int hWnd, int nCmdShow);

        public const  int SwHide  = 0;
        public  const int SwShow  = 1;

        public static void HideTaskBar()
        {
            ToggleTaskBar(SwHide);
        }

        public static void ShowTaskBar()
        {
            ToggleTaskBar(SwShow);
        }

        private static void ToggleTaskBar(int flag)
        {
            var hWnd = FindWindow("Shell_TrayWnd", "");
            ShowWindow(hWnd, flag);
        }
    }
{% endhighlight %}

That binds my application to Windows of course. That was never a problem though, since I used Windows to run WOW.

### Controlling OBS Studio
At this point, I had to be able to tell OBS Studio when to start recording and when to stop recording. The first place I looked what the command line of OBS Studio, which allows some control over it but it does not let you completely instrument the application. Next, I tried binding hotkeys to OBS Studio for starting and stopping a capture and tried sending keystrokes to the application. That worked most of the time but it turned to be not completely reliable as sometimes it seemed that OBS Studio did not receive the keystrokes I was sending it. This was probably because of the way I did it, which was to find the OBS Studio window through the Windows API, then use SendKeys to send the keystrokes to it.

Thankfully, OBS Studio allows plugins to run. And there is the [obs-websocket](https://obsproject.com/forum/resources/obs-websocket-remote-control-obs-studio-from-websockets.466/){:target="_blank"} plugin which allows remote control of OBS through a websocket connection. By using the [websocket-client](https://github.com/Marfusios/websocket-client){:target="_blank"}, I found a way to reliably instruct OBS Studio to start and stop a recording.

### Knowing when to start recording
Starting WOW turned out to be simple. The simple part is that I had to use the Process class of .Net which starts an application and can monitor when it exits. The original executable spawns another process that actually replays the battle. I had to use WMI from the System.Management package in order to find the spawned process and wait for that to exit but that was also straightforward. The problem was knowing when to start recording.

When WOW starts, it initializes itself and then starts replaying the battle. Initialization turned out to be unpredictable and worst of all I had no way of knowing when the actual replay started playing. As expected, the first initialization after the server booted was longer than subsequent initializations. Unfortunately, I had to resort to configuring time intervals to wait after the game launched before started recording. I have two intervals configured: first wait in seconds and subsequent wait in seconds.

### Knowing when to stop recording
That's easy - when WOW exits. Tiny detail: WOW closes its window a little bit before exiting. At that point, OBS Studio records the desktop at the end of the video, icons and all. I had to put up a nice WOW-related desktop backgroud and tell Windows to not show icons on the desktop.

## Cutting costs
I'm running Replay Video Creator out of AWS Frankfurt. The current on-demand hourly rate for a single g4dn.xlarge instance running Windows is $0.842. This is about $606 per month, which is a tad pricey for a pet-project. I would much prefer to have the instance start when it had a video to create and stop when it had work to do rather than having it run all the time waiting for work. I therefore changed the capture orchestrator to shutdown the server it is running on if there are no jobs queued for one minute and I created a [Lambda function](https://aws.amazon.com/lambda/){:target="_blank"} that starts up a server if work is queued in SQS.

## Next
In the future, I will write another article about the minor adventure I went through to integrate Replay Video Creator with YouTube.