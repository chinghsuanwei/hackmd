---
title: 如何寫一個 Woocommerce 點數折抵 的plugin
date: 2020-05-09 20:00
tags: WordPress, Woocommerce, php
categories:
- Project
---

最近接到一個e-commerce的案子

案主的網站主要是由 WordPress + Woocommerce  所組成的商城

其實Woocommerce 本身就有出紅利折抵的plugin **WooCommerce Points and Rewards**

但此案主 和 另外一方的公司(以下皆稱乙方)有合作 乙方有自己的點數 乙方的網站會員能夠登入到案主的網站 並可以使用乙方點數消費

所以統整一下 主要會有幾個功能

 提供給乙方會員登入入口

 能查詢/使用/退還 乙方點數

 要提供介面，讓後台管理員能輸入 對 全域/分類/單一商品 折抵的%數

 其餘小功能 類似Log...之類的

**WordPress** version 5.3.2

**Woocommerce** version 3.9.2

**WooCommerce Points and Rewards**  version 1.6.21

開發環境
OS: Windows 10
Web server: Appserv(Apache + mysql  + phpMyAdmin)

### 一、WordPress Migration

要修改既有的WordPress 網站 第一件事情就是將Live-site移到Local讓我們能亂玩

這裡我們使用 WordPress的 **All-in-one WP Migration**

但遇到的問題是php設定 限制了上傳檔案大小 (我使用php 7)

因為案主的WordPress封裝檔 有1GB

我使用的是AppServ裝的php 要修改的路徑在 [Appserv安裝路徑]\php7\php.ini

修改單一檔案大小上限：

```
; Maximum allowed size for uploaded files.
; http://php.net/upload-max-filesize
upload_max_filesize = 16M
```
修改 POST 資料大小上限：

```
; Maximum size of POST data that PHP will accept.
; Its value may be 0 to disable the limit. It is ignored if POST data reading
; is disabled through enable_post_data_reading.
; http://php.net/post-max-size
post_max_size = 32M
```

似乎 也要修改此Plugin的 constants.php 將值改成

```
...
define( 'AI1WM_MAX_FILE_SIZE', 2 << 30 );
...
```

不然他會Plugin會限制最大的WP封裝檔 512MB 

### 二、 登入功能

UI的部分我是直接修改wp-login.php，多新增一個按鈕 和 UI 讓乙方會員可以登入

在登入認證的部分 我是在我的plugin裡 加入 init的hook 兼 判斷是否為登入頁面(這邊我覺得可能有更好的hook可以使用)


```php
add_action( 'init', function () {
	if ( $GLOBALS['pagenow'] === 'wp-login.php' )
	{
        ...
```
再者是創立WP帳號 帳號 和 密碼 我是用 encryption(乙方帳號) 產生

最後在導向首頁

```php
$user_id = wp_create_user( $username, $password, $email_address );
$user = new WP_User( $user_id );
$user->set_role( 'customer' );

...
    
$creds = array();
$creds['user_login'] = $username;
$creds['user_password'] = $password;
$creds['remember'] = true;
			
$user = wp_signon( $creds, false );
$userID = $user->ID;
wp_set_auth_cookie( $userID, $remember=true, false );
wp_set_current_user($username);
			
wp_redirect(get_home_url());
exit();
```

### 三、能查詢/使用/退還 乙方點數

這邊就比較複雜 要熟悉WP的hook運作機制

但基本上的改法就是 把plugin裡面的參數 wc_points_xxxx 改成 miso_points_xxxx之類的

然後將本來在mysql 做點數控管的部分 改成 Call 乙方提供的API 其實這樣就完成了

可以看看下圖

![](/images/woocommerce-points-and-rewards-demo1.png)

紅利點數是使用 **WooCommerce Points and Rewards**  建置的

而下面的那欄就是乙方點數 UI是不是很像呢? 因為UI的部分 我就直接使用 **WooCommerce Points and Rewards**  plugin裡面的css

紅利點數 和 乙方點數可以同時使用 不衝突

![](/images/woocommerce-points-and-rewards-demo2.png)

Admin的部分也是沿用，因為案主只需要能設定 全域/分類/單一商品 折抵的%數

所以大部分的設定都被comment掉了

![](/images/woocommerce-points-and-rewards-demo3.png)



<strike>最後此專案只有一個地方 沒有寫進 plugin</strike>

<strike>就是wp-login.php裡面的UI部分 知道怎麼做 不過覺得有點麻煩</strike>

<strike>其實用wc_enqueue_js 另外把 login的UI包進去 append到wp-login.php即可</strike>

<strike>但我想 若是此plugin有在串接到別的 同樣使用woo-commerce網站上 我在把它完成吧</strike>

### Extra、將wp-login.php Code移到Plugin

因為案主會自動更新WordPress版本，或著之後可能會更新 此時wp-login.php 的code就會被新版本的WordPress洗掉

UI主要有三個部分

第一個是要顯示的Button

第二個是要載入的css

第三個是彈跳視窗

我把這三個部分都拆開



```php
function load_scripts()
{
    wp_enqueue_style("ml", plugin_dir_url(__FILE__) . "style.css");
    wp_enqueue_script("ml", plugin_dir_url(__FILE__) . "miso-login.js");
    include( dirname( __FILE__ ) .'/miso-login.php' );
}

add_action( 'login_footer', 'load_scripts' );
```

style.css 是單純的css

miso-login.js是在UI上加入的Button

miso-login.php是彈跳視窗和一兩行的php code



關於第一部分 我用的方法是 JQuery的insertAfter 記得這邊必須要用JQuery帶入$的符號

原因可參考: https://stackoverflow.com/questions/3931529/is-not-a-function-jquery-error

裡面應該是講為了不讓$符號撞到其他Library 他有Call **jQuery.noConflict()**

```javascript
jQuery(function($) {
	var $miso_object = $("<div class=\"miso-custom-login-form-main\" onclick=\"document.getElementById('miso_login_panel').style.display='block'\"><div class=\"miso-container\" data-align=\"left\" style=\"display: block;\"><div class=\"miso-label-container\"><button class=\"miso-container-button\">以 <b>Miso帳號</b> 登入</button></div></div></div>");
	$miso_object.insertAfter($("#loginform"));
});
```



