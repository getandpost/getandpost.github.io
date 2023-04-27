# PHP使用imagick和ImageMagick生成长图给移动端分享

### 背景
因为项目需要，产品要求移动端应用分享到微信朋友圈支持分享长图。理由是别人的应用都支持这个功能--! 

解决思考有两个：
- 1.移动端生成，好处是让客户端分担一部分工作减轻服务器压力。原理用webview，就是弹出一个页面，然后移动端在顶层展示个正在生成图片的Loading这样盖住。其实内里是加载服务端给前端的样式网页，然后加载完毕前端生成图片，生成后分享。或者H5画布生成base64位，调用js注入方式给移动端；
- 2.服务端生成长图。查了大量资料，因为局限于后端语言是PHP，没有直接的方案。原理是先生成静态页，配合使用mpdf库转PDF，然后使用imagick扩展和 imagemagick服务端读PDF转图片。

### 系统环境 
服务器 Ubuntu 20.04 LTS   
PHP 7.4.3-4ubuntu2.18 (cli) (built: Feb 23 2023 12:43:23) ( NTS )  
nginx version: nginx/1.18.0 (Ubuntu)  

### 安装 imagick扩展 和 ImageMagick程序
在Ubuntu上安装，先更新包
```
sudo apt update
sudo apt upgrade -y
```

### 一、安装 imagemagick
```
sudo apt install imagemagick
```

### 二、安装 PHP 扩展 ImageMagick
```
sudo apt install php-imagick
```

使用以下命令验证安装：
```
php -m | grep imagick
```

测试 ImageMagick。

一旦成功安装，现在我们测试 ImageMagick 是否工作是使用转换标志：
```
sudo convert logo: logo.gif
```

### 三、安装imagick
因为需要编译安装，命令比较多，所以还是先sudo到root用户方便操作
```
apt install pkg-config libmagickwand-dev -y
```

到pecl.php.net搜索imagick扩展压缩包 [传送门](https://pecl.php.net/package/imagick)

下载压缩包
```
wget https://pecl.php.net/get/imagick-3.4.4.tgz
```

解压安装包:
```
tar xvzf imagick-3.4.4.tgz
```
进入解压后的目录:
```
cd /mnt/data/software/imagick-3.4.4/
```
运行phpize命令:
```
phpize
```
运行configure命令:
```
./configure
```
编译安装:
```
make && make install
```

一般情况下会在mods-available目录下生成imagick.ini文件。如果没有就自行添加：
```
touch imagick.ini
vim imagick.ini
# 添加以下内容
echo ; configuration for php imagick module >> imagick.ini
echo extension=imagick.so >> imagick.ini
```

修改修改php.ini开启扩展
```
cd /etc/php/7.4/fpm/conf.d
ls -n /etc/php/7.4/mods-available/imagick.ini 20-imagick.ini
cd /etc/php/7.4/cli/conf.d
ls -n /etc/php/7.4/mods-available/imagick.ini 20-imagick.ini
```

重启php和nginx
```
service php7.4-fpm restart
service nginx restart
```

### 四、写业务代码测试test/testPdfToImage，
```
public function testPdfToImage()
    {
        $PDF = ROOT_PATH . '/API/pdf/test.pdf';
        if(!extension_loaded('imagick')){
            return false;
        }
        if(!file_exists($PDF)){
            return false;
        }
        $IM = new \imagick();
        $IM->setResolution(120,120);
        $IM->setCompressionQuality(100);
        $IM->readImage($PDF);
        foreach ($IM as $Key => $Var){
            $Var->setImageFormat('png');
            $Filename = ROOT_PATH . '/API/pdf/'.md5($Key.time()).'.png';
            if($Var->writeImage($Filename) == true){
                $Return[] = $Filename;
            }
        }
        $image_url = UPLOAD_PATH . 'images/'.'merge_' . md5($Key.time()).'.png';
        ImageHandle::mergeImage($Return,$image_url);
        print_r($image_url);
    }
```
结果报错:
```
attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/408
```
解决方案是编辑policy.xml文件[参考传送](https://stackoverflow.com/questions/63988719/attempt-to-perform-an-operation-not-allowed-by-the-security-policy-pdf-error)
```
vim /etc/ImageMagick-6/policy.xml
# 把PDF的rights 改为 read/write
<policy domain="coder" rights="read|write" pattern="PDF" />
```
再次重启php和nginx即可