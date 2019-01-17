---
title: HttpClient getContentLength()问题
link: http://www.axiomaster.com/index.php/2018/04/17/httpclient-getcontentlength/
author: axiomaster_lism
description: 
post_id: 106
created: 2018/04/17 14:34:25
created_gmt: 2018/04/17 14:34:25
comment_status: open
post_name: httpclient-getcontentlength
status: publish
post_type: post
---

# HttpClient getContentLength()问题

java中使用HttpClient的getContentLength方法获取相应体大小时，如果服务端进行了gzip压缩，响应中可能不包含Content-Length字段。该方法无效。