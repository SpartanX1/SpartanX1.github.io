---
tags: PowerShell
---
Debugging and Profiling memory leaks in NodeJS using Chrome
===========================================================

![Photo by Joan Gamell on Unsplash](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*wylSMCuEGGZXLnv82MtUbQ.png)

Recently while monitoring micro-services in Production, we noticed that some of our services written in NodeJS were restarting every week. On investigating the logs, all of them had the same of kind of error.

> FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed — JavaScript heap out of memory

Definitely not something you want to see on a weekend ! Phew, we were saved because the services were on autopilot and restarted whenever this problem occurred.

Nonetheless, we took the red pill and decided to go down the rabbit hole.

The immediate lines trailing the error do not often have sufficient info on what is causing it. Could be a global variable, an infinite function or a faulty npm package. Without debugging the code, a conclusion cannot be derived by just looking at the logs.

**Debugging mode:**
-------------------

Firstly we need to start our application in debug mode. I recommend using the visual studio code debugger. If you are new to it, follow the instructions [here](https://code.visualstudio.com/docs/nodejs/nodejs-debugging), although for Node it is quite straight forward whatever the framework is (Express/NestJS..)

![captionless image](https://miro.medium.com/v2/resize:fit:790/format:webp/1*zbtDO5zb9-sPQyEY45sXeA.png)

Or you can run the cmd,

```
node --inspect-brk <your path to main js file>
```

Your app should now start in debug mode.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*n8VJB3XcFbNXjilpUMDrjw.png)

Chrome:
-------

Fire up the chrome browser. Press F12 to bring up the developer tools. You should now see a green nodejs icon on left. (takes a few seconds to show up)

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ZHKTMxkr04LytxesXN2PQA.png)

Open it and your process will be showing up on the **connections** tab on the same port your debugger is running.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*qJZt0WxDw-SjPH5wdyyD9Q.png)

To verify, your app will also show debugger attached

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TnhO-tmkrcKFU5FzZBCI2w.png)

Taking Memory snapshots:
------------------------

Select memory tab and profiling type. The heap snapshot is a good option to start with as well as it is the default option. Click on take snapshot on the bottom.

What’s it gonna do is take a memory dump of your node app and create a snapshot file complete with details about object allocation, memory used and other stuff. It is recommended to take multiple snapshots over a period of time say every 10 mins or 30 mins depending on how fast your app is encountering the heap out of memory error.

![captionless image](https://miro.medium.com/v2/resize:fit:756/format:webp/1*Bv-VMz4ClQiIaX_BIoAA2g.png)

Interpreting the snapshot result:
---------------------------------

At first glance, it might seem complex and having too much info on the screen. But don’t worry, we will make sense out of it in a minute.

**Structure** — The constructor column details all the objects/functions/context initialized, while the retained size column shows the amount of memory taken by those. Ordering them by retained size will allow you to see which objects take up most of your memory. To inspect further, you can expand the objects in constructor column and find out which packages or part of the code is doing that.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*lickRr8v_exrE-6PCAJO8w.png)

**Comparison** — Perhaps the most interesting option is the _“object allocated between snapshots”_ available on the first snapshot selected. This will compare two different snapshots and tell exactly which objects were created between them. Often, the code causing the issue will show up in these comparisons. An easy indicator is same type of object class/function.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*o-iVFbhKw5BRbsvKJG0YBA.png)

**Statistics —** Select statistics option from the upper column and it will show how much of memory is taken up by different parts of your code in a pie chart view. This is a quick way to visualize the overall allocation between objects.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Q82WrhUrCeLqk7HhzN9zuQ.png)

**Taking CPU profile:**

Another option to profile is the CPU usage. It’s an added indicator to verify if there is some long running function. Go to profiler tab and hit the record button. Once done, the profile looks something like this,

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*oMHf2MR84xuxJwsiTV7Nzw.png)

Things to note here are the durations and functions executed.

**Additional**: Use this [extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-js-profile-flame) on visual studio code to get a live view of cpu vs heap graph.

Well, by combining both memory and cpu profiling, you should have a good enough idea about what caused the heap out of memory. Hope this article helps.

Thanks for reading !
