# Azure Honeypot
In this project, my goal is to create a cloud based honeypot that will log a specific kind of attack traffic (RDP failures) and run that through Microsoft Sentinel. I'll then be able to take that data and visualize it on a heat map to show generally where attacks come from and how many times they occurred. 

Special thanks to Josh Madakor on YouTube for creating a very clear guide to follow while I learned about this process. Though many of the things shown were out of date thanks to Microsoft changing things since the instructions were released, I was able to find my own solutions as they came up and still get the desired result. 
Link to his youtube channel: https://www.youtube.com/@JoshMadakor

> Please note if you are trying to follow along and use this writeup as a guide, Microsoft LOVES to change its UI on a regular basis so some of the things described below may be different or just plain not functional. I also don't claim that this is a difinitive method to set something like this up. This is just how I got it to work and achieve my goal. I'm confident there are better/simpler ways to go about this project.

# Things that were learned:
- Azure setup
- VM deployment and setup (Windows 11)
- Configure Log Analytics Workspace (LAW) to ingest logs from the virtual machine
- Set up Azure Sentinel (SIEM) which will take the logs from the LAW and present it in a specific way (heat map) using a custom workbook.  
- Using a custom PowerShell script to pass logs to a 3rd party API which will geolocate the IP addresses and store them locally on the VM for use by the LAW. 
- KQL scripting to parse the log data and create the map. 

# Setting up Azure
Setting up Azure is very straight forward and doesn't require a lot of skill. Simply going to https://portal.azure.com and going through the signup process is sufficient. It will ask for a payment card, but the service is good about warning users if they are about to incur costs. As of the time of writing, Microsoft gives you $200 worth of credit for the first month. **If you want to avoid being charged, try to finish what you are doing in this project then delete any resources being used within the first month to avoid charges.** 

>To delete the resources, just search `Resource Groups` and delete the resource group that contains everything we created for the project. 

# Creating the Virtual Machine
### Basic Setup
Either click or search for `Virtual Machines` then click on Create. 
Within the 'Basics' tab, create the resource group. Everything in this project will be under the same resource group. In this case, I've named mine `HoneypotLab`. 
Now name the virtual machine. I named mine `honeypot-vm` to keep things simple. Select the geographic region. It doesn't matter too much which one you pick so I chose the one closest to me personally. 
Select the Image you would like to use. I chose Windows 11 but any other Windows image should also work. 
Set up the username and password. **MAKE SURE IT'S SECURE** as this VM will be inherently insecure and open to the internet. The last thing we want is for someone to actually  successfully brute force the machine and get a login.  
Leave everything else default and confirm the licensing agreement at the bottom. 

### Network Setup
Click Next until you get to the Networking tab. Under `NIC network security group` select Advanced. Then under `Configure network security group` click create new. This is where you will be creating the firewall rules for the virtual machine. Delete any default rules and then click `+ Add an inbound rule`. 
Here are the parameters we are setting up:
```
Source: Any
Source port ranges: *
Destination: Any
Service: Custom
Destination port ranges: *
Protocol: Any
Action: Allow
Priority: 100
Name: DANGER_ANY_IN
```
You can name it whatever you want. I named it this to make it clear that this rule is a bad thing. Azure will also flag it with a yellow ⚠️ as well. 
Click add then next. You can now click Review and Create to begin creating the virtual machine. This process can take some time. 

The reason for this setup even though I'm only tracking RDP traffic is because I want this machine to be very discoverable even through other means such as ping sweeps or other scans. 
+
> Because the inbound rule has Any Any allow, anything we put inside this cloud network will be visible to the public. Don't store ANYTHING valuable on the network!

# Creating a Log Analytics Workspace
We will now need to create a Log Analytics Workspace, or LAW, which will ingest and create logs from our virtual machine. 
Click Create log analytics workspace. Click the drop down and select the resource group you created in the last step and then name the resource group. In this case I chose `law-honeypot`. Select the same region as your VM from earlier. Now click `Review + Create` and then `Create` to finalize everything. 

![Pasted image 20240326091345](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/ec01c62c-0d01-4500-a2ff-84915d64c219)

# Set up Security Settings
Next we will need to set up the Security Center. This used to be something included with Azure, but now is a separate thing that needs to be installed from the marketplace. Once it's all set up, select the workspace we just created and turn defender on. 

![Pasted image 20240326091841](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/d747ac22-847b-4f87-8591-c7113ac787ca)

Go into the Defender plan to ensure that it is set to protect servers. It should be the only thing with an instance: 

![Pasted image 20240326092403](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/5f760dff-d139-4b4d-b9b6-4f9669ee7692)

We'll also want to go into the settings for the LAW we created and make sure it is recording all events: 

![Pasted image 20240326092526](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/084adf8a-3626-4ead-beab-9628b4ae87e4)

Save any changes. 

# Back to the LAW
Go back into Log Analytics Workspace and select the honeypot workspace. In the blade menu there should be an option for Virtual Machines. Click on the one we created at the start and click `Connect` at the top. 

![Pasted image 20240326092755](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/6afb4e29-400c-4bf8-87c0-932a6160e924)

