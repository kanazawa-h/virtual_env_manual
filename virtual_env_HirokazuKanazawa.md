
# 構成手順書

## バージョン一覧
***

| 　　  |  バージョン |
| ---- | ---- |
|  PHP |  7.3  |
|  laravel  |  6.0 |
| MySQL | 5.7 |
| Nginx |  1.19.8  |
|Vagrant|  2.2.14   |
|OS     |   Centos7  |
***  
## 手順  
### Vagrant起動
以下のコードをターミナルで実行し  
```$ mkdir vagrant_Laravel ```  
ディレクトリを作成します  

そのディレクトリ内で  
```$ vagrant init centos7 ```  
を実行しOSをcentos7に指定した設定ファイルを作成  
上記の処理で作祭されたvagrantfileを編集します。  
変更箇所は  
**config.vm.network "forwarded_port", guest: 80, host: 8080**  
をコメントインし、  
**config.vm.network "private_network", ip: "192.168.33.10"**  
のIPを192.168.33.19に変更しコメントイン  
**config.vm.synced_folder "../data", "/vagrant_data"**  
上記を  
**config.vm.synced_folder "./", "/vagrant", type:"virtualbox"**  
にしホストとゲストのvagrantのディレクトリをリアルタイムで同期させます。  
 
```$ vagrant up```  
で仮想マシンを起動し、  
```$ vagrant ssh```  
で仮想マシンに接続いたします。  
***  
 
### PHPのインストール
```$ sudo yum -y groupinstall "development tools"```  
でGitなど必要なパッケージをインストールします。  
```$ sudo yum -y install epel-release wget```  
を実行しEPELのリポジトリをインストール  
```$ sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm```  
上記のURLを指定し  
```$ sudo rpm -Uvh remi-release-7.rpm```  
でパッケージをアップデートし  
```$ sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd``` ```php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip```  
でPHP7.3とモジュールをインストールします。  
```$ php -v```  
でバージョン確認とインストールが成功しているかを確認します  
 
***
### Nginxのインストール  
WebサーバーのNginxを使用するための準備としてファイアーウォールの設定をします。  
```$ sudo systemctl start firewalld.service```  
```$ sudo firewall-cmd --add-service=http --zone=public --permanent```   
上記を実行しファイアーウォールを起動しhttpの通信を許可します。  
```$ sudo firewall-cmd --reload```  
で設定を更新します。
WebサーバーのNginxを使用するため  
```$ sudo vi /etc/yum.repos.d/nginx.repo```  
で/etc/yum.repos.d/nginx.repoのリポジトリファイルに 
```Nginx
[nginx]  
name=nginx repo  
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/  
gpgcheck=0  
enabled=1  
```
を書き込み保存し (http://192.168.33.19) にアクセスしwelcomeが表示されるか確認します。  
 
***
### MySQLインストール  
Laravelを動かすためにMySQL5.7を準備します。  
```$ sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm```  
でURLを指定しファイルをダウンロードし、  
```$ sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm```  
でファイルをアップデートします。そして、  
```$ sudo yum install -y mysql-community-server```  
でMySQLをインストールし、  
```$ mysql --version```  
でバージョンが確認できれば完了。  
```$ sudo cat /var/log/mysqld.log | grep 'temporary password'```  
でmysqld.log内からtemporary passwordの文字列が一致するファイルを見つけて出力します。  
返り値の**root@localhost: XY**　の**XY**の箇所にランダムの文字列がMySQLのパスワードになるのでメモします。  
```$ sudo systemctl start mysqld```  
でMySQLのシステムを実行し、  
```$ mysql -u root -p```  
```$ Enter password:```  
で先程のパスワードを入力し起動させます。
mysql > set password = "新たなpassword";  
で大文字小文字の英数字と記号を合わせたパスワードを再設定し、  
```mysql >　create database laravel_app;```  
でlaravel_appのデータベースを作成します。  
```mysql > exit```  
でMySQLを終了  
 
***
### Larvelインストール  
vagrantディレクトリ内で  
```composer create-project laravel/laravel --prefer-dist laravel_app 6```  
を実行しlaravel6をインストールします。
vagrantディレクトリ内でPWDを入力しインストールされたか確認します。  
VSコードでvagrant_Laravelディレクトリ内にあるlaravel_appを開き.envを開き
**DB＿PASSWORDにMySQLのパスワードを入力**  
**DB_DATABASEがlaravel-appになっているか**  
以上を入力確認します。
ターミナルで**laravel_appディレクトリに移動**し  
```php artisan migrate```  
を実行し変更を反映します  
 
***
### Laravel動かすための準備
laravelを動かすために  
```$ sudo vi /etc/nginx/conf.d/default.conf```  
でNginxのファイルを編集します  
```Nginx
server {
  listen       80;
  server_name  192.168.33.19; #IPを変更
  root /vagrant/laravel_app/public; #追記
  index  index.html index.htm index.php; #追記


  location / {
      #root   /usr/share/nginx/html; #コメントアウト
      #index  index.html index.htm;  #コメントアウト
      try_files $uri $uri/ /index.php$is_args$args;  #追記
  }

　#{}がコメントインされているか確認
  location ~ \.php$ {
  #    root           html;
      fastcgi_pass   127.0.0.1:9000;　#コメントアウト
      fastcgi_index  index.php;　#コメントアウト
      fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name;　#に変更
      include        fastcgi_params;　#コメントアウト
  }

```  
 
保存し終了します  
 
```$ sudo vi /etc/php-fpm.d/www.conf```  
上記を実行し  
**user = apache**  
**group = apache**  
の箇所のapacheを**nginx**に変更し保存します。  
 
```$ sudo systemctl start nginx```
```$ sudo systemctl start php-fpm```  
を実行しNginxとPHPのアプリケーションサーバーを起動します。  
(http://192.168.33.19) にアクセスしlaravelが表示されるか確認します。  
***  
 
### ログイン機能  
 
laravel_appディレクトリに移動し  
```composer require laravel/ui "^1.0" --dev```  
```php artisan ui vue --auth```  
を実行しログイン機能を導入します。  
laravelの画面の左上のregisterかloginにアクセスすると、操作権限が無いためエラーが発生するので、laravel_appディレクトリ内で、  
```sudo chmod -R 777 storage```  
のコマンドでパーミッションを実行し、storageディレクトリ以下の操作権限を与えます。  
そうすると、registerにアクセスができるのでアクセスしアカウントを作成し作成ボタンを押します。  
再度エラーが表示され、laravel_appディレクトリ下にあるsessionにアクセスを拒否されるので、  
```$ sodo chmod -R 777 storage/framework/sessions```  
で操作権限を与えて、再度laravelの画面をリロードするとログインが成功します。  