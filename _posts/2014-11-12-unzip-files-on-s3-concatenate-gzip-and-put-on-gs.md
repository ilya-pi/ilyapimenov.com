---
layout: post
author: <a href="http://ilyapimenov.com/">Ilya Pimenov</a>
title: Unzip files on S3, concatenate, gzip and put on Google Cloud Storage
comments: true
categories:
- blog
---

*Now this is not such a big task, yet it took me surprisingly long time, therefore I've decided to write this.*

What happens — we have these huge zipped log files for analysis in our [Amazon S3](http://aws.amazon.com/s3‎) bucket on Amazon and we wanted to try processing them on [Google Big Query](https://cloud.google.com/bigquery), which supports only gzip as a matter of compression.

Overall — zip is a bad format when it comes to big data, the following [issue](https://issues.apache.org/jira/browse/MAPREDUCE-210) — *"want InputFormat for zip files"* been in progress with _major_ priority for the past __7 years__. 

Cascading had a support for ZIP sometime back, but it was removed in later releases. Please reference the issue thread for reasoning on the topic.

Nevertheless, we have about 2k+ files, 250Mb average size in ZIP, that we'd love to see ready for analysis and further processing in BigQuery. Which leaves you with __0.5 Terabyte__ to copy, unzip, gzip and upload.

# Unzip files on S3, concat and save in gzip

For this reason it calls for some hadoopy solution, and luckily enough there is just one — [Hadoop Streaming](http://hadoop.apache.org/docs/r1.2.1/streaming.html). Combining it together with ZipStream [library](https://bitbucket.org/tebeka/zipstream) written by Miki Tebeka:

{% highlight bash %}
# /opt/hadoop/bin/hadoop jar /opt/hadoop-1.2.1/contrib/streaming/hadoop-streaming-1.2.1.jar -libjars zipstream-1.1-SNAPSHOT.jar -mapper /bin/cat -reducer /bin/cat -inputformat com.mikitebeka.mapred.ZipInputFormat -input s3n://screen6-zip-bucket -output s3n://screen6-gzip-bucket/result
{% endhighlight %}

__Note:__ Be sure to use some a path with a bucket name, like `-output s3n://screen6-gzip-bucket/result`, not `-output s3n://screen6-gzip-bucket/`; streaming utility silently fails on those with only one error message: `Streaming Command Failed!`.

We also enabled the following options that were particularly usefull:

| Option | Meaning |
|:---------:|:------------:|
| -D mapred.output.compress=true | Enable compression |
-D mapred.output.compression.codec = org.apache.hadoop.io.compress.GzipCodec | Use gzip for compression |
| -D mapred.max.map.failures.percent=10 | Allow 10% map job to fail. Files are sent in by third parties and it's not uncommon to see that some are corrupted | 

Combining that together with a wrapper script that will use a file as a source of input files `process-zips.sh`:

{% highlight bash %}
#!/bin/bash

# Pass in $1 file with all the s3 paths you wish to process
# like:
# s3n://mybucket/file1
# s3n://mybucket/file1
# ...
# $2 number of lines per one job

# Create new file handle 5
exec 5< $1

# Step counter
n=0

# Now you can use "<&5" to read from this file
while read lines <&5 ; do
	for i in $(seq 1 $2) ; do read line <&5; lines+=','$line ; done

	nohup /opt/hadoop/bin/hadoop jar /opt/hadoop-1.2.1/contrib/streaming/hadoop-streaming-1.2.1.jar -libjars zipstream-1.1-SNAPSHOT.jar -D mapred.max.map.failures.percent=25 -D mapred.output.compress=true -D mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec -mapper /bin/cat -reducer /bin/cat -inputformat com.mikitebeka.mapred.ZipInputFormat -input $lines -output s3n://screen6-gzip/$1-part$n >$1-part$n.log 2>&1 &

	n=`expr $n + 1`
done

# Close file handle 5
exec 5<&-
echo "Finished in $n$ parts"
{% endhighlight %}

Launch with:

{% highlight bash %}
$ ./process-zips.sh fileA 10
{% endhighlight %}
and then enjoy it with:

{% highlight bash %}
$ tail -F fileA-part*

==> part8.log <==


==> fileA-part15.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar2317634900187317714/] [] /tmp/streamjob4578016248914426911.jar tmpDir=null

==> fileA-part10.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar619398222690999006/] [] /tmp/streamjob5237043347834171306.jar tmpDir=null

==> fileA-part16.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar5691853025461419129/] [] /tmp/streamjob2661134060791365061.jar tmpDir=null