> Note that this feature was labeled as deprecated at the time of writing. While this feature is still functional when I did the project, it may no longer work at any point after. 

# Azure Sentinel Setup
This is the SIEM which will be handling all this data we are collecting. 
Search for `Azure Sentinel` and open the menu. Now click Create and select the LAW created for the honeypot. Select it then click add. 


# Unsecuring the VM
Our VM should be done generating at this point. Go into the Virtual Machine menu and open the honeypot vm. Copy the Public IP address and try to log into it using Virtual Desktop. Use the username and password created earlier. 

Now that we are logged in, we need to make the machine vulnerable! 
First, go onto your local computer and open a command prompt. Use the command `ping [IPADDRESS] -t` replacing the IPADDRESS portion with the public IP of the VM. It should continuously ping the host. Right now it should show request timed out. We're going to be changing that so keep it running... 
Go back into the VM and search `wf.msc` to open the windows firewall. 

![Pasted image 20240326094247](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/9e66b46a-184d-4042-a73f-0b7641bcfa11)

Then click `Windows Defender Firewall Properties` to open a new window. Go into each profile tab and turn the firewall state off for each one. 

![Pasted image 20240326094436](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/46ff7bc6-f147-4b2b-b210-181feb2bc88e)

Click OK to confirm everything. Now check your ping on the local machine. You should see it start getting a reply. 


# Setting Up the PowerShell Script
Now go back into the VM. We are going to be downloading a PowerShell script. Open the Edge browser and go to https://github.com/joshmadakor1/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1 and download the PowerShell script. For convenience, just save it to the desktop. 

> The easiest way to do this is to copy the full text, open PowerShell ISE, click new, paste the code, and save to desktop. We will be running the script through PowerShell ISE anyway so keep it open. 

Now we need an API key so the script can pass our log info to the API and create our custom log file. To do this, go to https://ipgeolocation.io/ and create an account. For what we need, a free account is fine. Once you have the API key, paste it into the API key section of the script. 

Now run the script and it should start collecting data. Any time there is a failed log in attempt, it will show up in the results pane. 

![Pasted image 20240326100241](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/7e4691f7-1ecf-4810-9c27-bbba8252b23a)

> Note that with the free version of the API key, you are limited to 1000 queries per 24 hours. That sounds like a lot, but it goes VERY fast. 

The log file being made is saved to C:\ProgramData by default unless specified in the script. This is normally a hidden folder so you will need to manually type it in. 


# Creating Custom Table

Now that we have the script running properly and creating data, we need to do something with the data to present it in a more readable format. The old way to do this, and the way the script expects you to do this, is to go into LAW and create a custom log, feed a sample of data into it (which the script automatically generates for you), and then train it to grab specific information. Unfortunately Microsoft removed this feature (very cool) in favor of using KQL for everything. I had never heard of KQL before this project so I went into this blind. Here is what I did to get it to work... 

Open LAW in Azure and then find the Tables blade under Settings. Click `+ Create` and select `New custom log (MMA-based)`. Give it a sample file. This can be copied right from the generated log file on the VM. Just copy the text into a txt file on the local machine. For collection path, Type should be Windows and path will be the path to the log file on the VM. Click next and give it a name and description. Click Next then Create to finalize everything. 

![Pasted image 20240326102806](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/f2a0dc27-f1df-4757-b5b8-5b59b40dc127)


# Sentinel Workbook 
Now the fun part. We are going to go into Sentinel and create a workbook. Open Sentinel and click the Workbook blade. Then click Create. 
Click Edit and remove all everything that was there by default. No click the `+ Add` button and select `Add Query`.

This is where we are going to run queries against the table we made. This used to be a thing we could train, but now needs to be done manually (at least as far as I can tell). 

From here I had to do a lot of work to find the right query to display the map I wanted. Set the Time Range to whatever you want and the Visualization to "Map". The query I ran was this:
```KQL
let logData = 
    FAILED_RDP_WITH_GEO_CL
    | extend RawData = tostring(RawData)
    | extend latitude = toreal(extract(@"latitude:([^,]+)", 1, RawData)),
             longitude = toreal(extract(@"longitude:([^,]+)", 1, RawData)),
             username = extract(@"username:([^,]+)", 1, RawData),
             sourcehost = extract(@"sourcehost:([^,]+)", 1, RawData),
             state = extract(@"state:([^,]+)", 1, RawData),
             country = extract(@"country:([^,]+)", 1, RawData)
    | summarize count() by latitude, longitude, username, sourcehost, state, country;

logData
| summarize count() by country, latitude, longitude
| project country, latitude, longitude, count_

```

I've been collecting data for several days. Running this query should display the following:

![Pasted image 20240326111350](https://github.com/HoldenDeHaan/AzureSentinelProject/assets/165294830/618b700e-338a-4f65-ac00-949637c040ea)

> As you can see, Thailand was very excited to find my machine. 

You can set the map to auto refresh in several intervals and as long as the API doesn't lock you out, it will still work. Though, without paying for a subscription, you'll reach the 1000 request threshold very quickly. 

