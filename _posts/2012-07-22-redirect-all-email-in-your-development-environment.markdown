---
layout: post
title: Redirect all email in your development environment
date: '2012-07-22 14:52:42'
---

<p>Quick post. Scenario: You have a Drupal site running on production and as you should be doing it, all the development is done in a different server but you use the database with all the real users and their emails. How can you prevent the real users from receiving all the emails?</p>
<p>In the drupal site you&#8217;ll need the mail_redirect module. Is really easy to install. Basically what it does is just redirect all email to a domain, so if you have luis@gmail.com you&#8217;ll redirect emails to luis@mydomain.com</p>
<p>The drupal module does it really good, but you still have a problem, you&#8217;ll probably want to read all those emails, right? Well, easy to do it, all you have to do is create the file:</p>
<p><code>$ sudo nano /etc/postfix/virtual</code></p>
<p>and add the following line:</p>
<p><code>@mydomain.com luis@myrealdomain.com</code></p>
<p>So basically a multiple redirection will happen. The drupal module will redirect all real email to users who probably doesn&#8217;t exist in your server, it will change the real domain to the domain you specify in configuration of the module. If you have billgates@microsoft and timcook@apple.com as your users, all email will go to billgates@mydomain.com and timcook@mydomain.com and those emails will be lost.</p>
<p>With postfix you redirect emails sent to any-user@mydomain.com to your real email address, luis@myrealdomain.com, this could be even an external domain, like @gmail.com</p>
<p>After that edit the file: <code>/etc/postfix/main.cf</code> as root and add the following:</p>
<p><code>virtual_alias_domains = mydomain.com<br />virtual_alias_maps = hash:/etc/postfix/virtual</code></p>
<p>Finally, just run as root</p>
<p><code>$ postmap /etc/postfix/virtual<br />$ service postfix reload</code></p>
<p>And test, you should receive all email from your site to the email you specified.</p>
<p>One more thing, is probably not a good idea to redirect all email to your main email address, you&#8217;ll get lots of spam.</p>