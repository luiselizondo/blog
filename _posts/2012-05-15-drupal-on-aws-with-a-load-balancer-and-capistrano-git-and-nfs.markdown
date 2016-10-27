---
layout: post
title: Drupal on AWS with a Load Balancer and Capistrano, Git and NFS
date: '2012-05-15 03:56:00'
tags:
- aws
- capistrano
- drupal
- drush
- git
- loadbalancer
- nfs
- svn
---

<p>Things can get complicated when you grow older. The same thing happens to your site if you want to handle millions of page requests.</p>
<p>Let me describe you the problem and then I&#8217;ll go into possible solutions.</p>
<p>One website, multiple AWS instances to support the traffic, one NFS server (thinking about moving to GlusterFS BTW) and one big MySQL Dedicated server with no fancy master-slave replication right now.</p>
<p>So what&#8217;s the problem? The problem is maintenance, how do you update this whole thing without having nightmares at night? Possible solutions are:</p>
<ol><li>Put all the Drupal code on the NFS and just use the instances to connect to NFS, one single code-base and multiple instances running apache to support the traffic.</li>
<li>Multiple isolated code-bases just sharing the files directory</li>
</ol><p>Which way to go?</p>
<p>Solution 1, is no solution really, because if NFS fails it will bring down all your sites with it.</p>
<p>Solution 2 seems more reasonable but still presents a number of issues related to maintenance.</p>
<p>What happens when you install a module or upgrade one? Well, for that you could use capistrano to deploy the changes to all your servers and is really easy, no more than 4 lines of code, literally.</p>
<p>But then, capistrano will update one by one your sites, if you have 20 instances it will take a few minutes to do it and with high traffic sites, you could still have problems.</p>
<p>Another possible solution is to use <strong>Git</strong> and have one big site, that could work, but I&#8217;m no fan of having a huge git project, specially because Git can&#8217;t handle git repositories inside a git repository, and I usually clone multiple projects into the site so that&#8217;s a problem. The solution here could be to use SVN but I really never bothered learning SVN (I don&#8217;t think it would take me more than half an hour but I don&#8217;t really want to lean Subversion).</p>
<p>So, what do we do? To be honest, I think the answer depends on your needs, as usual, but neither of the approaches described above are 100% safe and satisfactory. What do you think?</p>