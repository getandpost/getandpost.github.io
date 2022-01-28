# 2022-01-27-Fastadmin后台模块不存在

后台模块不存在，配置了支持pathinfo的规则，Nginx伪静态规则配置修改

原本nginx配置：
```
location ~ \.php$ {
	fastcgi_pass   unix:/run/php-fpm/www.sock;
	fastcgi_index  index.php;
	fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
	include        fastcgi_params;
}
```
修改为：
```
location ~ .php(.*)$ { # 正则匹配.php后的pathinfo部分
	# 省略部分配置
	fastcgi_param PATH_INFO $1; # 把pathinfo部分赋给PATH_INFO变量
	# 省略部分配置
}
```