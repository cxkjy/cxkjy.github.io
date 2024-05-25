JAVA

### DocToolkitApplication

最明显的一个洞是在这个路由下，后来修复直接return固定值

![](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240507234849654.png)

赛后听说不只一个漏洞点，怪不得修了以后还有几支队伍一直打

源码中有CC、CB、shiro，很容易想到shiro反序列化攻击

![image-20240507235020646](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240507235020646.png)

/doLogin路由传参可控，username、password、rememberMe妥妥shiro

```java

    @PostMapping({"/doLogin"})
    public String doLogin(@RequestParam("username") String username, @RequestParam("password") String password, @RequestParam(name = "rememberMe", defaultValue = "") String rememberMe) {
        try {
            SecurityUtils.getSubject().login(new UsernamePasswordToken(username, password, rememberMe.equals("rememberMe")));
            return "forward:/admin/home";
        } catch (AuthenticationException e) {
            return "forward:/admin/loginFailed";
        }
    }

    @RequestMapping({"/loginFailed"})
    public String loginFailed() {
        return "loginFailed";
    }
}
```

#### 小插曲 😯端口被占用

![image-20240507235800741](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240507235800741.png)

### 1、发包验证有shiro

![image-20240508000028969](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240508000028969.png)

### 2、简单看了下发现貌似有SPEL表达式注入攻击（去除冗余执行表达式注入攻击 ）

![image-20240508000537360](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240508000537360.png)

透露了账号密码，admin/123456登录进去 显示“这不是你来的地方”其它没信息

![image-20240508001639115](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240508001639115.png)



密钥尝试了发现不对

![image-20240508001235129](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240508001235129.png)

### 3、还有一个文件上传，一眼必有洞

![image-20240508002413346](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240508002413346.png)



### 1、先分析shiro漏洞

就差一个密钥了，lib中CC、CB包都是有的可以直接打链子



### 2、分析文件上传漏洞

```java
简单的上传文件，但实现逻辑很奇怪像是为了漏洞而编写
new Yaml().dump(new ExifParser(filePath).exifResult, new FileWriter(new File(parsePath + File.separator + fileName)));
```



```java
 @PostMapping({"/upload"})
    public String handleFileUpload(@RequestParam("uploadFile") MultipartFile file, Model model) {
        try {
            String fileName = file.getOriginalFilename();
            if (fileName == null || fileName.endsWith(".jpg") || fileName.endsWith(".png") || fileName.endsWith(".jpeg") || fileName.endsWith(".gif")) {//为null或者图片后缀进入
                String userDir = Crypt.md5(fileName);//name加密
                String uploadPath = File.separator + "tmp" + File.separator + UPLOAD_DIRECTORY + File.separator + userDir;//    /tmp/upload/md5值
                String parsePath = OUTPUT_DIRECTORY + File.separator + userDir;//   /output/md5
                File uploadDir = new File(uploadPath);
                if (!uploadDir.exists()) {
                    uploadDir.mkdirs();
                }
                File parseDir = new File(parsePath);
                if (!parseDir.exists()) {
                    parseDir.mkdirs();
                }
                String filePath = uploadPath + File.separator + fileName;//一个拼接
                file.transferTo(new File(filePath));//复制操作
                new Yaml().dump(new ExifParser(filePath).exifResult, new FileWriter(new File(parsePath + File.separator + fileName)));
                return "redirect:/image/display?imageName=" + fileName;
            }
            model.addAttribute("title", "EXIF Parser");
            model.addAttribute("description", "Upload Image To Parse");
            model.addAttribute("message", "Only .jpg/.png/.jpeg/.gif files are allowed");
            model.addAttribute("action", "/image/upload");
            return UPLOAD_DIRECTORY;
        } catch (Exception ex) {
            model.addAttribute("title", "EXIF Parser");
            model.addAttribute("description", "Upload Image To Parse");
            model.addAttribute("message", "Something went wrong.");
            model.addAttribute("action", "/image/upload");
            ex.printStackTrace();
            return UPLOAD_DIRECTORY;
        }
    }

```

![image-20240525165847381](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240525165847381.png)
