---
layout: post
title:  "Hosting a custom domain for free with github-pages and Zoho"
date:   2017-03-13 17:55:56 -0700
categories: jekyll
---
For quite a while, I ran an Ubuntu server on [Google Cloud][gce], complete with a flask/uwsgi/nginx webserver hosting a [bootstrap](http://getbootstrap.com/) templated site, a postfix server, IRC server, and so on. While it was a good learning experience with GCE and gave me an environment I could always access, I was looking for a less expensive (read: free) way to just host my domain, with a simple blog, and email.

Fortunately, a coworker turned me on to [Github Pages][github-pages]. This free service allows you to use a public github repository to host a personal (or project) site for free, using a custom domain if you so choose. It also supports [Jekyll](https://jekyllrb.com), a "blog-aware, static site generator", allowing you to create great looking blogs with minimal overhead. The second piece, email, I accomplished by creating a free account on [Zoho](https://www.zoho.com/).

So for my first of what will hopefully be more than one total post on this blog, this is a crash course in how to set this type of thing up with minimal frustration.

#### 1. First register a domain, if you don't already have one
I personally have been using [Namecheap][namecheap], but there are lots of options out there, including using Google or AWS as a registrar (all of them seem to charge around $12/year for a .com). Namecheap offers a free basic DNS service, which is helpful later so you can create the relevant records for your domain.

#### 2. Create your personal github repository, pick a jekyll theme and link it to your domain
Follow the instructions from [Github Pages][github-pages] to set up your personal repository. It's basically as simple as creating a repo with the form username.github.io. You can then choose one of many [supported themes](https://pages.github.com/themes/) (mine is called jekyll-theme-minimal).

Then, in the Settings tab of your new repository, under the "Github pages" section, add your domain name under the "custom domain" field and save.

There is extensive documentation and troubleshooting tips on this whole process [available here](https://help.github.com/categories/customizing-github-pages/). 

**Note** If you're planning on developing locally, make sure to install the gems as described in the [Jekyll quickstart](https://jekyllrb.com/docs/quickstart/). However, when you run a "jekyll new . --force" on your cloned repository so you can run the local development server, it will overwrite the _config.yml and index.md with it's defaults. It's not the end of the world, you just need to replace the theme with whatever you chose above in that file and fill in the other defaults, as well as make sure to include the correct theme in your Gemfile. I basically followed [this](https://help.github.com/articles/adding-a-jekyll-theme-to-your-github-pages-site/#adding-a-jekyll-theme-on-the-command-line).

#### 3. Set up DNS to point your domain to Github
If you want to set up an apex domain (that is, one without any www or similar record under it, like I did with rorski.com), you need to set up a record in your DNS provider to point to github, following the [setting up an apex domain][github-apex] instructions.

Github suggests configuring an "ANAME" or "ALIAS" record as a first suggestion. Which is great, if your DNS provider can handle it. Unfortunately, many can't, including Google Cloud and Namecheap. In fact only DNS Made Easy supports ANAME, as they point out themselves:

> Some DNS providers support configuring apex domains with an ALIAS or ANAME record, but there is no industry standard for these. Only DNS Made Easy currently supports ANAME records and DNSimple is one of the few DNS providers that support ALIAS records.

So I went with option two, which is just using a standard A record to point at github. If, like me, you're using Namecheap, you can accomplish this by going to the "Advanced DNS" tab and adding two A records - one that points to 192.30.252.153 and one to 192.30.252.154 with "@" for the host.

Your other option of course is to simply use the default domain with your github username, in the format of username.github.io, which should work without any DNS configuration. But that just doesn't look as good, and it's more fun to tell people to go to your custom blog domain anyway, right?

Finally, since many people will probably default to adding "www" in front of your domain, you probably want to point that at the apex (e.g., rorski.com) zone as well. You can accomplish this by simply adding a CNAME record for www to point to whatever your domain is. 

At this point, your records should look something like this:

	~ > dig A rorski.com +short
	192.30.252.154
	192.30.252.153
	~ > dig www.rorski.com
	www.rorski.com.		296	IN	CNAME	rorski.com.

#### 4. Set up Email
There are obviously a lot of possibilities for email hosting, from running your own servers to paying a bunch of companies that would be happy to take your money to host for you. I personally use Zoho to host my email and I strongly recommend it. For one, it's free (for one domain), reliable, and the web UI is much better than gmail's - or really any other webmail service I have used. You can check out their pricing model [here](https://www.zoho.com/workplace/pricing.html?src=zmail').

I won't go into a lot of detail on the Zoho setup itself, since it's well documented. The important part is, again, setting up DNS so you can send and receive email correctly - most importantly setting up MX records according to [Zoho's directions](https://www.zoho.com/mail/help/adminconsole/configure-email-delivery.html). In my case, that was simply a matter of navigating back to the Namecheap "Advanced DNS" tab, and adding the two servers under the Mail Settings section by selecting "Custom MX" from the drop down. 

You should also set up [SPF](https://www.zoho.com/mail/help/adminconsole/spf-configuration.html) and [DKIM](https://www.zoho.com/mail/help/adminconsole/dkim-configuration.html) to make sure your email isn't blocked by spam filters. Dig your MX and TXT records to confirm they look something like the following:

	~ > dig MX rorski.com +short
	10 mx.zoho.com.
	20 mx2.zoho.com.
	~ > dig TXT rorski.com
	rorski.com.		1800	IN	TXT	"v=spf1 include:zoho.com ~all"

That should be it! Navigating to your new domain should link to your Github Jekyll site, and you should be able to send and receive email from your Zoho account - all for a grand total of $0 per month.

[github-apex]: https://help.github.com/articles/setting-up-an-apex-domain/
[gce]: https://cloud.google.com
[namecheap]: https://namecheap.com
[github-pages]: http://pages.github.com