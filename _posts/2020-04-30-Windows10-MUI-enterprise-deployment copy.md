---
layout: post
title: Windows 10 Multi-Language Enterprise Deployment
subtitle: make the deployment of windows 10 languages easy!
thumbnail-img: "/assets/img/blogw10lp/thumb.png"
tags: [Windows 10, OSD]
comments: true
---

Hello Everyone! My name is Christian Wunderlich, Premier Field Engineer (PFE) at Microsoft Germany for Microsoft Endpoint Manager. This blog post will provide you a better understanding on how to manage multilanguage windows 10 deployments. It will focus on how to deal with LXP files and language specific add-ons like Features on Demand! Once you have reached the end of this post, you can combine all those puzzle pieces to build your own multilanguage deployment!

Let´s get started with some basics….

## Features on Demand

### What are Features on Demand? 

Features on Demand (FODs) are windows feature packages that can be installed at any time. Common Features are .NetFx3 or additional language features for a language pack like Handwriting,  Text-To-Speech, speech recognition, …

Starting with Windows 10, version 1809 and Windows Server 2019, Windows has two different types of FODs available.  

#### FODs without satellite packages: 

Those FODs include all language resources packaged into just one file. Those FODs are actually distributed as a single cab file. 

![](/assets/img/blogw10lp/image.png)

#### FODs with satellite packages: 

FODs with satellite packages are language neutral features which have languages and/or architecture resources in separate packages, so called satellites. If you would point your installation to a source containing those language and/or architecture files, only the files which apply to the windows image are installed which reduces the disk footprint. 

![](/assets/img/blogw10lp/image-3.png)

FODs with satellites require a well-formed repository of cab files and metadata for an offline installation. You can´t just copy a few files from the source iso and expect that the installation is successful. Those satellite packages require metadata used during the installation.  

## Installation 

There are a couple of methods allowing you to install Features on Demand. FODs can be installed either online via windows update or offline using cab files. The later one requires a few preparation steps. 

## Preparation 

### Preparation of the offline repository 

There are a few options you have in order to prepare a repository of FOD files. Either you use the whole ISO files which can be up to 10GB or you extract only the files which are needed for a specific client and store them in a separate repository. If it comes to installations on just a few machines, the best way would be surely to just mount the iso and point the installation to this source. Another option would be to use a file share which contains the whole stack of FODs which you can also use in the source parameter. In case you want to deploy specific files during an Task Sequence with MEMCM, you might need to create a dedicated repository to use this as the package source. 

#### FODs without satellite packages 

All FODs without a satellite package just need the package.cab file in a folder. Once the file is in that folder, you can just point the installation to this source so it can be installed without further considerations. 

![](/assets/img/blogw10lp/image-1.png)

#### FODs with satellite packages 

FODs with satellite packages require more preparation in case you want use them offline. Beside from finding the necessary files and including them into a package, you also require the metadata folder as well as the FoDMetaData_Client.cab file! 

If you want to install a specific feature, you need to take the installed languages on the client into consideration. You have to provide the language neutral file(s) as well as all language related files for this feature. In addition, you also have to take care about the architecture amd64 and/or wow64. 

In this example, my client is a windows 10 1909 which has an en-us base installation with the german and french language pack on top. To install the DHCP RSAT Tool, you have to provide the following files. 

![](/assets/img/blogw10lp/image-2.png)

#### Preparation FODs online 

If you are planning to retrieve the content via the Internet (Windows Update) you just need to enable a GPO called “Specify settings for optional component installation and component repair” and select the option “Download repair content and optional features directly from Windows Update instead of Windows Server Update Services (WSUS). 

If you don´t to this, you might run into the Errorcode 0x800f0954 

### Installation methods 

You do have several methods and commands available to install FODs. Either via DISM or via add-windowscapability. In both cases, please use the parameter /add-capability instead add-package as the parameter /add-capability work with both satellites and non-satellite packages. 

There are other parameters available which can be found here.  

