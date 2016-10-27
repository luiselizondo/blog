---
layout: post
title: Move files from multiple drupal sites to the site they belong
date: '2012-07-10 21:46:00'
---

<p>A long time ago I had a multisite installation. Eventually I moved all sites to their own directories but the files directory, once shared by all sites, remained as one.</p>
<p>Now I&#8217;m moving all sites to a new server and right now is the perfect opportunity to move files where they belong.</p>
<p>The problem is, how to know which file belongs to what site? If you take a look at the structure of the directory there&#8217;s just no way to tell.</p>
<p>But Drupal is just amazing, why? Because it keeps record of all the files needed in a table. This way we can get a list of all the files needed for that particular site and copy/move the files to the site&#8217;s directory. Yes, we can do this manually, but is no good if we&#8217;re talking about thousands of files.</p>
<p><strong>Export the files needed by the site</strong></p>
<p>With MySQL we just do:</p>
<p>D6:<br />SELECT filepath FROM files INTO OUTFILE &#8216;/tmp/files.txt&#8217;;</p>
<p>D7:<br />SELECT filepath FROM managed_files INTO OUTFILE &#8216;/tmp/files.txt&#8217;;</p>
<p>If we take a look at files.txt we&#8217;ll see we have a huge list of files, something like:</p>
<p>sites/default/files/talis1.jpg<br />sites/default/files/pic_02.jpg<br />sites/default/files/pic_03.jpg<br />sites/default/files/pic_05.jpg<br />sites/default/files/pic_07.jpg<br />sites/default/files/pic_09.jpg</p>
<p>Now, remember I said I was moving the sites to a new server? Well I had to move the files too and for my own sake, I recreated the structure of the files in /srv so I have something like:</p>
<p>/srv/www/sites/default/files/tails1.jpg</p>
<p>This way I save myself a few lines in the script.</p>
<p><strong>The script</strong></p>
<p>The script is really simple, it does three things and is written in Python.</p>
<p>1. Iterate over each line of the file files.txt</p>
<p>2. Strip the n that MySQL added on each line when we exported the file</p>
<p>4. Move the file to the final destination</p>
<p>#!/usr/bin/python<br />from subprocess import call<br />for line in open(&#8220;/tmp/files.txt&#8221;, &#8220;r&#8221;):<br />    file = line.strip()<br />    print &#8220;Moving file &#8221; + file<br />    call([&#8220;mv&#8221;, &#8220;/srv/www/&#8221; + file, &#8220;/srv/www/example.com/files&#8221;])</p>
<p>That&#8217;s it. Save the script as whatever.py, give it permission to execute (chmod u+x) and run it. If you can&#8217;t move files around in those directories you have to run the script as sudo. You&#8217;ll need to create any directory that you doesn&#8217;t exists too.</p>