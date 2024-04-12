### incp 后台搭建文档

#### 项目环境(可根据项目配置自由调整)
- PHP: ^8.0.2
- Composer: ^2.5
- Redis: ^5.0.14.1
- MySQL: ^8
- Laravel: ^9
- Node: ^18
- Nginx: *

#### 环境搭建步骤

**Windows**
1. 运行 `composer install` 安装 PHP 依赖。
2. 使用 `npm install package-name --save` 安装 Node 生产依赖。
3. 使用 `npm install package-name --save-dev` 安装 Node 开发依赖。
   
**Linux**
1. 修改文件权限：`chmod -R 777 storage` 和 `chmod -R 777 bootstrap/cache`。
2. 执行 `composer install` 安装 PHP 依赖。
3. 使用 `npm install package-name --save` 安装 Node 生产依赖。
4. 使用 `npm install package-name --save-dev` 安装 Node 开发依赖。

因为项目版本过多如出现版本冲突提示
--ignore-platform-reqs （忽略版本安装）

**项目环境搭建过程种如果出现版本或者扩展 报错不匹配请下载本地环境对应的版本
文件在根目录下**
1. package.json 项目依赖
2. composer.json 项目扩展

### 请先运行查看当前项目的所有指令
php artisan list

#### 项目启动--windwos
1. 确保 Nginx 配置正确，将后台页面目录指向 `public/admin`。
2. 使用 `npm run dev` 启动 Vite 开发服务器。
3. 使用 `php artisan serve` 启动 API 服务。
4. 使用 `php artisan websockets:serve` 启动 WebSocket 服务。
5. 每次修改代码后，执行 `php artisan queue:restart` 重启队列。
6. 启动脚本任务：`php artisan schedule:run`。

#### 项目启动--liunx（开启PHP-FPM指令）
1. 确保 Nginx 配置正确，将后台页面目录指向 `public/admin`。
2. 确保 Nginx 配置正确，api 指向 `public/api`。
##### 请参考框架脚本目录 console/commands/tool
3. php artisan tool:crontab 管理所有任务 （stop:停止,start:启动, restart:重启）
4. `不全需修改` php artisan tool:supervisor 管理所有进程 （stop:停止,start:启动, restart:重启）**可以按照自己的方式来进行进程管理**


#### 其他
- Laravel 文档: [https://laravel.com/docs/10.x](https://laravel.com/docs/10.x)


## 项目 Nginx代理详解 
#### 所有版本后台页面代理
```bash
server {
    listen 80;
    server_name ~^www.admin.(?<version>[^.]+)\.test\.com$;
    root /data/$version/public/admin;
    index index.html index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$query_string;
    }

    location ~* \.(svg|ttf|otf|eot|woff|woff2)$ {
        add_header Access-Control-Allow-Origin *;
    }

    location ~* \.(wgt|zip)$ {
        add_header Access-Control-Allow-Origin *;
        add_header Content-Disposition 'attachment; "filename=$args"';
    }

    location ~ \.php$ {
        fastcgi_keep_conn on;
        try_files $uri =404;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### 所有版本前端编译文件目录-（uni-app 编译文件）
```bash
server {
    listen 80;
    server_name ~^(?<subdomain>[^.]+)\.wwww\.(?<version>[^.]+)\.test\.com$;
    root /data/$version/public/web/$subdomain;
    index index.html index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$query_string;
    }

    location ~* \.(svg|ttf|otf|eot|woff|woff2)$ {
        add_header Access-Control-Allow-Origin *;
    }

    location ~* \.(wgt|zip)$ {
        add_header Access-Control-Allow-Origin *;
        add_header Content-Disposition 'attachment; "filename=$args"';
    }

    location ~ \.php$ {
        fastcgi_keep_conn on;
        try_files $uri =404;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### 代理H5各个版本的api,ws
```bash
server {
    listen 80;
    server_name ~^(?<version>[^.]+)\.test\.com$;
    root /data/$version/public/api;
    index index.html index.php;

    set $ws_port '6001';
    if ($version = "v2") {
        set $ws_port '6002';
    }

    if ($version = "v3") {
        set $ws_port '6003';
    }

    if ($version = "v4") {
        set $ws_port '6004';
    }

    if ($version = "v5") {
        set $ws_port '6005';
    }

    location ~ ^/ws(.*)$ {
        proxy_pass http://127.0.0.1:$ws_port$1;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.php$is_args$query_string;
    }

    location ~* \.(svg|ttf|otf|eot|woff|woff2)$ {
        add_header Access-Control-Allow-Origin *;
    }

    location ~* \.(wgt|zip)$ {
        add_header Access-Control-Allow-Origin *;
        add_header Content-Disposition 'attachment; "filename=$args"';
    }

    location ~ \.php$ {
        fastcgi_keep_conn on;
        try_files $uri =404;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```