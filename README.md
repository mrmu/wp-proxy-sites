# WP Proxy Sites - Dockerize Multiple WordPress Sites

### 特點
* 啟用多個 WordPress 和 非 WordPress 網站容器，再讓它們共用一個 DB 容器。另外還可搭配 Redis、MailCatcher、WP-Cli 等容器服務。

* 請注意，本 Repo 需要先使用：WP Proxy Companion (https://github.com/mrmu/wp-proxy-companion) 建立 Nginx Proxy 之後，才能正常使用。

### 使用容器
* MariaDB
* WordPress/Apache/PHP 7.3
* Redis Cache Server：需搭配 Redis Object Cache 外掛使用
* MailCatcher：獨立運作的容器，建議於本地端開發時使用。
* WP-Cli

### 安裝
1. git clone 此 repo。
2. 將 sample.env 另存成 .env。
3. 修改 .env 裡面的db 設定。

### 開發流程說明

1. 確認已先建立 wp-proxy network 並啟用 wp proxy companion  (參考說明)
2. 建立網域指向

##### 正式環境
把你在 docker-compose.yml 裡設定的 VIRTUAL_HOST 網域 DNS 指向正式主機的IP。

##### 本機開發
把你要在本機建立的網域建立在 hosts 檔案上，以 mac 為例是編輯 /etc/hosts，以本repo 的docker-compose.yml 為例，會建立3個網站容器，網域設定如下：
```
127.0.0.1       wp1.test
127.0.0.1       wp2.test
127.0.0.1       phpweb.test
```
3. 啟用本repo的 docker-compose.yml
```
docker-compose up -d --build
```
若是第一次使用，建議你docker-compose up 不要加-d，可以觀察一下執行過程中有沒有發生問題，只要 Ctrl+c 就能離開，再下 docker-compose down 關掉所有 container，就可以重新下 up 指令。

4. 此時若一切正常，在瀏覽器輸入網址，就能看到 wp 的安裝畫面了。

### docker-compose.yml 容器設定說明

* db: 定義一個資料庫容器(mariadb)，這會讓後面 3 個網站容器共用。db 容器是一定要存在的，其他的容器則是依使用狀況增減修改。

* wp1: 定義一個官方的 WordPress 容器 (本身自帶 apache 和 php)，是故意寫的比較像正式環境的設定，所以會使用到 db 和 redis 這兩個容器的服務。它定義了 VIRTUAL_HOST: wp1.test，表示會讓佔用 80 /443 port的 wp proxy companion 以 Nginx 的反向代理設定導至此 wp1 容器的 apache。另外還設定了 LETSENCRYPT_HOST: wp1.test 及 LETSENCRYPT_EMAIL: my@email.adds，這會讓 wp proxy companion 幫忙申請 Let's encrypt 憑證並以 docker-gen 產生 nginx 反向代理的設定，再自動套用，但網域必須是真正指向主機IP的，所以本機測試不會成功。

* redis: 做為 redis server 的容器。wp1 要使用 redis，還須要額外安裝 Redis Object Cache 這個外掛，要設定連結的 redis host，就寫 redis 的 container name 即可 (本repo裡的設定就叫 redis)。

* wp2: 一樣是定義一個官方的 Wordpress 容器，這邊故意寫的比較像本機的設定，會使用到 db 和 mailcatcher。這個容器就只定義了 VIRTUAL_HOST。

* cli-for-wp2: 定義一個 wp-cli 容器指定給 wp2 使用。使用方式如下：
下指令進入cli的bash：
```
docker-compose run --rm cli-for-wp2 bash
```
這樣就會進入wp目錄，並且可以下wp-cli 指令。

* phpweb: 這裡定義了第三個網站容器，但它不是 wp，就只是一般的 php+apache。它的 volumn 是對應到本機的 ./app/sites/phpweb，所以如果這裡沒有東西，瀏覽 phpweb.test 會出現 Forbidden。

* mailcatcher: 這個容器會幫忙攔下網站發出的信件，只要用瀏覽器開 1080 port 就可以看見web mail 的介面，裡頭就會有信件，對於測試信件的情境 (不讓它真的被寄出) 很方便。

* 最後定義了一個外部的 docker network 叫 wp-proxy，也就是與 wp proxy companion 連線的設定。

### 其他說明

* 關於wp 容器裡 volume 有個 conf.d/uploads.ini 的設定，如果沒有預先建立好 uploads.ini，docker 會建立一個空目錄叫 uploads.ini，可以把這個空目錄刪除，再建立一個真的 uploads.ini 來改變上傳檔案的限制，參考內容如下：
```
file_uploads = On
memory_limit = 64M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
```
修改好要 docker-compose down 再 docker-compose up -d --build 讓它套用設定。

* 要分開管理開發環境及正式環境的 docker-compose.yml 設定，可依據用途建立多個 yml 檔，不會更動到的設定就放在  docker-compose.yml，假設我習慣在本機的容器設定不同，就把相關設定移到 docker-compose.local.yml；正式環境的設定就移動到 docker-compose.prod.yml。

若是在本機環境要執行 docker-compose up，就可以這樣下指令：
```
docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d
```
docker down 也是比照辦理：
```
docker-compose -f docker-compose.yml -f docker-compose.local.yml down
```

* 如果更新或新增了 docker-compose.yml (或 prod.yml, local.yml 都一樣) 裡的容器服務設定，但不要整個全部 down 再 up (Downtime 時間長)，可下指令 (以本機為例)：
```
docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d --no-deps --build 容器名稱
```

### 其他工具

* PHP Composer：可以在容器外使用，也可以啟用一個 composer 容器，就像 docker-compose.yml 範例裡註解的一樣，要執行它可取消註解，並且下指令 (這樣做沒有比較方便，所以用習慣的方式就好XD)：
```
docker-compose run wp2-composer update
```

* Git：因為網站目錄已透過 volumn 設定 mapping 到容器外部了，所以只要在外部使用 Git 即可。

* Node.js：會用到 Node.js 通常是前端需要套件管理工具，如：Webpack, Gulp ...等，因為佈景目錄已透過 volumn 設定 mapping 到容器外部，所以在外部執行即可。

### 常用指令

* 查看目前所有的容器
```
docker ps -a
```
* 停止容器
```
docker stop [容器名]
```
* 移除容器
```
docker rm [容器名]
```
* 停止 docker-compose.yml 裡所有容器 (但沒有移除)
```
docker-compose stop
```
* 停止 docker-compose.yml 裡所有容器並且移除
```
docker-compose down
```
* 強制重新建立 docker-compose.yml 裡的容器
```
docker-compose up -d --force-recreate
```
* 查看所有 docker network
```
docker network ls
```

### 備份

#### Local Database Backup
Here's how to dump your local database with Docker into a .sql file
```
docker exec -it db /usr/bin/mysqldump -u username -ppassword database_name > backup.sql
```

#### Local Database Restore
Restore a previous database backup
```
docker exec -i db /usr/bin/mysql -u username -ppassword database_name < backup.sql
```