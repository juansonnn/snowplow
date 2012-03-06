# Self-hosting SnowPlow

## Table of Contents

1. [Introduction](#intro)
2. [Self-hosting the tracking pixel](#pixelsh)
3. [Self-hosting snowplow.js](#jssh)
5. [A note on privacy](#privacy)

<a name="intro"/>
## Introduction

This guide takes you through the process for self-hosting SnowPlow. There are two distinct aspects to self-hosting:

1. **Self-hosting the tracking pixel** - the tracking pixel is served from your Amazon CloudFront distribution, rather than the SnowPlow team's 
2. **Self-hosting snowplow.js** - the SnowPlow JavaScript is hosted and served by your web server(s) 

SnowPlow makes it easy for you to self-host either the S3 pixel or `snowplow.js`, or both. We look at each in turn below - but first please make sure that you have read and understood [Integrating `snowplow.js` into your site] [integrating] - the below will make little sense without.

<a name="pixelsh"/>
## Self-hosting the tracking pixel

### Overview

If the SnowPlow team provide you with an account ID, then the data collected by SnowPlow will be stored within our Amazon S3 account, rather than yours. If you prefer, you can self-host the tracking pixel and store your SnowPlow data within your own Amazon S3 account. This is straightforward to do, and we will explore how to do it in the rest of this section.

### Pre-requisites

If you want to self-host the tracking pixel, you will need the following:

* An account with [Amazon Web Services] [aws]
* S3 and CloudFront enabled within your AWS account
* Some level of technical ability _ta1_, where `noob < ta1 < ninja`

Once you have those ready, please read on...

### Self-hosting instructions

#### 1. Create a bucket for the pixel

First create a new bucket within your Amazon S3 account to store the pixel. Call this bucket `snowplow-static`:

![pixelbucket] [pixelbucket]

A couple of notes on this:

* Don't enable logging on this bucket
* Because we will be using CloudFront, it doesn't particularly matter which data center you choose (although see [A note on privacy](#privacy) below)

#### 2. Create a bucket for the CloudFront logging

Now let's create a second bucket to store our CloudFront logs - i.e. our actual SnowPlow data. Call this bucket `snowplow-logs`:

![logbucket] [logbucket]

Again, no need to enable logging on this bucket.

#### 3. Upload a tracking pixel

You can obtain a 1x1 transparent tracking pixel by right-clicking [this image file] [pixel] and selecting **Save Link As...**, or if you prefer run:

    $ wget https://github.com/snowplow/snowplow/raw/master/tracker/static/ice.png 	

Now you're ready to upload the pixel into S3. Within the S3 pane, hit **Upload** and browse to your tracking pixel:

![pixelchoose] [pixelchoose]

Then hit **Open** and you will see the following screen:

![pixelupload] [pixelupload]

Hit **Set Details >**, then hit **Set Permissions >** to set permissions on this file allowing Everyone to Open it:

![pixelsecurity] [pixelsecurity]

Now hit **Start Upload** to upload the pixel into your bucket.

#### 4. Create your CloudFront distribution

Now you are ready to create the CloudFront distribution which will serve your tracking pixel:

[!createdistrib]

Write down your CloudFront distribution's URL - e.g. `http://d1x5tduoxffdr7.cloudfront.net`. You will need this in the next step.

That's it - you now have a CloudFront distribution which will serve your tracking pixel fast to anybody anywhere in the world and log the request to Amazon S3 in your `snowplow-logs` bucket. 

#### 5. Testing your tracking pixel on CloudFront

Before testing, take a 10 minute coffee or brandy break (that's how long CloudFront takes to synchronize).

Done? Now just check that you can access your pixel over both HTTP and HTTPS using a browser, `wget` or `curl`:

    http://{{SUBDOMAIN}}.cloudfront.net/ice.png
    https://{{SUBDOMAIN}}.cloudfront.net/ice.png

If you have any problems, then double-check your CloudFront distribution's URL, and check the permissions on your pixel: it must be Openable by Everyone.

#### 6. Update your header script

Now you need to update the JavaScript code for SnowPlow in your website's `<head>` section to work with your custom tracking pixel. We're assuming here that you have already followed the steps in [Integrating `snowplow.js` into your site] [integrating].

The secret is to realise that SnowPlow's `setAccount()` method in fact takes a CloudFront subdomain as its argument - so using your own CloudFront distribution is super-simple.

If you are using **asynchronous tracking** and your CloudFront distribution's URL is `http://d1x5tduoxffdr7.cloudfront.net`, then update your header script to look like this:

```javascript
var _snaq = _snaq || [];

_snaq.push(['setAccount', 'd1x5tduoxffdr7']);
...
```

Whereas if you are using **synchronous tracking**, then update your header script to look like this:

**Yali to add**

#### 7. Test snowplow.js with your tracking pixel

**To write**

#### 8. Inspect the CloudFront access logs

**To write**

<a name="jssh"/>
## Self-hosting snowplow.js

### Overview

In addition to self-hosting the tracking pixel, it also possible to self-host the SnowPlow tracking JavaScript, `snowplow.js`. Unlike the tracking pixel, this does not have an impact on where your SnowPlow data gets stored, but it does have some definite advantages over using a SnowPlow-hosted JavaScript: 

1. Hosting your JavaScript allows you to use your own JavaScript minification and asset pipelining approach (e.g. bundling all JavaScripts into one minified JavaScript)
2. As [Douglas Crockford] [crockford] put it about third-party JavaScripts: _"IT IS EXTREMELY UNWISE TO LOAD CODE FROM SERVERS YOU DO NOT CONTROL."_
3. Perhaps most importantly, hosting `snowplow.js` on your own server means that the SnowPlow tracking cookie will be **first-party**, not **third-party**. This is good from a user-privacy perspective, and it also gives better accuracy in counting unique visitors (as first-party cookies are more often accepted and less often deleted by users) 

So if you want to self-host `snowplow.js`, please read on...

### Prerequisites

To self-host `snowplow.js` you will need the following:

* Access to a Unix-like command line (containing tools such as `sed`)
* Familiarity with and access to [Git] [git]
* Some level of technical ability _ta2_, where `ta1 < ta2 < ninja`

Once you have those ready, please read on...

### Self-hosting instructions

#### 1. Check out the source code

First please download the source code to your development machine:

    $ git clone git@github.com:snowplow/snowplow.git
	...
	$ cd snowplow/tracker/js/
	$ ls 
    snowplow.js   sp.js         snowpak.sh

In the listing above, `snowplow.js` is the original JavaScript; `sp.js` is the minified version and `snowpak.sh` is a Bash shell script for performing the minification.

#### 2. Minify the JavaScript

You can minify the 'full fat' version of `snowplow.js` by using `snowpak.sh` if you have **XXX** installed. To do this:

    $ ./snowpak.sh snowplow.js > sp.js

This will overwrite your existing `sp.js`.

In theory it should be possible to use any JavaScript minifier or pipelining tool to minify the JavaScript - however, you would need to read through and understand what `snowpak.sh` is doing and make sure to recreate that same behaviour in your minification process.

#### 3. Upload the minified JavaScript

Use your standard asset pipelining strategy to upload the minified `sp.js` JavaScript to your servers. Note that to avoid "mixed content" warnings, SnowPlow expects the `sp.js` JavaScript to be available both via HTTP and via HTTPS.

#### 4. Update your header script

Now you need to update the JavaScript code for SnowPlow in your website's `<head>` section to use your hosted copy of `snowplow.js`. For the purposes of this section, we're going to assume that you have a minified `sp.js` available at the URL:

    http(s)://bigcorpstatic.com/sp.js

If you are using **asynchronous tracking**, then update your header script to look like this:

```javascript
var sp = document.createElement('script'); sp.type = 'text/javascript'; sp.async = true; sp.defer = true;
sp.src = ('https:' == document.location.protocol ? 'https' : 'http') + '://bigcorpstatic.com/sp.js';
...
```

Whereas if you are using **synchronous tracking**, then update your header script to look like this:

**Yali to add**

Simple as that really.

#### 5. Test your hosted JavaScript

As a final step, you'll want to just check that your self-hosted JavaScript is working okay. To do this:

* Upload a test page (possibly based on `tracker/examples/async.html`) to your server
* 

**To write**

<a name="privacy"/>
## A note on privacy

Above we mentioned that, from a performance perspective, it is not important which Amazon data center you choose to self-host your pixel (or indeed your JavaScript):

**Add in image**

[aws]: http://aws.amazon.com/
[pixel]: /snowplow/snowplow-js/raw/master/tracker/static/ice.png
[pixelbucket]: /snowplow/snowplow/raw/master/docs/images/04_pixel_bucket.png
[logbucket]: /snowplow/snowplow/raw/master/docs/images/04_log_bucket.png
[pixelchoose]: /snowplow/snowplow/raw/master/docs/images/04_pixel_choose.png
[pixelupload]: /snowplow/snowplow/raw/master/docs/images/04_pixel_upload.png
[pixelsecurity]: /snowplow/snowplow/raw/master/docs/images/04_pixel_security.png
[integrating]: /snowplow/snowplow/blob/master/docs/03_integrating_snowplowjs.md
[git]: http://git-scm.com/
[crockford]: https://github.com/douglascrockford