---
layout: post
title:  "Tutorial: Cloudflare Logpush with Google Cloud Storage"
ref:  20200515a
date:   2020-05-15 09:00:00 +0800
categories: cloudflare analytics
tags: logs cloudflare logpush
lang: en
---

In [this previous article][logpull] I have talked about how to pull Cloudflare raw logs periodically and save them in your choice of storage server. That's one way of retrieving Cloudflare logs. However more recommended way is to set a Logpush endpoint so you do not have to even run the script, Cloudflare pushes logs periodically to your storage, every 5 minutes.

# Prerequisite
{:.no_toc}

- Cloudflare Enterprise Zone & Dashboard Access (Raw log feature is enterprise only)
- Your Ready-to-use Cloud Storage (has to be one of the below - as of May 15 2020.)
  - Google Cloud Storage
  - AWS S3
  - Microsoft Azure
  - Sumologic (Sumo is a SIEM, but can directly get logs from Cloudflare.)

I'm using Google Cloud Storage here.

# Steps
{:.no_toc}

* TOC
{:toc}

## Decide your storage folder/path that you would like to receive Cloudflare logs.

   At Google Cloud Storage, I've created a storage `jean-logstream` and I'll be receiving my Cloudflare logs in a folder `http-requests`. The console should look like below.

   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-11.52.53-AM.png)

## Navigate to Cloudflare dashboard, Analytics-Logs.

   You should see a panel like below.

   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-11.39.51-AM.png)

## Select GCS.

Click `Connect a service`. Choose `Google Cloud Storage`. Click `Next`.

   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-11.56.31-AM.png)

## Configure the storage path you've created at step 1. 

In my case, `jean-logstream/http-requests`. Replace this with your path. For daily subfolders, I'd recommend you select `Yes, automatically organize logs in daily subfolders`, because you're supposed to receive quite a lot of number of JSON files each day.

   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-11.57.52-AM.png)

## Add a IAM user at your GCS.

This is to grant Cloudflare access to upload files in your bucket. 

Step for this could be different based on storage product you've chosen. For Google Cloud Storage, follow below steps.

a. Copy `Cloudflare IAM user to add`

b. Navigate to your storage. In my case, it is `jean-logstream`. Navigate to `Permissions` tab at your storage.

![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-12.05.05-PM.png)

c. Click `Add Members`.

d. Paste Cloudflare IAM user at `New Members`. Select `Storage Object Admin` as a role. Click `Save`.

![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-12.06.36-PM-1.png) 

Once this is done, at Cloudflare Logpush, click `Validate Access`. If the step is done correctly, it will show `Validated` and bring you to the next step.

## Provide Cloudflare Logpush an Ownership Token.

   This is to prove your ownership of the storage. Since IAM access is granted, Cloudflare would've added a ownership challenge file in your storage folder. 
   
   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-12.08.56-PM.png)
   
   Navigate to your Google Storage. In my case, I should go to `jean-logstream/http-requests`. There should be a new file added by Cloudflare.

   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-12.12.51-PM.png)

   Copy-paste the content of the file to Cloudflare Logpush, and click `Prove Ownership`.

   ![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-12.13.59-PM.png)

## Select which set of logs you'd like to be pushed. 

As of 15 May 2020, below are available data.

- HTTP Requests : Raw logs from Cloudflare's standard product for HTTP/S applications. Have security +  performance + detailed information of what happened.
- Spectrum Events : Event logs from Cloudflare Spectrum, for non-web TCP/UDP applications.
- Firewall Events(API only now) : Security events for Cloudflare's standard product for HTTP/S applications.

I'll choose HTTP requests for this tutorial. Once you select `HTTP Requests`, you can select which fields you'd like to be pushed. You can start from `All fields`, and optimize further based on your needs.
   
Below are advanced options.

- Timestamp format: use `RFC3339` unless you have specific needs to get it in unix format.
- Sampling: If you wish to get randomly sampled % of logs instead of 100%, decide sample rate here.

## Well done! 

Now you should see Cloudflare logs coming through at your storage.

![](https://jeann.net/wp-content/uploads/2020/05/Screenshot-2020-05-27-at-12.22.29-PM.png)

Note that if your storage becomes unavailable, or if you delete Cloudflare IAM user, this console will show error and you'll be missing Cloudflare logs during that time.



[logpull]: https://blog.jeann.net/cloudflare/logs/2019/03/19/pulling-cloudflare-logs-with-bash.html
