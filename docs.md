# Cấu hình Server cho Site WordPress có lượng truy cập lớn
======================
Bài viết này chủ yếu hướng đến các site dạng tin tức, nội dung trong một URL hầu như không (hoặc ít) thay đổi. Các site dạng forum vẫn có thể áp dụng phương pháp này bằng cách giảm thời gian live time trên Redis. Tuy nhiên mình mới chỉ áp dụng trên site dạng tin tức nên không chắc chắn về hiệu năng đối với các site dạng forum

Có khi nào các bạn gặp tình trạng Server WP của mình đang yên đang lành thì bị sập mà k rõ nguyên nhân?, bạn đã cố gắng chỉnh sửa, cấu hình nginx & php, thậm chí cài đặt cache các kiểu nhưng chẳng khá khẩm hơn tí nào. Bạn restart server thì một lúc sau đâu lại vào đấy. Cảm giác bất lực mà chẳng thể làm gì hơn ?

Cá nhân mình đã có lúc như bạn, sau một thời gian tìm hiểu & thử nghiệm, câu trả lời cho bài toán này chỉ có 1 từ `Cache`. Tuy nhiên, cách mà bạn áp dụng lại quyết định tình trạng server của bạn:

Có 2 phương pháp để `Cache` : 

*	Cache in Local Disk
* 	Cache in Server Cache (Memcache/Redis)

Mỗi loại `Cache` trên đều có ưu nhược điểm riêng. Chúng ta sẽ cùng phân tích từng loại cache & nguyên nhân có thể gây ra tình trạng sập server nhé

### Cache using Local Disk

	**Flow:** Khi user request URL của Page trên WP lần đầu tiên, Hệ thống sẽ lưu lại đoạn mã HTML (cache) lên Local Disk sau khi WP render thành công trước khi trả về cho user. Sau đó, khi user request lại URL này, server sẽ kiểm tra xem có tồn tại cache của URL đó hay không, nếu có thì sẽ trả luôn đoạn HTML đó về cho người dùng & không cần tính toán gì cả. (Nâng cao hơn thì set thêm expired cho cache)

	**WP Plugin:** WP Super Cache

	**Ưu điểm:**
		* 	Không cần tới Server Cache của bên thứ 3
		*  Dễ thao tác, dễ cấu hình
		*  Không lo thiếu dung lượng bộ nhớ (thường bộ nhớ Local Disk rất lớn, hơn nữa, cache sẽ tự động xóa khi expired)

	**Nhược điểm:**
		*  Vì phải lưu file & đọc file liên tục nên tốc độ thực thi phụ thuộc vào I/O per second của Local Disk
		*  Khi có lượng truy cập lớn, server không sập vì thiếu RAM hay CPU mà sập vì I/O per second không đủ.

	#### Cache using Server Cache

	* 	**Flow:** Tương tự như flow trên Local Disk, chỉ khác là cache sẽ được lưu trên Server Cache thay vì lưu trên Local Disk
	*  **WP Plugin:** W3 Total Cache
	*  **Ưu điểm:**
		*  Tốc độ thực thi cực kỳ nhanh (do lưu trên RAM)
		*  Giảm tải cho Server vấn đề đọc & ghi cache (do đã có Server Cache lo vấn đề này)
	*  **Nhược điểm:** 
		*  Cần tới Server Cache của bên thứ 3
		*  Khó thao tác, cấu hình hơn Local Disk
		*  Cần kiểm soát vấn đề lưu & xóa cache, tránh full bộ nhớ
		*  Khi có lượng truy cập lớn, server không sập vì thiếu I/O per second mà do thiếu RAM, CPU. Các tiến trình của PHP-FPM được tạo ra để gọi tới server cache sẽ không kịp release memory. Do đó, khi có hàng loạt connection tới cùng lúc, server sẽ không cấp đủ memory cho PHP-FPM. Việc thực thi nhiều tiến trình PHP-FPM cùng lúc cũng tăng CPU usage, góp phần dẫn đến sập server.

Cả hai phương pháp `Cache` đều có thể dẫn đến sập server. Vậy làm sao để bảo đảm server **không bị sập** mà **không cần nâng cấp thêm server hiện tại** ?

