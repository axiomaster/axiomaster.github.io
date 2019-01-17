title: 修改nginx代码支持从文件读取密文密码
link: http://www.axiomaster.com/index.php/2018/04/17/ngx-read-pwd-from-file/
author: axiomaster_lism
description: 
post_id: 110
created: 2018/04/17 14:47:31
created_gmt: 2018/04/17 14:47:31
comment_status: open
post_name: ngx-read-pwd-from-file
status: publish
post_type: post

# 修改nginx代码支持从文件读取密文密码

主要修改的是event模块下的ngx_event_openssl.c文件。将原来的密码字符串当做文件名来处理，然后使用nginx的读取文件功能，读出密码。