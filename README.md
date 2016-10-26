macOS Sierra 已经帮我们预装了 Ruby、PHP(5.6)、Perl、Python 等常用的脚本语言，以及 Apache HTTP 服务器。由于 nginx 既能作为 HTTP 服务器也能作为反向代理服务器，且配置简单,这里我们用 nginx 代替 Apache 作为我们默认的 HTTP 服务器。

下面是我在 macOS Sierra 配置的 PHP 开发环境：

### 安装命令行终端 

这里我们选择 [iTerm2](http://www.iterm2.com/)，iTerm2 功能强大，可以替代系统默认的命令行终端。下载解压后，将iTerm2 直接拖入"应用程序"目录。

### 安装 IDE

这里我们选择 [JetBrains PhpStorm](https://www.jetbrains.com/phpstorm/) 作为集成开发环境。这个应该是这个星球最棒的 PHP IDE 了。

### 安装 Xcode

Xcode 是苹果出品的包含一系列工具及库的开发套件。

通过 App Store 安装最新版本的 Xcode。我们一般不会用 Xcode 来开发 PHP 项目。但这一步也是必需的，因为 Xcode 会帮你附带安装一些如 Git 等必要的软件。当然你也可以通过源码包安装 Git。


### 安装 Xcode Command Line Tools

这一步会帮你安装许多常见的基于 Unix 的工具。Xcode 命令行工具作为 Xcode 的一部分，包含了 GCC 编译器。在命令行中执行以下命令即可安装：

```
xcode-select --install
```

### 安装包管理器
 
[Homebrew](http://brew.sh/index_zh-cn.html) 作为 macOS 不可或缺的套件管理器，用来安装、升级以及卸载常用的软件。在命令行中执行以下命令即可安装：

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

安装后可以修改 Homebrew 源，国外源一直不是很给力，这里我们将 Homebrew 的 git 远程仓库改为中国科学技术大学开源软件镜像：

```
cd "$(brew --repo)"
git remote set-url origin git://mirrors.ustc.edu.cn/brew.git
```

### 安装 HTTP 服务器

这里我们选择 nginx 代替系统自带的 Apache，作为我们的 HTTP 服务器：
```
brew install nginx
```

安装完成后，nginx 的一些常用命令：

```
sudo nginx # 启动 nginx 服务
nginx -h # nginx 帮助信息
sudo nginx -s stop|quit|reopen|reload # 停止|退出|重启|重载 nginx 服务
```

### 安装数据库

这里我们选择 MySQL 作为我们的数据库服务器：
```
brew install mysql
```
当然，你也可以选择安装 PostgreSQL 或者 MariaDB。

安装完成后，启动 MySQL：
```
mysqld
```
如果不执行上述操作，直接通过命令行进入 MySQL，一般会报一个 ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock’ 的错误。

进入 MySQL 服务器：
```
mysql -u root -p
```

### 开启 PHP-FPM

nginx 本身不能处理 PHP，它只是个 HTTP 服务器，当接收一个 PHP 请求后，nginx 会将其交由 PHP 解释器处理，并把结果返回给客户端。nginx 一般是把请求发 FastCGI 管理进程处理，FastCGI 管理进程选择 CGI 子进程处理结果并返回被 nginx。

PHP-FPM是一个 PHP FastCGI 管理器，一开始只是 PHP 源代码的一个补丁，旨在将 FastCGI 进程管理整合进 PHP 包中。必须将它 patch 到 PHP 源代码中，在编译安装 PHP 后才可以使用。PHP 从版本 5.3 开始官方集成 PHP-FPM。

添加 PHP-FPM 的配置文件：

```
cp /private/etc/php-fpm.conf.default /private/etc/php-fpm.conf
php-fpm --fpm-config /private/etc/php-fpm.conf
```

修改 PHP-FPM 的 error_log 路径：
```
vi /var/log/php-fpm.log # 新建文件
vi /private/etc/php-fpm.conf # 将 error_log=log/php-fpm.log 修改为：error_log = /var/log/php-fpm.log，保存
```

启动 PHP-FPM：
```
sudo php-fpm
```

关闭 PHP-FPM：
```
ps aux|grep php-fpm
sudo kill php-fpm min pid # 杀死 php-fpm 最小的进程id
```

### 配置 nginx.conf 文件

通过以下命令可以查看 nginx.conf 文件的位置： 
```
nginx -h
```
输出：
```
nginx version: nginx/1.10.1
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/local/Cellar/nginx/1.10.1/)
  -c filename   : set configuration file (default: /usr/local/etc/nginx/nginx.conf)
  -g directives : set global directives out of configuration file
```
打开配置文件：
```
vi /usr/local/etc/nginx/nginx.conf
```
在文件末尾可以看到：
```
include servers/*;
```
它将同目录下的servers目录里的文件都包含了进来，由此，我们可以在servers文件里创建开发项目的配置信息：
```
cd servers/
vi test.conf
```
将以下配置信息，写入 test.conf文件中：
```
server {
    listen       8099;
    server_name  localhost;
    root /home/www/php_project;
    rewrite . /index.php;
    location / {
        index index.php index.html index.htm;
        autoindex on;
    }

    #proxy the php scripts to php-fpm
    location ~ \.php$ {
        include /usr/local/etc/nginx/fastcgi.conf;
        fastcgi_intercept_errors on;
        fastcgi_pass   127.0.0.1:9000;
    }
}
```

在上述的`/home/www/php_project`的目录下，我们创建一个 index.php文件：
```
cd /home/www/php_project
vi test.php
```
写入内容：
```
<? php
phpinfo();
```
重启 nginx:
```
sudo nginx -s stop
sudo nginx
```

打开浏览器，访问`localhost:8099`。可以看到关于 PHP 配置的信息。

至此，MNMP(MacOS-nginx-MySQL-PHP)环境已经搭建完成。

### 安装 PHP 扩展

环境搭建完成后，你可能还需要安装一些 PHP 扩展，如 MemCache、Redis、Mongo、Solr 等。

在安装 PHP 扩展之前，**你需要完成一些必要的操作**。

#### 关闭 SIP

这是安装 PHP 扩展前的必要操作。如果跳过这一操作，即使你用 sudo 命令安装扩展，依旧会报 Operation not permitted 的错误。这是因为 OSX 10.11 El Capitan（或更高）新添加了一个新的安全机制叫系统完整性保护 System Integrity Protection (SIP)，所以对于以下目录：
* /System 
* /sbin 
* /usr 不包含(/usr/local/) 

仅仅供系统使用，其它用户或者程序无法直接使用，而我们的 /usr/lib/php/extensions/ 则刚好在受保护范围内（误伤世界上最好的语言）。

所以解决方法就是禁掉 SIP 保护机制，步骤是：

1. 重启系统
2. 按住 Command + R（重新亮屏之后就开始按，象征地按几秒再松开，直到出现苹果标志性的 Logo）
3. 菜单“实用工具” ==>> “终端” ==>> 输入：`csrutil disable`。执行后会输出：`Successfully disabled System Integrity Protection. Please restart the machine for the changes to take effect`
4. 重启系统

当然，PHP 扩展安装完成后，就可以重新打开 SIP，方法同上，命令改为：`csrutil enable`。

#### 安装一些必要的依赖包

安装 autoconf，PHP动态编译 phpize 时需要：

```
brew install autoconf
```

安装 openssl，安装某些 php 扩展如 mongo 时需要。

```
brew install openssl
```

mongo 扩展安装是可能会报 openssl 错误，解决方法如下：

```
ln -s /usr/local/Cellar/openssl/1.0.2j/include/openssl /usr/include/openssl
```


#### 正式安装扩展

这里有两种方法安装 php 扩展：

* 通过 pecl 管理工具安装
* 通过源码包安装

##### 通过 pecl 管理工具安装

首先安装 pecl：

```
cd /usr/lib/php
sudo php install-pear-nozlib.phar
```

pecl 一般就会安装成功，如果失败，换另一种方式安装 pecl：

```
curl -O http://pear.php.net/go-pear.phar
sudo php -d detect_unicode=0 go-pear.phar
```
1. 输入 `1`，回车，输入`/usr/local/pear`
2. 输入 `4`，回车，输入`/usr/local/bin`
3. 回车

安装好 pecl 之后，我们就可以愉快地安装 PHP 扩展了：
```
sudo pecl install solr
sudo pecl install memcache
sudo pecl install mongo
```

##### 通过源码包安装

除了通过 pecl 安装，我们还可以通过下载源码包来进行安装扩展：
```
wget http://pecl.php.net/get/redis-2.2.8.tgz
tar -zxvf redis-2.2.8.tgz
cd redis-2.2.8
phpize
./configure
make
sudo make install
```

扩展安装完成后，我们还需最后一步，修改`php.ini`文件，并重启 PHP-FPM：
```
cd /private/etc/
cp php.ini.default php.ini
vi php.ini
```
追加扩展信息：
```
extension=memcache.so
extension=mongo.so
extension=redis.so
extension=solr.so
```
重启 PHP-FPM:
```
ps aux|grep php-fpm
sudo kill php-fpm min pid # 杀死 php-fpm 最小的进程id
sudo php-fpm
```

打开浏览器，访问`localhost:8099`。查看扩展是否安装成功。

### 参考

* http://blog.csdn.net/suxianbaozi/article/details/40617885
* http://blog.csdn.net/pang040328/article/details/41259385
* http://blog.csdn.net/qq285744011/article/details/52810066
* http://www.xitongzhijia.net/xtjc/20150526/49276.html