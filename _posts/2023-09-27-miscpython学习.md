---
layout: post
title: misc python脚本学习之路
categories: [blog ]
tags: [MISC,PYTHON,]
description: "misc"
image:
  feature: windows.jpg
  credit: Azeril
  creditlink: azeril.com
 

---

`测试`

![](/img/swirl/11.jpg)

## 这里也是复现的一道misc题

```
就是脚本批量读取二维码数据，所以需要判断图片然后 扫描二维码
```

```python
import os
import string
import image
from PIL import Image
import pyzbar.pyzbar as pyzbar
import os

def name(String):
    imgs=[]
    list =os.listdir(String)
    for i in list:
        ext=os.path.splitext(i)
        if(len(ext)>1 and jpgimage(ext[1])):
            imgs.append(i)


    return imgs

#扫描批量的二维码

def qrcode_parse_content(img_path):
    '''
    单张图片的二维码解析
    '''
    img = Image.open(img_path)

    # 使用pyzbar解析二维码图片内容
    barcodes = pyzbar.decode(img)

    # 打印解析结果，从结果上可以看出，data是识别到的二维码内容，rect是二维码所在的位置
    # print(barcodes)
    # [Decoded(data=b'http://www.h3blog.com', type='QRCODE', rect=Rect(left=7, top=7, width=244, height=244), polygon=[Point(x=7, y=7), Point(x=7, y=251), Point(x=251, y=251), Point(x=251, y=7)])]

    result = []
    for barcode in barcodes:
        barcode_content = barcode.data.decode('utf-8')
        result.append(barcode_content)

    return result




def jpgimage(ext):

    ext=ext.lower()
    if(ext==".jpg") :
        return True
    if(ext==".gif"):
        return True
    if(ext==".png"):
        return True
    return False

print(name("C:\\Users\\c'x'k\\Desktop\\桌面壁纸"))
print(qrcode_parse_content("E:\\Tobby\\erwei.png"))
print(qrcode_parse_content("E:\\Tobby\\erwei.png")[0])



['bay.jpg', 'wallhaven-9mjoy1.png', 'wallhaven-l8kggq.jpg', 'Zero.png']
['JYcxk']
JYcxk
```

