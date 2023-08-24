### Mr. Robot Wirteup

Enumeration

First things first, after an ip route my host network was revealed as 10.38.1.0/24. Let's look around in the network. nmap -sP 10.38.1.0/24 shows the IP 10.38.1.111 in the network. Hence this should be the victim IP.

Another Nmap run, this time nmap -sV -sC 10.38.1.111 showed Port 22 is closed, so no SSH is possible. An Apache Server is running on Port 80 as its HTTPS pendant on Port 443.

Time to visit the website on the victim machine. which shows a static video. Nicely made but useless. More digging is needed. nikto --host 10.38.1.111 shows that there is robots.txt without any "disallow" entries (which is odd). On admin/index an admin login page was found. The jackpot was found on wp-login which means that we have a WordPress site. That could be our ticket to accessing the machine. While looking to robots.txt I decided to enumerate the site a bit more with gobuster. gobuster dir -u 10.38.1.111 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt might show additional useful directories. I canceled it after 10-15min, it wasn't giving me any more useful directories

Getting Access

By navigating to http://10.38.1.111/robots.txt I found and downloaded the fsocity.dict. Also, I got the first flag in key-1-of-3.txt. I Inspected the fsocity.dict. It contains usernames and passwords. The file is huge, 7.2 MB, too big to use in a CTF (If I want to finish this box anytime soon). I tried to find some duplicates, maybe it's possible to shrink the dictionary a bit. The command sort fsocity.dict | uniq > sorted.txt did exactly that and the resulting file is only 96,7kB small.

Note at this point I had to quit the machine. Two days later I returned to it this time not on with the vulnhub version but on TryHackMe using their attackbox. From this point on the IP addresses change from my virtual Network to the ones TryHackMe provided.

I visited the previous login page at wp-login. The page asks me for a username and password. I tried the usual admin, admin. Which, of course, did not work. But it gave me an oddly specific error message, which told me that the username was wrong. I started Burpsuite Community and watched the login sequence again through the interceptor. This line caught my eye wp-login.php:log=admin&pwd=admin. In combination with the specific error messages, I can try and craft something to enumerate usernames. I used Hydra and the provided wordlist for this operation. The command hydra -L sorted.txt -p test 10.10.96.54 http-post-form "/wp-login.php:log=^USER^&pwd=^PWD^:Invalid username" -t 30 gave me three usernames:

elliot

Elloit

ELLIOT

I could've guessed them since this is a Mr. Robot-themed box and the main guy is named Elliot. OSINT and maybe some Social Engineering are always good methods to obtain usernames or even guessing passwords. But this gave me another opportunity for me to play around with Burpsuite.

Now it's time to get elliots password. For that, I'm using wpscan. We could've used Hydra again to brute force the password again. But in this case, I found the wpscan command much quicker and easier. With wpscan -U 'elliot' -P 'sorted.txt' --url 10.10.96.54/wp-login I quickly had elliots password.

After logging in as elliot it seems like he's an admin for this site. It's always good training to search through every tab and press every button. While browsing through tabs and pressing buttons I noticed that, in the plugins tab, I can upload .php scripts. This can be used for an RCE. On https://pentestmonkey.net/ we can find a suitable PHP reversed shell. We need to change $IP and $port In this case, we can choose every Port we like, the IP is of course the IP of our attack box. Then with nc -lvnp [port] I started a netcat listener on my attack box.

I saved and applied changes and then uploaded the file. Which is not working due to some "signature" error. Luckily the edit tab allows me to change the code of the sites. So I edited the 404 page with the code provided by pentestmonkey, ran Netcat again, invoked a 404 page in my browser and got a reversed shell. I ran whoami , which showed I'm the user "deamon". An ls /home gave me only one additional user named robot. In robot's home, there is the 2. flag which can only be opened as the user robot. Luckily he left a password.hash.md5 which can be cracked with John.

john md5.hash --wordlist=sorted.txt --format=RAW-MD5 Maybe because I used the TryHackMe attack box, john wasn't working. So I used the first online Raw-md5 decoder Google showed to me. Which decoded elliots password.

Before login as robot the shell must be stabilized, otherwise, the su command won't work. Python would be my first choice to do so. So I checked if Pyhton is installed, which it is python3 -c 'import pty;pty.spawn("/bin/bash")' is stabilizing the shell and we can log in as robot. Now we can grab the 2. flag. Only one thing to do which is escalate privilages to root.

Become Root

This took me a while. I tried scripts and whatnot to gain root access but nothing seemed to work. While researching techniques for gaining root access I stumbled on this tutorial. https://macrosec.tech/index.php/2021/06/08/linux-privilege-escalation-techniques-using-suid/

I gave it a shot. This command find / -perm -u=s -type f 2>/dev/null is finding files owned by root. Now I'm looking for everything out of the ordinary in that case I was seeing /usr/local/bin/nmap. At https://gtfobins.github.io/ we can search for nmap and with nmap --interactive and using the !sh command we're gotten root access. In /root we've found the 3. and last flag thus the room is completed.