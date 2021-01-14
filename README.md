# WP Proxy Sites - Dockerize Multiple WordPress Sites

<img width="1434" alt="wp-proxy-scope" src="https://user-images.githubusercontent.com/271049/61531755-5ddc2d80-aa5a-11e9-9383-ac099f4eb3a6.png">

### 特點
* 啟用多個 WordPress 和 非 WordPress 網站容器，再讓它們共用一個 DB 容器。另外還可搭配 Redis、MailCatcher、WP-Cli 等容器服務。

* 請注意，本 Repo 需要先使用：WP Proxy Companion (https://github.com/mrmu/wp-proxy-companion) 建立 Nginx Proxy 之後，才能正常使用。

### 使用容器
* MariaDB
* WordPress/Apache/PHP 7.x
* Redis Cache Server：需搭配 Redis Object Cache 外掛使用
* MailCatcher：獨立運作的容器，建議於本地端開發時使用。
* WP-Cli

### 安裝

無論本機或正式環境：
1. 確認已安裝並運行 [WP Proxy Companion]((https://github.com/mrmu/wp-proxy-companion))
2. 找個目錄存放本設定和網站相關檔案，比如正式環境可以放 /var/docker-www/ (名稱隨你取) 或本機放 /Users/xxxx/，建議可以和 WP Proxy Companion 放在同目錄。進入該目錄後 git clone 本 repo。(看你有沒有其他慣放 Docker 設定的目錄也行)
    ```
    cd /var/docker-www
    git clone https://github.com/mrmu/wp-proxy-sites.git
    ```
3. 到這裡，你選定的目錄下會有 wp-proxy-companion 和 wp-proxy-sites 這兩個目錄，未來你的網站檔案都會在 wp-proxy-sites/sites 底下。
4. 將 /wp-proxy-sites/sample.env 另存成 .env。
    ```
    sudo cp sample.env .env
    ```
5. 修改 .env 裡面的 mySQL 資料庫設定，包含指定 <strong>Root 密碼及資料庫名稱</strong>，之後會成為 <strong> DB 容器的設定值，請一定要修改密碼 </strong>。後續的 WP 容器會使用到.env 裡設定的 Root 密碼並且另建各自的資料庫，這裡設定的資料庫不會使用到。


### 開發流程說明

無論本機或正式環境：
1. 如果你是第一次使用，建議把目標放在「先完成一個可運作的簡單 WP 網站」，之後就會覺得設定其實不複雜。
2. 確認已先建立 wp-proxy network 並已啟用 wp proxy companion  (參考說明)
3. 雖然現在 docker 還沒起來，不過要先把網址指到目前這台主機，若是...
    * 正式環境：請建立 DNS A 指向主機 IP。
    * 本機開發：把你要在本機建立的網域建立在 hosts 檔案上，以 mac 為例是編輯 /etc/hosts，以本repo 的docker-compose.sample.yml 為例，若要建立3個網站容器，本機的網域設定如下：
        ```
        127.0.0.1       wp1.test
        127.0.0.1       wp2.test
        127.0.0.1       phpweb.test
        ```
4. 開始設定 WP 網站容器，將 docker-compose.sample.yml 另存為 docker-compose.yml：
	```
	sudo cp docker-compose.sample.yml docker-compose.yml
	```
	再打開 docker-compose.yml 修改。範例中對應的容器是 wp1，可以修改的部份：
	1. 把 wp1 和 container_name 改為你要的網站代稱。 
	2. image: 範例要安裝的是 [WordPress 官方的 docker image](https://hub.docker.com/_/wordpress/) (除了 wp 這個容器也自帶了 php 和 apache)，你可以修改成你要的 image 版本。
	3. volumes: 把容器裡主要的目錄對應到指定的目錄，裡面的 wp1 也可修改成你要的代稱。
	4. environment:
    	1. WORDPRESS_DB_NAME: wp容器要使用的資料庫名稱。
    	2. VIRTUAL_HOST：改為你要使用的網域。(必填，有個 docker-gen 容器會以此來產生對應的 nginx 設定檔)
    	3. LETSENCRYPT_HOST: 設定此項會自動申請套用 SSL 憑證，改為你要使用的網域。(<strong>注意，第一次跑請先注解掉，要先讓 http 版本網站上線，不然 Let's Encrypt 的 ACME challenge 無法得知你的 http 網站存在，就發不出憑證。</strong>)
    	4. LETSENCRYPT_EMAIL: 改為你要用於申請 SSL 憑證的E-mail。
	5. redis: 底下的 volume: 要改為你的 WP 容器對應到的目錄。
	6. 把你不需要的容器設定註解掉或刪掉，這樣就完成 docker-compose.yml 的設定，下一步我們要開始啟動容器了。

5. 啟用本repo的 docker-compose.yml，在 wp-proxy-sites/ 下執行：
    ```
    docker-compose up -d --build
    ```
    若是第一次使用，建議你docker-compose up 不要加-d，可以觀察一下執行過程中有沒有發生問題，只要 Ctrl+c 就能離開，再下 docker-compose down 關掉所有 container，就可以重新下 up 指令。(註：docker-compose down 和 up 都要在 .yml 同目錄下執行才能針對該設定生效)

6. 第一次啟用時間會比較長，因為要下載各個 docker image 並啟用。

7. 跑完若一切正常，執行 docker ps -a 就會看到所有運行起來的容器了。而此時在瀏覽器輸入網址，就能看到 wp 的安裝畫面了。

8. 之後要再增加其他容器 (比如網站容器)，只要再編修 docker-compose.yml 存檔後，再啟用該容器即可，要怎麼增加其他容器的設定可以參考下面的說明。

### docker-compose.sample.yml 容器設定說明

* db: 定義一個資料庫容器(mariadb)，這會讓後面 3 個網站容器共用。db 容器是一定要存在的，其他的容器則是依使用狀況增減修改。

* wp1: 定義一個官方的 WordPress 容器 (本身自帶 apache 和 php)，是故意寫的比較像正式環境的設定，所以會使用到 db 和 redis 這兩個容器的服務。它定義了：
    VIRTUAL_HOST: wp1.test，表示會讓佔用 80 /443 port的 wp proxy companion 以 Nginx 的反向代理設定導至此 wp1 容器的 apache。
    LETSENCRYPT_HOST 及 LETSENCRYPT_EMAIL: 這會讓 wp proxy companion 幫忙申請 Let's encrypt 憑證並以 docker-gen 產生 nginx 反向代理的設定，再自動套用，但網域必須是真正指向主機IP的，所以本機測試不會成功。(請先讓 http 版本網站上線，再設定這兩項，才能正確通過 challenge 取得憑證，請參考 wp proxy companion 的說明)

* redis: 做為 redis server 的容器。wp1 要使用 redis，還須要額外安裝 Redis Object Cache 這個 WP 外掛，要設定連結的 redis host，就寫 redis 的 container name 即可 (本repo裡的設定就叫 redis)。

* wp2: 一樣是定義一個官方的 WordPress 容器，這邊故意寫的比較像本機的設定，會使用到 db 和 mailcatcher。這個容器就只定義了 VIRTUAL_HOST。

* cli-for-wp2: 定義一個 wp-cli 容器指定給 wp2 使用。使用方式如下：
下指令進入cli的bash：
    ```
    docker-compose run --rm cli-for-wp2 bash
    ```
這樣就會進入wp目錄，並且可以下wp-cli 指令。

* phpweb: 這裡定義了第三個網站容器，但它不是 wp，就只是一般的 php+apache。它的 volumn 是對應到本機的 ./app/sites/phpweb，所以如果這裡沒有東西，瀏覽 phpweb.test 會出現 Forbidden。

* mailcatcher: 主要用於開發環境，這個容器會幫忙攔下網站發出的信件，只要用瀏覽器開 1080 port 就可以看見web mail 的介面，裡頭就會有信件，對於測試信件的情境 (不讓它真的被寄出) 很方便。

* 最後定義了一個外部的 docker network 叫 wp-proxy，也就是與 wp proxy companion 連線的設定。

### dockerfile 設定範例 (dockerfile-sample/)
這裡有些設定範例，可以參考寫法。

* dockerfile-sample/php7.3：如果要安裝純 PHP 環境，但想自行安裝 Ubuntu 套件或啟用 Apache module，可參考這個目錄下的 dockerfile，它額外安裝了 mysqli、pdo、pdo_mysql 並且啟用 apache rewrite module，剛好可以作為 Laravel 開發使用。

* dockerfile-sample/wp5.3-xsl：如果要安裝 WP 環境，但想自行安裝 Ubuntu 套件，可參考這個目錄下的 dockerfile，它額外安裝了 xsl 套件，比如要安裝 Google XML sitemap 的外掛，就需要 ubuntu 有 xsl 套件。

完成自訂的 Dockerfile 後，可於 docker-compose.yml 裡使用「build:」來取代原來的「image:」設定。如使用「build: ./dockerfile-sample/php7.3」取代掉 「image: 'php:7.3-apache'」。(但請另建目錄，不要直接使用 dockerfile-sample/)

### 其他說明

* 關於改變 php.ini 的設定部份有兩種方式：
    1. 增加 uploads.ini：wp 容器裡 volume 有個 conf.d/uploads.ini 的設定 (已註解掉)，如果沒有預先建立好 uploads.ini，docker 會建立一個空目錄叫 uploads.ini，可以把這個空目錄刪除，再建立一個真的 uploads.ini 來改變上傳檔案的限制，參考內容如下：
        ```
        file_uploads = On
        memory_limit = 256M
        upload_max_filesize = 64M
        post_max_size = 64M
        max_execution_time = 600
        ```
    修改好要 docker-compose down 再 docker-compose up -d --build 讓它套用設定。
    2. 修改 .htaccess，加入：
        ````
        php_value post_max_size 64M
        php_value upload_max_filesize 64M
        php_value max_execution_time 600
        php_value memory_limit 256M
        ````

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

* 如果要加入 www to non-www 的設定，可以在 VIRTUAL_HOST 加入 www 的設定，如：
```
VIRTUAL_HOST: audilu.com, www.audilu.com
```
如此 nginx-proxy 就會多生出一組設定。

* 要變更現有WP容器的PHP版本，可以修改 Image 來源，比如 wordpress:5.2.2-php7.3 改為 wordpress:5.0.1-php5.6，再重啟容器即可生效，WP 目錄下的檔案都不會更動。也因為 WP 目錄下的檔案都不會更動到，所以 wp 版本不會因此降到 5.0.1，除非是第一次運行，才會下載 WP 5.0.1 的檔案。

* WP 底下目錄和檔案的權限設定，以目錄 755 及檔案 644 為建議 ([官方建議在此](https://wordpress.org/support/article/changing-file-permissions/))，指令如下：
```
sudo find ./目標目錄 -type d -print0 | xargs -0 sudo chmod 0755
sudo find ./目標目錄 -type f -print0 | xargs -0 sudo chmod 0644
```
或
```
sudo find ./目標目錄 -type d -exec chmod 0755 {} \;
sudo find ./目標目錄 -type f -exec chmod 0644 {} \;
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
* 重啟容器
```
docker restart [容器ID]
```
* 停止容器
```
docker stop [容器ID]
```
* 移除容器
```
docker rm [容器ID]
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
* 查看 db 容器的密碼及相關資訊
```
docker exec db容器ID env
```
* 執行 db 容器內的 mysql 指令
```
docker exec -it db容器ID mysql -uroot -p
```
* 查看容器裡運行的 processes (也就是在容器裡下 top 指令)
```
docker container top 容器ID
```
* 只查看 error logs:
```
docker logs 容器ID -f 1>/dev/null
```
* 只查看 access logs:
```
docker logs 容器ID -f 2>/dev/null
```

### 備份

#### 資料庫備份
執行 db 容器內的 mysqldump 將資料倒到 .sql (配合utf8mb4語系)
```
docker exec -it db /usr/bin/mysqldump --default-character-set=utf8mb4 --hex-blob -u root -p{root_password} {database_name} > backup.sql
```

#### 資料庫還原
將 .sql 還原至 db 容器內的 mysql
```
docker exec -i db /usr/bin/mysql -u root -p{root_password} {database_name} < backup.sql
```

### 其他資源
* #.htaccess ref: https://gist.github.com/HechtMediaArts/c96bc796764baaa64d43b70731013f8a

### 搬家示例 Migration

1. 打包好 wp-content (如 wp-content.tar.gz) 移到新機備用。
1. 打包好資料庫 (如 db.sql.gz)，打包前注意：
    1. 要先把網址取代成 http://，因為我們要先把 http 版本跑起來，才能加裝憑證。
    1. 如果資料庫裡有網站根目錄的設定，要記得改為 /var/www/html。
    1. 注意 WordPress 資料表前縐字 (Prefix) 新舊設定要一致，才不會出現安裝 WP 的畫面。
1. 編輯 /wp-proxy-sites/docker-compose.yml，建立網站容器的設定，先不加或註解掉 LETSENCRYPT_HOST 和 LETSENCRYPT_EMAIL 設定。
1. 把網站容器跑起來：
    ```
	docker-compose up -d --no-deps --build 網站容器名稱
    ```
	到這裡，新的網站目錄 (/wp-proxy-sites/sites/新站/) 和全新的WP檔案會被建立，對應的資料庫也會被建立 (但還是空的，因為還沒跑安裝)。好奇的話也可以看一下 nginx proxy 設定 (wp-proxy-companion/nginx/conf.d/default) 也都準備好指到網站容器裡的 Apache 了，
1. 把之前打包好的 wp-content 備份都解壓放到新的網站目錄 (/wp-proxy-sites/sites/新站/)，並且把資料庫匯入到新主機的 db 容器。
1. 要準備把網址指向新的IP，若要搬的是 HTTPS 的網站，怕在 「HTTP 版生效 -> 取得憑證 -> HTTPS 版生效」 的轉換過程中出現大破圖，可以先 rename index.php 成 _index.php，再建立 index.html 並把 index.html 佈置成維護公告頁。
1. 開始把網址指到新的IP，生效後瀏覽器應該看得到正常運行的新網站 (http)。(注意，若還是用 https 的網址來觀看 HTTPS 版本的 WP 網站，這時會發生 500，因為現在容器裡還沒有對應的 nginx 設定及 SSL 憑證，建議一直要確實先讓 HTTP 版生效)
1. 開始裝SSL憑證，打開 /wp-proxy-sites/docker-compose.yml，幫新的網站容器加上 LETSENCRYPT_HOST 和 LETSENCRYPT_EMAIL 設定。
1. 重跑網站容器：
    ```
	docker-compose up -d --no-deps --build 網站容器名稱
    ```
	到這裡，等個5~30秒 (也可以查看一下 wp-proxy-letsencrypt 容器的 log，看它有沒有在運作，若等不及可以 restart 它，讓它重跑檢查)，檢查 /wp-proxy-companion/nginx/certs 目錄下有沒有對應網域的憑證檔，若有的話，就表示憑證已安裝成功了。
1. 瀏覽器重整一下，應該看得到正常運行的新網站 (https)，資料庫裡的連結應該都還是 http，可以取代成 https。

### 常見問題
1. 400 Bad Request：檢查網站容器的環境變數 VIRTUAL_HOST (網址)，避免使用特殊底線，如 "_"。
1. 如果出現 502 bad gateway，請先檢查 ./wp-proxy-companion/nginx/conf.d/default 裡有沒有對應到 wp-proxy-sites/docker-compose.yml 裡定義的網站容器網址，若沒有就要檢查 yml 裡該網站容器的 volumn 是否有正確設定對應到 /var/www/html/。若還是 502，要檢查 ./wp-proxy-sites/docker-compose.yml 的 docker network 設定與 companion 是否一致，可用 docker network ls 看有沒有異常。
1. 503 Service Temporarily Unavailable：通常是網址請求對應不到 Server 裡的網址設定才會發生。請檢查 DNS 設定的網址與 yml 裡的 VIRTUAL_HOST 設定是否完全一致。
1. 如果有用到「主網域」和其「子網域」的設定，在 yaml 檔裡請先放置主網域的設定，否則 docker-gen 可能會在設定完子網域的 conf 後就略過建立或更新主網域的 conf，造成 503 的問題。
1. 如果懷疑設定有被 cache 住，可以刪掉 docker image 讓它重抓重建，但記得要先停掉 container：
    ```
    	docker images
	docker rmi {image名字}
    ```
1. wp-proxy (nginx) 容器的 logs 才能看到 real ip；而網站容器的 logs 只能看到內部 ip。
1. 若遇到這種警告訊息，應該是在錯的目錄執行 docker-compose up 了，記得要在 docker-compose.yml 和 .env 設定檔都在的 wp-proxy-sites 目錄下執行： 
    ```
    WARNING: The MYSQL_ROOT_PASSWORD variable is not set. Defaulting to a blank string.
    WARNING: The MYSQL_DATABASE variable is not set. Defaulting to a blank string.
    ```