==> fileA-part13.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar6780268392176885393/] [] /tmp/streamjob7066625037729937192.jar tmpDir=null

==> fileA-part17.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar1819205972838198675/] [] /tmp/streamjob4559632407084620827.jar tmpDir=null

==> fileA-part9.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar2117775095566780310/] [] /tmp/streamjob2182082276684486804.jar tmpDir=null

==> fileA-part12.log <==
packageJobJar: [/mnt/hadoop/hadoop-hduser/hadoop-unjar7321821655058192548/] [] /tmp/streamjob7774227815593783167.jar tmpDir=null
{% endhighlight %}

# Copy to Google Cloud Storage

In order to move data from S3 or HDFS to Google Cloud Storage (`gs://`) we'll utilize another hadoop-family tool that's been around for some time — [distcp](http://hadoop.apache.org/docs/r0.19.0/distcp.html).

First of all you will need to configure the Google Cloud Storage Connector for Hadoop, in order to do so, please follow this [guide](https://cloud.google.com/hadoop/google-cloud-storage-connector).

Once it's done, should be as simple as:

{% highlight bash %}
# /opt/hadoop/bin/hadoop distcp2 s3n://my/bucket/here gs://my/bucket/there
{% endhighlight %}

__Note:__ Be sure to use `distcp2` instead of `distcp`. For whatever the reason `distcp` doesn't correctly resolve the classpath, no matter what you do and you are very likely to end up with:

{% highlight java %}
java.lang.RuntimeException: java.lang.ClassNotFoundException: com.google.cloud.hadoop.fs.gcs.GoogleHadoopFileSystem
	at org.apache.hadoop.conf.Configuration.getClass(Configuration.java:857)
	at org.apache.hadoop.fs.FileSystem.createFileSystem(FileSystem.java:1440)
	at org.apache.hadoop.fs.FileSystem.access$200(FileSystem.java:67)
	at org.apache.hadoop.fs.FileSystem$Cache.get(FileSystem.java:1464)
	at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:263)
	at org.apache.hadoop.fs.Path.getFileSystem(Path.java:187)
	at org.apache.hadoop.mapred.JobHistory$JobInfo.logSubmitted(JobHistory.java:1761)
	at org.apache.hadoop.mapred.JobInProgress$3.run(JobInProgress.java:709)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:415)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1190)
	at org.apache.hadoop.mapred.JobInProgress.initTasks(JobInProgress.java:706)
	at org.apache.hadoop.mapred.JobTracker.initJob(JobTracker.java:3890)
	at org.apache.hadoop.mapred.EagerTaskInitializationListener$InitJob.run(EagerTaskInitializationListener.java:79)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
{% endhighlight %}

Voilà

# Links

1. [Hadoop Streaming utility](http://hadoop.apache.org/docs/r1.2.1/streaming.html)
1. [ZipStream library](https://bitbucket.org/tebeka/zipstream)
1. [Hadoop Distributed Copy](http://hadoop.apache.org/docs/r0.19.0/distcp.html)
1. [Google cloud storage connector](https://cloud.google.com/hadoop/google-cloud-storage-connector)


_&#91;Originally published in [Screen6 Technical Blog](http://techblog.s6.io/blog/2014/11/12/unzip-files-on-s3-concatenate-gzip-and-put-on-gs.html) on 11/12/2014&#93;_