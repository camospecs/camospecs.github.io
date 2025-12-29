---
title: Hosting on AWS EC2 (Free Tier Guide)
date: 2025-12-23
tags:
  - aws
  - ec2
  - webserver
  - guide
---

Hello! Elastic Compute Cloud (EC2) is an IaaS platform provided by Amazon Web Services and is what I use to host this very website.

As a little freebie, Amazon give new users 750hrs of credit to use each month for 12 months. Not bad! However, as you’ll see throughout this guide, this is limited to their ‘free tier’ products so be careful when selecting your choices! The instance hosting this site uses a static IP which costs money. Here’s a snippet of the cost:

![[aws_costs.png]]

But don’t worry! You can still host a website without these costs. Just stick to the free tier options and don’t bother with a static IP.

So without further ado, I’m going to explain how I got this website up and running using AWS (and how you can too).

**Creating an EC2 Instance**

- Go to [https://aws.amazon.com/ec2/](https://aws.amazon.com/ec2/) and create an account.

- Input your details, and set up a bank card to ‘authenticate’ your account.

*Note. I’d highly recommend you use an account without an overdraft or funds. I had a situation with Google where I forgot about a VM for awhile and was billed £300. Don’t worry though, I contacted them and they were happy to refund it. I’m not sure if Amazon would be as forgiving…

- Once on the AWS homepage, navigate to the EC2 dashboard and select ‘Launch Instance’. Here you’ll be prompted with the instance creation page.
![[ec2_launch_instance.png]]
- Select an appropriate name for the instance. Something like ‘webserver’ or your website domain name.![[ec2_naming.png]]
- Select ‘Ubuntu’ as the OS with the following AMI. You can choose another if you wish, however, the following steps may be different.

- Select the t2.**micro** instance type. This is the free tier option as of time of writing.

- Select “create a new key pair”, give an appropriate name and select the options shown. This is how you’ll SSH (gain entry) into the server instance. We won’t be using a password as this is the less secure method of authentication. I will show you how to use keys later in this guide.

- Set your Network settings to the below. These will ensure only machines from your IP address can SSH into the web server, but so anyone can use HTTP or HTTPS and access to the website. If you use a dynamic public IP address (like most home networks), you will need to update this every so often in the instance security group settings.

- Leave the storage configuration as default.

- Double check the configurations and, when you’re happy, select ‘Launch Instance’.

**Accessing the server using SSH keys:**

Using keys to access servers is the preferred method of authentication. This is because passwords are human created and thus can be guessed, socially engineered or worse, leaked in a database if reused. Therefore, it’s much safer to use use public and private keys. Their exact workings will be explained in another writing, but just know a **public** key is stored on the server and each administrator (in this case you) is given a **private** key that is used to authenticate. **THE PRIVATE KEY MUST NOT BE SHARED!** Like your house keys, keep this key in a safe place and don’t give anyone else access to it.

- Open terminal or command line on your machine. (I use a Linux OS, so file locations and 

- Navigate to your Download folder.

```bash
cd Downloads
```

- Now, you'll want to move your key to a folder where you know it is. I tend to keep all mine in the .ssh folder.

```bash
mv webserver_keys.pem ~/.ssh
```

- Move into that folder and set up a config to make the connection easier in future.

```bash
cd ../.ssh;
nano config 
```

- Add an entry with your the configurations to your new server like so.

```bash
Host webserver
   HostName ec2-xx-xxx-xx-xx.eu-west-2.compute.amazonaws.com #Public DNS found on EC2 instance network page.
   User ubuntu
   IdentityFile /home/camospecs/.ssh/'webserver_keys.pem' #Use you correct path.
```

- Once you have added the config, you need to change the permissions of your key to make it more secure.

```bash
chmod 400 webserver_keys.pem #Set to read only for user. 
```

- Finally, to gain access to the server, thanks to your config, simply type and select yes to any prompts about fingerprints:

```bash
ssh webserver
```

Well done! You are now in your web server!

**General server setup and  nginx**

Once on your new, fresh, Ubuntu server, you want to make sure that it's updated and upgraded.

- The following command fetches a list of all packages and the latest information and install their upgrades.

```bash
sudo apt update && sudo apt upgrade -y
```

- Install nginx. The software that will be used to make a reverse proxy and serve the website.

```bash
sudo install nginx -y
```

- Start the web server by typing the following command. You'll now be able to navigate to the default nginx web page using your IP address and a browser. Remember to use HTTP as HTTPs won't work.

```bash
sudo systemctl start nginx
```

- If this is working, great news! Now, you can edit and save the html file to make your own static website!

```bash
cd /var/www/html
sudo nano index.nginx-debian.html

```

- Remove the existing html and add your own. Use this example generated by AI if you wish:
 
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My First HTML Page</title>
</head>
<body>
    <header>
        <h1>Welcome to My Website</h1>
    </header>
    
    <nav>
        <ul>
            <li><a href="#home">Home</a></li>
            <li><a href="#about">About</a></li>
            <li><a href="#contact">Contact</a></li>
        </ul>
    </nav>
    
    <section id="home">
        <h2>Home</h2>
        <p>This is the homepage. Here, you can find an introduction to the website.</p>
    </section>
    
    <section id="about">
        <h2>About</h2>
        <p>This section provides information about the purpose of the website and its creators.</p>
    </section>
    
    <section id="contact">
        <h2>Contact</h2>
        <p>You can reach us via email at <a href="mailto:example@example.com">example@example.com</a>.</p>
    </section>
    
    <footer>
        <p>&copy; 2024 My Website</p>
    </footer>
</body>
</html>

```

- If you refresh the web page your new website will appear!