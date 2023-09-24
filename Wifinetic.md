

HackTheBox write up: “Wifinetic” Machine
========================================

[
![zerosandones](https://miro.medium.com/v2/resize:fill:88:88/1*8PgWCWtteykN7zPXDqImEQ.jpeg)

[Follow](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fa3f1b2b68ccf&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40ahmedgomaa_45441%2Fhackthebox-write-up-wifinetic-machine-3a2dfe81e08f&user=zerosandones&userId=a3f1b2b68ccf&source=post_page-a3f1b2b68ccf----3a2dfe81e08f---------------------post_header-----------)

5 min read·Just now


![](https://miro.medium.com/v2/resize:fit:875/0*GymvSckkiXy9SSLu.png)

Wifinetic as exciting machine to solve, as it is about wifi and wireless connections, which are areas of intrest for me personally, lets start with the way i approached solving this machine

First, we start with a basic nmap scan against the machine;

```
nmap -sC -sV 10.10.11.247
```

We are using -sV and -sC here for probing the open ports and trying to get info about the version running by running it against default nmap scripts.
![](https://miro.medium.com/v2/resize:fit:875/1*x0p6WQpJQ43mTVlMKVUwHw.png)

as we can clearly see here, we have an ftp service running, with anonymous login allowed, we proceed by logging in as anonymous on the ftp service via the command

```
ftp 10.10.11.247
```

and then followed by : “anonymous”. We see that we logged in successfully, now by listing the files we see 5 files available, we download them to our machine for further inspection, using command mget \*
![](https://miro.medium.com/v2/resize:fit:875/1*Wzkuwx4LjZKlT7Logj2IYw.png)

after inspecting the pdf files, and uncompressing the compressed file using the command

```
tar -xvf backup-OpenWrt-2023-07-26.tar
```

upon searching for information, we observe the etc/passwd file, which contain the usernames for the users in the system, a specific user stands out, “netadmin”, which seems to be custom made with some administrative privileges. upon further searching, we come across config/wireless file, that contains what seems to be wireless networks’ information, alongside with the key

![](https://miro.medium.com/v2/resize:fit:875/1*GbK2NVQ-lM5XPJRKYtk3Dw.png)

maybe if this key is reused, we can try to login into the machine through ssh via this password.

we try to login as admin@10.10.11.247 and password “VeRyUniUqWiFIPasswrd1!” but fais, we try again to login as netadmin@10.10.11.247 and password “VeRyUniUqWiFIPasswrd1!”, and this time we get a succeful login.

and we can look for the flag at user.txt
![](https://miro.medium.com/v2/resize:fit:875/1*6kBHO5F4qbECHJ_7k2oBJQ.png)

Privilege Escalation
====================

As for the root flag, we need to do Privilege Escalation in order to access the root directory.

I first started by checking the allowed commands for our user to run as root without the password, by running sudo -l, but i did not find and files the user is able to run with sudo without the password.

Second thing comes to mind, is to run (Linpeas) and enumerate for any interesting ways to get our privileges escalated

![](https://miro.medium.com/v2/resize:fit:875/1*hbMD91vlRERIVdPHeDLhow.png)


But it does not give us any clear cut exploitable CVE, so we will try to find another way.

we can start viewing network and wireless information via the command:

```
iwconfig
```
![](https://miro.medium.com/v2/resize:fit:875/1*81Jgpo1T49DRhWxT7DA9Pw.png)

we find that

1.  wlan0: This interface is in master mode, indicating that it is configured as the Access Point (AP)
2.  wlan1: This interface is in client mode, confirming that it functions as a Wi-Fi client, connecting to another wireless network. We observed the AP BSSID associated with this client interface earlier.
3.  mon0: This interface is in monitor mode, which is commonly used for wireless network monitoring and testing purposes. It does not participate in regular client or AP activities.
4.  wlan2: This interface is in managed mode but is not associated with any network. It appears to be inactive at the moment.

now we can try to brute force attack the network on wlan2 in order to obtain the password, there are many tools used for this purpose, lets see which of them are available and ready to be used on the system. we search using the command :

```
 dpkg -l | grep -E 'reaver|pixiewps|aircrack|wifite'
```
![](https://miro.medium.com/v2/resize:fit:875/1*uJVR6sPPNLaULylh049rEQ.png)

as we can see we have both reaver and pixiewps available, let us go with reaver to bruteforce the password from the wpspin

we run it using this command:

```
reaver -i mon0 -b 02:00:00:00:00:00 -vv -c 1
```

1.  `reaver`: This is the name of the command-line utility, "reaver," that you're executing.
2.  `-i mon0`: This option specifies the wireless interface to use for the attack. In this case, it's set to "mon0," which is monitor mode interface. Monitor mode allows the wireless card to capture and analyze network traffic, which is often used in Wi-Fi penetration testing.
3.  `-b 02:00:00:00:00:00`: This option specifies the target BSSID (Basic Service Set Identifier) or MAC address of the Wi-Fi network you want to test. Replace "02:00:00:00:00:00" with the MAC address of the target network we are testing.
4.  `-vv`: These are verbosity options. They increase the level of detail in the output for debugging and monitoring purposes. `-vv` indicates a high level of verbosity, meaning you'll see more detailed information about the attack as it progresses.
5.  `-c 1`: This option specifies the channel on which the target Wi-Fi network operates. In this case, it's set to channel 1.

![](https://miro.medium.com/v2/resize:fit:875/1*eAG3qfceJipcPUSxvW5Ybg.png)

we can see here that we obtained the WPA key, which is the password of the wifi network.

now based on earlier enumeration, it has been observed that the system’s user is highly likely to be reusing passwords across different accounts. We can try to reuse this password on SSH as root user and see if it works.

![](https://miro.medium.com/v2/resize:fit:735/1*5PgwMvdaqUE_Vb3CL87CRg.png)

and as we can see, we login successfully as root, and are able to obtain the root flag.
