---
layout: post
author: <a href="http://ilyapimenov.com/">Ilya Pimenov</a>
title: FTP Proxy to Google Cloud Storage
comments: true
categories:
- blog
---

# Introduction

After reading this article you will be able to setup an FTP server on a Google Cloud Compute instance, that uploads/downloads content directly to/from Google Cloud Storage. For EC2 backed with S3 refer to [1].

# Step1: Setting up Google Cloud Engine instance

To setup an instance, install `gcloud`, Google Cloud command-line util following these [instructions](https://cloud.google.com/sdk/).

Then fire up the console and do the following:

{% highlight bash %}
$ gcloud compute instances create ftp-proxy --image ubuntu-1404-trusty-v20141212 --image-project ubuntu-os-cloud --zone europe-west1-b --machine-type n1-standard-1 --tags ftp-proxy ftpproxy
{% endhighlight %}

This will fire up a machine called `ftp-proxy` on your Google Cloud Engine account in zone `europe-west1-b` (change to the one desired).

<div class="note">Looking ahead, It is important to go for ubuntu-1404 image. With earlier versions of ubuntu you wont be able to install gcsfs, due to missing libboost-regex1.49.0 and libboost-thread1.49.0 libraries, while ubuntu-1210 is out of support. <br/><br/>With debian you will have issues configuring vsftpd service that enables ftp access to your server: you won't be able to restrict users to their home directories (<a href="https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=735191#10">bug-735191</a>).</div>

Now in order to ssh into this machine:

{% highlight bash %}
$ gcloud compute ssh ftp-proxy --zone europe-west1-b
{% endhighlight %}

Handy, eh?

# Step 2: Mounting Google Cloud Storage onto the instance

We'll require a thirdparty tool in order to mount Google Cloud Storage to our instance, in particular — [gcsfs](https://code.google.com/p/gcsfs/).

Get the proper download link from their website, you will need this one — `gcsfs_0.15-1_amd64.deb`, and then:

{% highlight bash %}
$ wget https://drive.google.com/uc?id=0B071t7_clY1HdG8zUTc5TGszU0E&export=download
$ mv uc\?id\=0B071t7_clY1HdG8zUTc5TGszU0E gcsfs_0.15-1_amd64.deb
{% endhighlight %}

Install the package itself:

{% highlight bash %}
$ sudo dpkg -i gcsfs_0.15-1_amd64.deb
{% endhighlight %}

Next update the dependencies and work out through the missing libraries (slightly dirty approach, hopefully won't be needed in  the upcoming future):

{% highlight bash %}
$ sudo apt-get update

$ wget http://launchpadlibrarian.net/141626883/libboost-regex1.49.0_1.49.0-4_amd64.deb
$ sudo dpkg -i libboost-regex1.49.0_1.49.0-4_amd64.deb

$ wget http://launchpadlibrarian.net/141626895/libboost-thread1.49.0_1.49.0-4_amd64.deb
$ sudo dpkg -i libboost-thread1.49.0_1.49.0-4_amd64.deb

$ wget http://launchpadlibrarian.net/153428081/libicu48_4.8.1.1-12ubuntu2_amd64.deb
$ sudo dpkg -i libicu48_4.8.1.1-12ubuntu2_amd64.deb

$ sudo apt-get install -f
{% endhighlight %}

Check that everything is fine:

{% highlight bash %}
$ gcsfs -V

>gcsfs, 0.15 (r792M), FUSE driver for cloud object storage services
>enabled services: aws, fvs, google-storage
{% endhighlight %}

Let's authenticate it with Google Cloud Storage:

{% highlight bash %}
$ sudo gcsfs_gs_get_token /etc/gcsfs_token

> Paste this URL into your browser:
>https://accounts.google.com/o/oauth2/auth?client_id=xxxxxxxxxxxx.apps.googleusercontent.com&redirect_uri=urn%3aietf%3awg%3aoauth%3a2.0%3aoob&scope=https%3a%2f%2fwww.googleapis.com%2fauth%2fdevstorage.read_write&response_type=code
>
>Please enter the authorization code:
>
{% endhighlight %}

__Important:__ In the string taht is given to authorize against google cloud (the "Paste this URL into your browser" link) find the text `read_write` and replace it with `full_control`. Otherwise listing in the folders won't work and it is very likely you will have random transient I/O Errors.

Create gcsfs configuration file `/etc/gcsfs_config` and paste in:

{% highlight bash %}
service=google-storage
# Your bucket on Google Cloud Storage, i.e. `xyz-ftp-bucket`
bucket_name=<Bucket-Name>
gs_token_file=/etc/gcsfs_token
{% endhighlight %}

Create mount point and mount the cloud storage:

{% highlight bash %}
$ mkdir /mnt/ftp
$ sudo gcsfs -o config=/etc/gcsfs_config -o allow_other /mnt/ftp
{% endhighlight %}

# Step 3: Setup FTP daemon `vsftpd`

Add the new repository and install with `apt-get`:

{% highlight bash %}
$ sudo add-apt-repository ppa:thefrontiergroup/vsftpd
$ sudo apt-get -y install vsftpd
{% endhighlight %}

Configure it — `sudo nano /etc/vsftpd.conf`, here are the properties you want to set to `YES` in order for each user to have his isolated "directory" on your ftp, without being able to mess up with others:

{% highlight bash %}
# You may restrict local users to their home directories.  See the FAQ for
# the possible risks in this before using chroot_local_user or
# chroot_list_enable below.
chroot_local_user=YES
# Keep non-chroot listed users jailed
allow_writeable_chroot=YES
{% endhighlight %}

Add passive support to your configuration:

{% highlight bash %}
# Passive support
pasv_enable=yes
pasv_min_port=15393 # The start port range configured in the security group
pasv_max_port=15592 # The end port range configured int he security group
pasv_address=xxx.xxx.xxx.xxx # the public IP address of the FTP serv 
{% endhighlight %}

If later on encountered [530 Login incorrect](http://askubuntu.com/questions/413677/vsftpd-530-login-incorrect), rename the pam service to:

{% highlight bash %}
pam_service_name=ftp
{% endhighlight %}

All done, restart the service:

{% highlight bash %}
$ sudo service vsftpd restart 
{% endhighlight %}

# Step 4: Creating new users

Now in order to create a new user `alfred` with home directory `/mnt/ftp/alfred`, which will be persisted under `gs://<Bucket-Name>/alfred` on Google Cloud Storage, ssh into the machine:

{% highlight bash %}
$ gcloud compute ssh ftp-proxy --zone europe-west1-b
{% endhighlight %}

And create the user himself:

{% highlight bash %}
$ sudo useradd -d /mnt/ftp/alfred -s /sbin/nologin alfred
$ sudo mkdir /mnt/ftp/alfred
$ sudo chown alfred /mnt/ftp/alfred
$ sudo passwd alfred
> New password:
{% endhighlight %}

# Step 5: Verification

From your local machine fire this to see it in action — 

{% highlight bash %}
$ ftp ftp://alfred:alfred@130.211.89.90
>put ./<Some-local-file> <File-name-to-put-as-on-Ftp>
{% endhighlight %}

And look at your Google Cloud Storage now:
{% highlight bash %}
$ gsutil ls gs://<Bucket-Name>/alfred/
> gs://<Bucket-Name>/alfred/
> gs://<Bucket-Name>/alfred/<File-name-to-put-as-on-Ftp>
{% endhighlight %}

Yay! You are all set now.

# Links

1. [Google Cloud Platform](https://cloud.google.com/)
1. [Google cloud storage connector](https://cloud.google.com/hadoop/google-cloud-storage-connector)
1. [Amazon AWS: EC2 FTP server using S3 as the backend](http://www.codeproject.com/Tips/546903/Amazon-AWS-EC-FTP-server-using-S-as-the-backend)
