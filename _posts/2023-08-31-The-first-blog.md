---
layout: post
title: 第一篇博客
categories: [blog ]
tags: [Java,]
description: "测试"
image:
  feature: windows.jpg
  credit: Azeril
  creditlink: azeril.com
 
---

`测试`

![](/img/swirl/11.jpg)

```php
<?php

function waf($filename){
    $black_list = array("ph", "htaccess", "ini");
    $ext = pathinfo($filename, PATHINFO_EXTENSION);
    foreach ($black_list as $value) {
        if (stristr($ext, $value)){
            return false;
        }
    }
    return true;
}

if(isset($_FILES['file'])){//
    $filename = urldecode($_FILES['file']['name']);
    $content = file_get_contents($_FILES['file']['tmp_name']);
    if(waf($filename)){
        file_put_contents($filename, $content);
    } else {
        echo "Please re-upload";
    }
} else{
    highlight_file(__FILE__);
}
```



<center>
    <img src="/img/swirl/11.jpg">
</center>
