---
layout: post
date:   2023-10-08 12:21:11 -0400
categories: wordpress
author: Emma
---

It should be easy to move the hosting provider for a wordpress, right? Right?

I thought it was easy, too...until I tried to do it.

(If you want to continue to read my yammering, feel free to read on. [If you want to skip to the instructions, click here.](#The-Instructions))

I found AWS Lightsail documentation for moving a wordpress over. It looked very promising. Lots of pictures, step by step instructions, no required pre-knowledge. [Click here to see the AWS Lightsail documentation.](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/migrate-your-wordpress-blog-to-amazon-lightsail)

So, I started following the instructions. And they were a breeze. Super easy to follow, and didn't take any knowledge for granted. I created a new wordpress instance in Lightsail, and was able to successfully log in. I used the wordpress export tool to create an XML file with info from my old wordpress site.

And then I got to 'Step 4: Import your XML file into your new Lightsail blog'. And I started running into issues. Or rather, a lot of failed imports. StackOverflow and other sites were all telling me how to move over each of the imports that failed, one by one. There were extra plugins, and all sorts of things that I needed to do if I wanted to even TRY to move over the items that failed. I figured that there must be a better way.

And there was. I found a plugin that promised to do just that, called "All-In-One WP Migration". It looked pretty safe, and well-supported. It had been updated in the last month, and had 5+ million installations. 

So, I added the plugin to both the new Lightsail AWS instance, and the old GoDaddy instance. I exported a file from the old GoDaddy instance. And went to import it into the new instance, before BAM! a problem. The file that I was trying to upload was around 800MB, but there was a 80MB limit on the upload. I tried to upload it anyhow, but the plugin was pretty unhappy about that, and displayed a message telling me it didn't work.

The plugin very helpfully had a link for resolving this sort of issue. ([See the page here](https://help.servmask.com/2018/10/27/how-to-increase-maximum-upload-file-size-in-wordpress/)) Of course, I could buy some sort of upgraded version of the plugin. I wasn't going to do that. Or I could contact my hosting provider. I laughed at the idea of trying to contact AWS to ask how to chance my upload file size for a Wordpress Lightsail Container. And option 3 was marked "Hard". It suggested changing the .htaccess file or the wp-config.php file.

Fun fact, the Wordpress image used by Lightsail utilizes Bitnami. And Bitnami does .htaccess files a little differently. ([See here for an explanation from Bitname](https://docs.bitnami.com/aws/infrastructure/lamp/administration/use-htaccess/)) So, I thought I'd go down the wp-config.php route. I added the suggested lines. And nothing.

At this point, I was fairly frustrated, as you can imagine. But I had a very cute yorkipoo (that's a dog who is part yorkie and part poodle) sound asleep on my lap. And I didn't want to get up and disturb him. So, I pressed on.

I inspected the line on the plugin page that showed the file size limit, and from there dug into the plugin code. Plugin code can be viewed and edited from the Plugins -> Plugin File Editor page or by adding `/wp-admin/plugin-editor.php` to the end of your url. I found the function that was producing the info about the file size limit in the `lib/view/import/pro.php` file. 

Line 33:
```
<?php printf( __( 'Maximum upload file size: <strong>%s</strong>.', AI1WM_PLUGIN_NAME ), esc_html( ai1wm_size_format( wp_max_upload_size() ) ) ); ?>
```

I edited the text to say something silly, and when I refreshed the plugin page, my silly text showed up, so I knew I had found the correct source.

And the culprit was definitely the `wp_max_upload_size` function. [The wordpress documentation (and definition) for the function is here.](https://developer.wordpress.org/reference/functions/wp_max_upload_size/) This function basically outputs the minimum of two variables `upload_max_filesize` and `post_max_size`. This was super interesting, because the instructions for changing the `wp-config.php` file had these variables. I was getting closer to an answer.

I ssh'ed to my new Lightsail instance (which btw, there's literally a button in Lightsail to open this in a new browser window). And I searched through my files for one of the variables.

```
grep -Rnw  -e 'upload_max_filesize'
```

It showed this variable in a php.ini file. The other variable was there too! And they were both set to 80MB. I used vim to edit the file. (`vim stack/php/etc/php.in`). I saved. No change. But then I rebooted the server, and the message about the file size was gone!

From there I was able to import the file I exported from the old Godaddy server. And it worked! The new website looked exactly the same as the old one.

The next step will be to change the dns and ssl cert to point to the new server. But that's for tomorrow. I've accomplished enough today.

# The Instructions
### A) Make a wordpress instance in Lightsail
AWS does a much better job of explaining this than I ever will. [Follow steps 1-4 in the tutorial here.](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/amazon-lightsail-tutorial-launching-and-configuring-wordpress)

### B) Install All-in-one WP Migration plugin in origin site
1. Login to your old, origin wordpress site.
2. Go to your plugins page (Plugins -> Add New), and search for All-in-one WP Migration.
3. Open the plugin information and click `Install Now`
4. Navigate to the plugin, and click `Activate`

### C) Install All-in-one WP Migration plugin in new Lightsail instance
Follow the instructions in B, but on your new Lightsail instance.

### 4. Export using plugin from origin site
1. Navigate to the plugin on the origin site, by either adding `/wp-admin/admin.php?page=ai1wm_export` to the end of the url or by hovering over All-in-One WP Migration in the left bar and then clicking `Export`.
2. Click `Export To` and select `File`

### 5. Change the limits for upload_max_filesize and post_max_size in php.ini to bigger than the export
1. Navigate to your instance in AWS Lightsail.
2. Click `Connect Using SSH` or use your terminal to ssh in.

![SSH Button](image.png)

3. Open the php.ini file with your favorite terminal text editor. If you want to use vim, the command will look something like this: `vim stack/php/etc/php.in`
4. Edit the lines that set the `upload_max_filesize` and `post_max_size` variables to be larger than the file size that you downloaded. If you are using vim, you will need to hit the `insert` key to edit the file.
5. Save and close the file. If you are using vim, you will hit escape, then `:wq`.
6. Reboot the instance. 
![Instance Reboot Button](image-1.png)

### 6. Import the exported files
1. Navigate to the plugin on the origin site, by either adding `/wp-admin/admin.php?page=ai1wm_import` to the end of the url or by hovering over All-in-One WP Migration in the left bar and then clicking `Import`.
2. Click `Import From` and select `File`.
3. Select the file.


# Good Luck!
