---
layout: post
title: O365 – File Format Bug in ConfigMgr
subtitle: the story continues …
thumbnail-img: "/assets/img/20200430/bugfix.jpg"
tags: [M365, W10, Bug, SCCM, MEMCM, OSD]
comments: true
---

at one of my last customer engagement i saw some client machines which still couldn´t be updated with the last O365 1912 Monthly Channel Targeted Patch. The errors we have seen were actually looking exactly the same as the bug which got identified quite a few weeks back. Also in this case we have seen that ConfigMgr couldn´t download the update due to a wrong url format for the bits job!

![wrongformat](/assets/img/20200430/wrongformat-1024x80.png)

Microsoft provided a new release of those O365 updates as well as released a fix for clients which do have already a specific version of office installed!

![bugfix](/assets/img/20200430/bugfix.jpg)

  
  
[https://support.microsoft.com/en-us/help/4532996/office-365-version-1910-updates-in-configmgr-do-not-download-or-apply](https://support.microsoft.com/en-us/help/4532996/office-365-version-1910-updates-in-configmgr-do-not-download-or-apply)

As you can see, the bugfix actually checks if the version of office is between 16.0.12130.20272 and 16.0.12130.20390. My customer had version 16.0.12130.20210 installed. With this version, the bugfix terminates with the result that office can´t be updated as it still tries to use the wrong url format! You have now three ways of fixing the issue:

-   uninstall and Reinstall Office
-   update via CDN to the latest Office Build (requires internet connection)
-   simulate the bugfix tool for the version 12130.20210

I have decided to use the latest one as it does not interrupt the user. The challenge was to figure out what the tool does in detail to fix the issue. Below you will find the solution which works without an internet connection as well as does not interrupt the user.

-   On an affected machine, stop the  _ClickToRunSvc_  Service
-   Browse to the folder:  _C:\Program Files\Common Files\microsoft shared\_
-   Rename the  _ClickToRun_ folder to something else
-   Copy from a working machine with a later build the whole  _ClickToRun_  Folder
-   Start the  _ClickToRun_  service again
-   Restart needed
-   Now you can wait for the next update cycle or execute manually:
    -   WMIC /namespace:\\root\ccm path sms_client CALL TriggerSchedule “{00000000-0000-0000-0000-000000000108}” /NOINTERACTIVE
    -   WMIC /namespace:\\root\ccm path sms_client CALL TriggerSchedule “{00000000-0000-0000-0000-000000000114}” /NOINTERACTIVE