### Cache using Server Cache, communicate without php-fpm

Như mình đã đề cập ở trên, nhược điểm của phương pháp `Cache using Server Cache` đó là khi có lượng truy cập lớn đồng thời, các tiến trình của PHP-FPM được tạo ra để kết nối tới server cache sẽ tiêu tốn hết RAM của server, tăng cpu usage dẫn tới server bị sập. 

Để khắc phục vấn đề này, chúng ta sẽ cấu hình để Nginx communicate trực tiếp với server cache thông qua Lua script. Bằng phương pháp này, chúng ta có thể bỏ qua bước thực thi PHP-FPM.

Trong bài viết này, mình sẽ hướng dẫn các bạn cài đặt & cấu hình nginx & lua script để communicate tới server cache (Redis), sử dụng OS Linux - Ubuntu. 

### I. Summary
*	**Ưu điểm**
	*	Không cần can thiệp vào source code của WordPress, tách riêng được phần code & phần hệ thống
	* 	Vì tốc độ thực thi của Nginx & Lua script là cực kỳ nhanh, tiêu tốn cực kỳ ít resource (Ram/Memory) nên có thể handle được lượng truy cập lớn & đồng thời
*  **Nhược điểm**
	*  Chỉ thực hiện được trên OS Linux
	*  Cần truy cập được SSH tới server
	*  Cần quyền Root để cài đặt & cấu hình
	*  Cần một chút hiểu biết về cách cấu hình trên Linux
	
### II. Prepare
Để Nginx kết nối tới Redis, chúng ta cần recompile nginx với Redis2 Module, Hash MD5 Module. Đối với các bạn đã cài đặt Nginx/Apache từ trước, các bạn cần unistall nó trước khi recompile
#### 1. Compiler environment

> Dependencies to compile