[https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-v2–capabilities#adding-or-removing-features-on-demand ](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-v2--capabilities#adding-or-removing-features-on-demand%E2%80%AF)

#### Online installation 

add-WindowsCapability -Online -Name Rsat.DHCP.Tools~~~~0.0.1.0 

#### Offline installation 

When installing FODs offline, you might have several commands to do so. As an example, I have listed a dism as well as a more recent powershell-cmdlet! 

-   DISM.exe /Online /add-capability /CapabilityName:”Rsat.DHCP.Tools~~~~0.0.1.0″ /limitaccess /source:”C:\temp\DHCP” 
-   Add-WindowsCapability -Online -Name “Rsat.DHCP.Tools~~~~0.0.1.0” -Source “C:\temp\DHCP” -LimitAccess 

References: 

-   [https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-v2–capabilities#overview](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-v2--capabilities#overview) 

-   [https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-non-language-fod](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-non-language-fod) 

## Windows Language Packs

Microsoft offers a variety of options to change the Windows Language to your demands. At the first glance this might seem to very confusing as there are cab , lxp, lip, appx, FODs and a lot more expressions used in those discussions about language packs. First, let´s bring in some clarity into this. 

### Language component explanation 

There are different language components used which need to be explained. Beside from the below mentioned main components, there are other components available which are not part of this guide. 

-   Language Packs (LP): Fully-localized Windows UI text for the dialog boxes, menu items, and help files that you see in Windows. Those files are delivered as .cab files 

-   Language Interface Packs (LIP): partially-localized languages. LIPs require a base language pack. LIPs only translate the most common dialog boxes, menu items and help files. UI language is not changed, therefore it requires the base language pack. In previous windows edition, those files have also been delivered as .cab files, since Windows 1803 those files are delivered as .appx packages, now called LXP files! 

-   Local Experience Packs (LXP): Since Windows 1803, Microsoft introduced Local Experience Packs which are modern language packs delivered through the Microsoft Store or Microsoft Store for Business. LXPs are the successor of the legacy LIPs which will be replaced!  

-   Features on Demand (FODS): Features on Demand include language specific additional features like spell checking, handwriting, text-to-speech and so on. Features on Demand include beside those language files a lot more features which can extend your windows environment like NetFx3. Those Features can be added at any time. 

#### Installation notes 

All kind of the above listed components can be installed during the provisioning process or afterwards as an additional language. According to Microsoft´s documentation, it is recommended to install language FODs (handwriting, text-to-speech, …) after you have applied a LP. To minimize the disk footprint, only install languages on the device what you think your users will require at the users location. 

### Installation 

As already mentioned above, you have the option to apply / install language packs while the OS runs or you have the option to inject them into the image itself.  

### Add languages to an running Operating System 

#### DISM 

LP or LIPs can be installed while the OS is being executed with the dism tool. You just need to copy the LP.cab file to the desired location and execute the following installation command. 

![](/assets/img/blogw10lp/l1.png)

-   dism /online /add-package /packagepath:”C:\temp\es-es\Microsoft-Windows-Client-Language-Pack_x64_es-es.cab” 

Beside installing a single lp, you can specify further files(LIP, FOD) into this location and execute the following: 

-   dism /online /add-package /packagepath:”C:\temp\es-es\” 

#### LXP 

LXPs are the successor of the traditional deprecated LIP files. Additional to the list of the previous LIPs, you now also have LXP files for all those languages for which you have had only a full language pack available. But be aware, this doesn´t mean you actually can deploy a LXP for a language like german, spain or french and expect that the whole OS got translated. LXP files, even for languages where a full language pack exist, will just translate the mostly used parts of the Operating System! 

Local experience packages can be applied with the following command: 

-   Add-appxpackage –Path C:\temp\de-de\LanguageExperiencePack.de-DE.Neutral.appx 

![](/assets/img/blogw10lp/l2.png)

By using add-appxpackage you are just registering the Application for the current logged-on user.  You can take a look for which user the app is actually registered by running this command. 

-   Get-AppxPackage -AllUsers | ? Name -Like *LanguageExperiencePack* | Format-List Name, PackageUserInformation 

Even after the language has been installed, it is not yet in the list of available display languages. How to add the language is covered in a later section.

There is another way to install a Local Experience Pack. This time, we are applying the LXP files with the powershell command add-appxprovisionedpackage. This cmdlet does not only install the LP for the current user, it does install it for all newly created users automatically.

-   Add-AppxProvisionedPackage -Online -PackagePath C:\temp\de-de\LanguageExperiencePack.de-DE.Neutral.appx -LicensePath C:\temp\de-de\License.xml 

After installing the LXP and the creation of a new user account, we can check the installation status by using the previously used command:

-   Get-AppxPackage -AllUsers | ? Name -Like *LanguageExperiencePack* | Format-List Name, PackageUserInformation

![](/assets/img/blogw10lp/l3.png)

So far the process went quite smooth, but unfortunately the newly installed LXP does not yet appear in the list of available windows display languages. It must be added afterwards, a more detailed guide is covered in a later section.

### Manually add the recent installed LXP to the list of available windows display language

Once you have installed the LP, LIP or LXP, you have to make it available to the user. Unfortunately, this is not yet automated, so even after installing the language, it is not yet automatically available in the list of windows display languages.

![](/assets/img/blogw10lp/l4.png)

Now either the user adds the installed language to his preferred list of languages OR you have to do this manually via powershell.

![](/assets/img/blogw10lp/l5.png)

> $OSLanguages = (Get-WmiObject -Class Win32_OperatingSystem -Namespace root\CIMV2).MUILanguages
> 
> $userlanguagelist = Get-WinUserLanguageList
> 
> foreach ($OSLanguage in $OSLanguages) {
> 
> $userlanguagelist.add($OSLanguage)
> 
> }
> 
> Set-WinUserLanguageList $userlanguagelist -force

References: 

[https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/localize-windows](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/localize-windows)

[https://techcommunity.microsoft.com/t5/windows-it-pro-blog/local-experience-packs-what-are-they-and-when-should-you-use/ba-p/286841](https://techcommunity.microsoft.com/t5/windows-it-pro-blog/local-experience-packs-what-are-they-and-when-should-you-use/ba-p/286841)

[https://docs.microsoft.com/en-us/powershell/module/dism/add-appxprovisionedpackage?view=win10-ps](https://docs.microsoft.com/en-us/powershell/module/dism/add-appxprovisionedpackage?view=win10-ps)

[https://docs.microsoft.com/en-ca/powershell/module/appx/Add-AppxPackage?view=winserver2012r2-ps](https://docs.microsoft.com/en-ca/powershell/module/appx/Add-AppxPackage?view=winserver2012r2-ps)

## How to automate this process with SCCM?

As Microsoft Endpoint Configuration Manager is widely used by a lot of enterprises, I wanted to take this into consideration and provide some guidance on how to actually deploy those LPs and LXPs in an automated way.

As so often, there is no general recipe which covers every use case, so there are a few things to consider before deciding which solution you want to implement!

First of all, there is the decision to make if you would like to install this LXP or LP as part of the Operating System Deployment or as an additional language you will offer via the Software Center. In both cases, there are a few things to take care of, which I will cover in the following sections!

### Deployment of additional Languages with MEMCM

The Deployment of additional LXP files with MEMCM might be a bit tricky as the language does not automatically register itself as an available display language you can select. Therefore we might need to actually tweak the installation here a bit 🙂

First of all, I just would like to highlight the different deployment options you have with MEMCM. The LXP installation can be done via

-   built-in application wizard (deployment of appx packages)
-   via the store as an (connected to Store 4 Business)

-   online application
-   offline application

In all deployment situations you will have to manually adjust the language settings afterwards as the installed language does not appear in the list of available languages in the dropdown menu. To bypass this problem, we have to execute a script, so windows automatically registers this LXP language as a selectable language.

I have build an application which executes a script as the user account, not system (to set default language and make it available in the language selection screen) which has a dependency of the application to install the LXP. So SCCM downloads both applications but first installs the LXP and executes secondly the script to make this language selectable in the list and sets it as default 😉 so easy!

Remember – Of course you can also install the LXP without the script, but it will just change the language for the current user. If another user will logon to windows, it will use the default language of your image.

The later section will cover what scripts and configuration is needed to build the application(s).

### Deployment of Languages as part of the OSD

I will not tackle the LP installation with MEMCM that much, as there are hundreds of other blogs describing this more detailed. Furthermore the approach even with the latest .cab files is still the same as in good old days.

This article is more about the deployment of the newer local experience packs with MEMCM. You might face the challenge, that you would like to use those LXP files, as this is the future, but still would like to apply them as you would have done with the traditional lp.cab files.

In order to do this, you have to use add-appxprovisionedpackage as this command installs but does not register the appx package during the task sequence. This is required as no user is logged-on yet! Furthermore, the System Account is not allowed to run the add-appxpackage command!  

![](/assets/img/blogw10lp/l6.png)

This command does pre-provision the appx package to the user, nevertheless this LXP still needs to get registered. To be able to do this, you have to add an additional step (run command line) in the task sequence to add a registry key.

-   cmd.exe /c reg add “HKLM\SOFTWARE\Policies\Microsoft\Windows\Appx” /t REG_DWORD /v AllowDeploymentInSpecialProfiles /d 1 /f

This regkey allows deployment operations (adding, **_registering_**, staging, updating or removing) of Windows Store apps when using a special profile. If this regkey does not exist or is disabled, the next step will be blocked!

The next step is to register the LXP for a user. To achieve this, we would need to add another step (run powershell script) and run this step as a user (f.e. service account).

![](/assets/img/blogw10lp/l7.png)

![](/assets/img/blogw10lp/l8.png)

Now, as the app is registered and not anymore in staging status, we can use again f.e. our custom xml file to make this LXP Language as the default language for the current and all new users!

![](/assets/img/blogw10lp/l9.png)

Another computer restart at the end will finish off the project to use LXP files and treat them like as regular lp files! You will now have the default language set to your desired one! If you consider to work with task sequence variables, you can do this for a lot more languages!

Just to provide you an overview about my task sequence setup:

![](/assets/img/blogw10lp/l10.png)

### Deployment Tools & Configurations

This section should outline the tools and configurations to automate the deployment of multi languages.

Custom XML file to set default language

You still can define custom “international settings” by using the intl.cpl tool in conjunction with a custom XML file. Those settings include default language, local, keyboard values and other language related configurations. Please be aware, that with Windows 10 the intl.cpl command line tool does not support newer settings available in the Region and Language section of the control panel. Furthermore you should use the new PowerShell cmdlet settings to automate customizing international settings. Nevertheless, I have used this particular solution to not only set the default language but also some other language settings for the current and for new users!

> <gs:GlobalizationServices xmlns:gs=”urn:longhornGlobalizationUnattend”>
> 
> <!–User List–>
> 
> <gs:UserList>
> 
> <gs:User UserID=”Current” CopySettingsToDefaultUserAcct=”true” CopySettingsToSystemAcct=”true”/>
> 
> </gs:UserList>
> 
> <!– user locale –>
> 
> <gs:UserLocale>
> 
> <gs:Locale Name=”de-DE” SetAsCurrent=”true”/>
> 
> </gs:UserLocale>
> 
> <!– system locale –>
> 
> <gs:SystemLocale Name=”de-DE”/>
> 
> <!– GeoID –>
> 
> <gs:LocationPreferences>
> 
> <gs:GeoID Value=”94″/>
> 
> </gs:LocationPreferences>
> 
> <gs:MUILanguagePreferences>
> 
> <gs:MUILanguage Value=”de-DE”/>
> 
> <gs:MUIFallback Value=”en-US”/>
> 
> </gs:MUILanguagePreferences>
> 
> <!– input preferences –>
> 
> <gs:InputPreferences>
> 
> <!–de-DE–>
> 
> <gs:InputLanguageID Action=”add” ID=”0407:00000407″ Default=”true”/>
> 
> <!–en-US–>
> 
> <gs:InputLanguageID Action=”remove” ID=”0409:00000409″/>
> 
> </gs:InputPreferences>
> 
> </gs:GlobalizationServices>

#### Powershell Skript to call this XML file (intl.cpl)

With this tiny script you can just call the intl.cpl command line tool and provide the custom XML file to change the language. I have build an sccm application (install as user) to just execute this script. As already outlined above, if you create another application for installing the LXP, you can use this as a dependency to install and set a new language automatically!

> start-sleep -seconds 20
> 
> $LogFileLocation = $env:TEMP+”\importlanguage.log”
> 
> Start-Transcript -path $LogFileLocation -Force
> 
> function prepare-scriptname
> 
> {
> 
> if ($hostinvocation -ne $null){$hostinvocation.MyCommand.Path}
> 
> else{$script:MyInvocation.MyCommand.Path}
> 
> }
> 
> [string]$ScriptName = prepare-scriptname
> 
> [string]$ScriptDirectory = Split-Path $ScriptName
> 
> Write-Output “Create Path”
> 
> $dedeXML = $ScriptDirectory + “\de-DE.xml”
> 
> Write-Output “import language settings from XML”
> 
> & $env:SystemRoot\System32\control.exe “intl.cpl,,/f:`”$dedeXML`””
> 
> Stop-Transcript

This is the content you would need to deploy this.

![](/assets/img/blogw10lp/l11.png)

References:

[https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-international-settings-in-windows](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-international-settings-in-windows)

Christian Wunderlich

PFE

**This post is provided “AS IS” with no warranties, and confers no rights. The solution is not officially supported. Any support provided by Microsoft regarding this solution may be limited. Microsoft does not guarantee the solution will work in all environments and/or scenarios.**