```{r, engine='bash', count_lines}
sudo apt-get install build-essential zlib1g-dev libpcre3 libpcre3-dev unzip
```
#### 2. Page Speed Module
> Module này có thì càng tốt, không có cũng không sao. Chi tiết về module này, các bạn xem tại link [Google Pagespeed Module](https://developers.google.com/speed/pagespeed/module/)

```{r, engine='bash', count_lines}
cd
NPS_VERSION=1.11.33.2
wget https://github.com/pagespeed/ngx_pagespeed/archive/release-${NPS_VERSION}-beta.zip -O release-${NPS_VERSION}-beta.zip
unzip release-${NPS_VERSION}-beta.zip
cd ngx_pagespeed-release-${NPS_VERSION}-beta/
wget https://dl.google.com/dl/page-speed/psol/${NPS_VERSION}.tar.gz
tar -xzvf ${NPS_VERSION}.tar.gz  # extracts to psol/
```
#### 3. Redis2 Nginx Module
> Module dùng để connect từ Nginx tới Redis

```{r, engine='bash', count_lines}
cd
REDIS2_VERSION=0.13
wget https://codeload.github.com/openresty/redis2-nginx-module/tar.gz/v${REDIS2_VERSION} -O redis2-nginx-module-${REDIS2_VERSION}.tar.gz
tar -xzvf redis2-nginx-module-${REDIS2_VERSION}.tar.gz
```
#### 4. Lua Jit
> Module LUA - Runtime Environment for LUA Language

```{r, engine='bash', count_lines}
cd
LUAJIT_VERSION=2.0.4
wget http://luajit.org/download/LuaJIT-${LUAJIT_VERSION}.tar.gz
tar -xzvf LuaJIT-${LUAJIT_VERSION}.tar.gz
cd LuaJIT-${LUAJIT_VERSION}
make && sudo make install
export LUAJIT_LIB=/usr/local/lib
export LUAJIT_INC=/usr/local/include/luajit-2.0
```
#### 5. Lua Nginx Module
> Module giúp Nginx communicate với Lua

```{r, engine='bash', count_lines}
cd
LUA_NGINX_VERSION=0.10.5
wget https://github.com/openresty/lua-nginx-module/archive/v${LUA_NGINX_VERSION}.tar.gz -O lua-nginx-module-${LUA_NGINX_VERSION}.tar.gz
tar -xzvf lua-nginx-module-${LUA_NGINX_VERSION}.tar.gz
```
#### 6. Nginx Developer Kit Module
> Module SDK for Nginx

```{r, engine='bash', count_lines}
cd
NDK_VERSION=0.3.0
wget https://codeload.github.com/simpl/ngx_devel_kit/tar.gz/v${NDK_VERSION} -O ngx_devel_kit-${NDK_VERSION}.tar.gz
tar -xzvf ngx_devel_kit-${NDK_VERSION}.tar.gz
```
#### 7. MD5 Nginx Module
> Module giúp Nginx tạo chuỗi Hash từ một chuỗi bất kỳ. Dùng để tạo Hash String cho URL

```{r, engine='bash', count_lines}
cd
sudo apt-get install libssl-dev
MD5_NGINX_VERSION=0.2.1
wget https://github.com/simpl/ngx_http_set_hash/archive/${MD5_NGINX_VERSION}.tar.gz -O ngx_http_set_hash-${MD5_NGINX_VERSION}.tar.gz
tar -xzvf ngx_http_set_hash-${MD5_NGINX_VERSION}.tar.gz

```
#### 8. Nginx Echo Module
> Module giúp các bạn có thể set experied time cho từng cache

```{r, engine='bash', count_lines}
cd
NGX_ECHO_VERSION=0.59
wget https://github.com/openresty/echo-nginx-module/archive/v${NGX_ECHO_VERSION}.tar.gz -O echo-nginx-module-${NGX_ECHO_VERSION}.tar.gz
tar -xzvf echo-nginx-module-${NGX_ECHO_VERSION}.tar.gz
```

#### 9. Nginx
> Không cần nói thêm gì về Nginx nhé

```{r, engine='bash', count_lines}
cd
# check http://nginx.org/en/download.html for the latest version
NGINX_VERSION=1.10.1
wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz
tar -xvzf nginx-${NGINX_VERSION}.tar.gz
```
### III. Compile
> Prepare thì nhiều vậy nhưng compile chỉ cần 3 dòng thôi

```{r, engine='bash', count_lines}
cd nginx-${NGINX_VERSION}/
./configure --user=nginx --group=nginx --sbin-path=/usr/sbin/nginx --pid-path=/run/nginx.pid --with-ld-opt="-Wl,-rpath,/usr/local/lib"  --add-module=$HOME/ngx_pagespeed-release-${NPS_VERSION}-beta ${PS_NGX_EXTRA_FLAGS} --add-module=$HOME/redis2-nginx-module-${REDIS2_VERSION} --add-module=$HOME/ngx_devel_kit-${NDK_VERSION} --add-module=$HOME/ngx_http_set_hash-${MD5_NGINX_VERSION} --add-module=$HOME/echo-nginx-module-${NGX_ECHO_VERSION} --add-module=$HOME/lua-nginx-module-${LUA_NGINX_VERSION} --conf-path=/etc/nginx/nginx.conf --pid-path=/run/nginx.pid --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log
make && sudo make install
```
### IV. Configure
#### 1. Nginx Daemon
Điều đầu tiên bạn cần quan tâm sau khi build nginx từ source chắc chắn là tập tin daemon để dễ dàng start/stop nginx đúng không. Đơn giản là bạn chỉ cần tạo tập tin `/etc/init.d/nginx` với nội dung như [file đính kèm](nginx.sh)

Sau đó chmod nó

```{r, engine='bash', count_lines}
sudo chmod a+rx /etc/init.d/nginx
```
Giờ bạn có thể start/stop/restart nginx server của mình bằng lệnh:

```{r, engine='bash', count_lines}
sudo service nginx restart
```
#### 2. Nginx - Redis
Mở tập tin cấu hình nginx server của bạn (thường nằm ở `/etc/nginx/sites-available/default`)
> Note: Mình sẽ bỏ qua các vấn đề về thiết lập cấu hình cơ bản cho Nginx như setup server block, location, php, etc. Chi tiết các bạn có thể tham khảo tại [How to install Nginx, Mysql, PHP on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-12-04)

Đầu tiên, chúng ta cần tạo một chuỗi Hash tương ứng với mỗi URL của bài viết. Trong `server block`, các bạn thêm đoạn code sau:

```{r, engine='nginx', count_lines}
set $current_uri $scheme://$http_host$request_uri;
set_md5 $ukey $current_uri;
```
Các bạn thay đổi <redis_url> và <port> tùy theo server redis của các bạn.

Đối với mỗi URL hiện tại `$current_uri`, chúng ta sẽ tạo một chuỗi MD5 tương ứng với URL đó & lưu vào biến `$ukey`. Chúng ta cũng cần kiểm tra chuỗi URL đó, nếu nó là file thì trả về file luôn. Nếu URL đó trỏ tới wp-admin thì forward nó tới `location ~ \.php$` Còn nếu không thì forward nó tới `location redis`


```{r, engine='bash', count_lines}
location / {
        try_files $uri $uri/ /redis;
}

location = / {
        try_files $uri /redis;
}
```

**location redis** 
> Thực hiện các tính toán logic như kiểm tra cache tồn tại hay không. Nếu có thì trả về cache, nếu không thì gọi tới php xử lý URL & cache output trước khi trả về cho người dùng.


```{r, engine='nginx', count_lines}
location = /redis {
    default_type text/html;
    add_header 'X-URI' "$current_uri";
    content_by_lua_block {
        local res = ngx.location.capture("/redis_check")
        if string.match(res.body, "1") then
            local data = ngx.location.capture("/redis_get")
            local inx = string.find(data.body, "<")
            if inx ~= nil then
                local html = string.sub(data.body, inx)
                ngx.print(html)
            else
                ngx.print(data.body)
            end
            ngx.print("<!--Service by nginx w redis-->")
        else
            local ori = ngx.location.capture("/original")
            local res
            if string.find(ngx.var.request_uri, "feed") then
                res = ngx.location.capture("/redis_set_date",
                    {
                        method = ngx.HTTP_PUT,
                        body = ori.body
                    }
                )
            else
                res = ngx.location.capture("/redis_set",
                    {
                    	method = ngx.HTTP_PUT,
                    	body = ori.body
                    }
                )
            end
            ngx.print(ori.body)
            if string.match(res.body, "OK") then
                ngx.print("<!--Original-->")
            else
                ngx.location.capture("/redis_flush")
                ngx.print("<!--Flushed-->")
            end
        end
    }
}

```

Như các bạn thấy trong đoạn script ở trên, chúng ta có 1 số location: 

**location redis_check**

> Kiểm tra xem URL hiện tại đã tồn tại trong redis hay chưa.


```{r, engine='bash', count_lines}
location = /redis_check {
        internal;
        redis2_query exists $ukey;
        redis2_pass 127.0.0.1:6379;
}
```
**location redis_get**
> Lấy nội dung cache từ server redis


```{r, engine='bash', count_lines}
location = /redis_get {
        internal;
        redis2_query get $ukey;
        redis2_pass 127.0.0.1:6379;
}
```
**location original**
> Gọi PHP thực thi URL & trả về code HTML


```{r, engine='bash', count_lines}
location = /original {
        try_files $uri /index.php?$args;
}
```
**location redis_set_date**
> Custom behavior của Redis đối với mỗi dạng của URL. Như **location redis** bên trên, mình cấu hình cho các URL có chứa từ khóa `feed` thì sẽ có thời gian expired ngắn hơn so với URL bình thường


```{r, engine='bash', count_lines}
location = /redis_set_date {
        internal;
        redis2_query set $ukey $echo_request_body "EX" 2700;
        redis2_pass 127.0.0.1:6379;
}
```

**location redis_set**
> Set cache cho một URL


```{r, engine='bash', count_lines}
location = /redis_set {
        internal;
        redis2_query set $ukey $echo_request_body "EX" 604800;
        redis2_pass 127.0.0.1:6379;
}
```

**location redis_flush**
> Remove all cached


```{r, engine='bash', count_lines}
location = /redis_flush {
        internal;
        redis2_query flushall;
        redis2_pass 127.0.0.1:6379;
}
```

Sau khi đã cấu hình Nginx xong, các bạn restart lại Nginx Daemon


```{r, engine='bash', count_lines}
sudo service nginx restart
```

Tới đây, site của các bạn đã có thể hoạt động bình thường với sự hỗ trợ từ nginx & redis cache. Tuy nhiên, các bạn cũng cần quản lý cache trên redis của mình. Ví dụ như nếu bạn viết một bài mới và muốn nó nằm trên trang chủ. Nhưng trang chủ đã có cache & cache chưa hết timeout thì phải làm thế nào ?

Mình có viết sẵn cho các bạn một plugin cho phép các bạn quản lý các vấn đề như vậy. Tuy nhiên mới chỉ ở mức cơ bản. Các bạn có thể xem hướng dẫn & tải về tại [Github Predis Cache](https://github.com/yehnkay/predis-cache)

### V. Lời kết
Tất cả những hướng dẫn mình viết trên đây đều dựa trên kinh nghiệm thực tiễn của mình. Kết quả khi mình áp dụng trên site của cty (xin lỗi mình không được phép dẫn link) thì không còn thấy tình trạng sập server & tốc độ thực thi cực kỳ nhanh. Do mình là dân lập trình, không phải dân hệ thống nên có thể có nhiều vấn đề mình chưa lường hết. Nếu các bạn gặp vấn đề gì khó khăn trong quá trình cài đặt hay phát hiện vấn đề gì chưa ổn thì vui lòng comment bên dưới hoặc gửi mail về địa chỉ `glmanhtu@gmail.com`

### VI. [UPDATED] Test Result
Để kiểm nghiệm lại phương pháp trên, mình đã thực hiện một bài test đơn giản dùng JMeter. Máy VPS mình dùng để test có cấu hình như sau: 
> CPU: Intel(R) Xeon(R) CPU E5-2630 0 @ 2.30GHz 1 Core
> 
> RAM: 512 MB

Đối với mỗi phương pháp cache, mình sẽ cài Web Server Nginx, Wordpress và dùng JMeter để gửi lần lượt 150, 200, 500, 750, 1000, 1500, 2000 requests đồng thời trong 1s tới trang chủ. Sau đó, ghi nhận lại % error để so sánh. % Error được JMeter tính là tỷ số của tổng request bị false (bad request) trên tổng số request.

Những phương pháp có dùng tới cache server (memcache/redis). Mình cũng cài luôn trên con server này. Do đó cũng có chút thiệt thòi cho các server này khi phải chia sẻ RAM/CPU cho cache server. Nếu các bạn đặt cache server ở một VPS khác thì hiệu năng có thể tốt hơn.

Mình sẽ cài luôn Mysql trên con server này để Wordpress dùng làm DB.

#### 1. Test without any Cache
Đầu tiên, mình sẽ test xem khả năng chịu đựng của server khi không sử dụng bất kỳ phương pháp cache nào. Mình chỉ cài webserver nginx & Wordpress.

#### 2. Test Cache using Local Disk
Để test `Cache using Local Disk`, mình cài WP Super Cache plugin lên server

#### 3. Test Cache using Server Cache
Để test `Cache using Server Cache`, mình xóa WP Super Cache plugin & cài W3 Total Cache, Memcached server. Kết nối W3 Total Cache tới Memcached server

#### 4. Test Cache using Server Cache, communicate without php-fpm
Để test phương pháp này, mình xóa Memcached server, W3 Total Cache. Cài Redis server & connect Nginx tới Redis server vừa cài.

### Kết Quả

|                                                       | 150 | 200 | 400 | 750 | 1000  | 1500  | 2000   |
|-------------------------------------------------------|-----|-----|-----|-----|-------|-------|--------|
| Without any Cached                                    | 0%  | 28% | 64% | 80% | 83%   | 84%   | 86%    |
| Cached using Local Disk                               | 0%  | 0%  | 0%  | 0%  | 0%    | 7%    | 14%    |
| Cached using Server Cached                            | 0%  | 0%  | 0%  | 25% | 28.8% | 25.6% | 30.35% |
| Cache using server cache, communicate without php-fpm | 0%  | 0%  | 0%  | 0%  | 0%    | 0%    | 0.5%   |
