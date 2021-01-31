---
title: Laravel 4
tags:
  - Laravel
  - PHP
date: 2013-06-04T00:01:06+08:00
---

我自己學習 PHP 其實都只是一年前開始，這是因為我選修了兩科關於 PHP 的科目。那時學的都是一些最基本的 PHP，製作 HTML 表單連驗證、修改、儲存到資料庫都是用同一個 PHP 檔案，內裏用幾個 `if` 來分隔開不同的部分，還會用 session 來重新填寫表單內容。現在回想起都覺得這種寫法非常嘔心，因為實在太長，而且夾雜着 PHP 和 HTML，自己寫完都不想再去看。

<!--more-->

所以那時就試用 [CodeIgniter](http://ellislab.com/codeigniter) 這套 PHP framework 來做 project，因為這些 Framework 都會替你分好 MVC，不用自己慢慢研究怎樣分，只要跟着它的習慣去寫就不會再有一個 PHP 檔做全部的動作。 過了幾個月之後，在 [Nettuts+](http://net.tutsplus.com/) 看到有關 [Laravel](http://laravel.com/) 的介紹，覺得它不錯。它提供的 function 就算不看說明文件也可以大約估計到它的意思，而且說明文件簡單明瞭，大部分要用到的功能都會在說明文件中找到，不用查 API。據介紹 Laravel 是抄 [Ruby on Rails](http://rubyonrails.org/) 的特色，例如有：

* ActiveRecord：即是 Laravel 叫的 Eloquent ORM (Object Relational Mapping)，可以用簡單易明的語法來做 database CRUD，無需寫 SQL statement
* Migration：將 Database schema 的改動寫入 PHP 檔內，再用 Laravel 提供的 Artisan CLI 工具執行或者還原到上一個版本。方便和其他開發人員同步各自 database 的 schema，不用再 export SQL file
* Generate：透過 Laravel 4 的 Artisan CLI 工具產生 PHP 檔案（例如 Controller class、Model class、View 等等，Laravel 3 可以裝第三方的 command 來補上這個功能）
* Gems：Laravel 3 是用自家制式的 Bundle；Laravel 4 是用 [Composer](http://getcomposer.org/)
* [RESTful](http://zh.wikipedia.org/wiki/REST)：可以做到 RESTful controller，Laravel 4 還新加了 Resource controller
再加上一堆常用的現成功能如 HTML 表單製作 helper class、表單驗証、登入系統、Blade template（Laravel 自家 Templating 格式）、Pagination、Routing 等等令開發時間加快。上月開站的 [CityU GE 指南](http://cityuge.swiftzer.net/)都是用 Laravel 3 寫的，再配 [Twitter Bootstrap](http://twitter.github.io/bootstrap/)、[Font Awesome](http://fortawesome.github.io/Font-Awesome/) 做 UI。

而 Laravel 4 (L4) 在上星期推出了 4.0.0 版，這個版本基本上是重新寫過。亦正因為它是重新寫過，大部分 Laravel 3 (L3) 的 Class 和 Function 都改了，如果要升級的話就要逐個 PHP 檔案改成 L4 的 Class 和 Function。除了加添更多新功能外，L4 最大的改變是轉用 Composer 來管理組件，整個 Framework 都是由不同的組件組合而成。如果不喜歡 Laravel 的作法可以裝原裝組件換成自己喜歡的，例如將 Blade 換做 [Smarty](http://www.smarty.net/)。而 L4 本身亦用上了其他現成的組件，例如 [Symfony](http://symfony.com/)、[whoops!](http://filp.github.io/whoops/)（L4 那個很漂亮的錯誤報告畫面就是用這個 Library 做出來的）、[Swift Mailer](http://swiftmailer.org/) 等等。組件的安裝和更新可以用 Composer 替你完成，不用再自己手動換檔案。另一樣比較明顯的變化是 L4 遵守 [PSR-0](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md) and [PSR-1](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md) 標準，Class 名要用 StudlyCaps、Function 名要用 camelCase，和新的 PHP 5 class 或者 Java 一樣。幸好 CityU GE 指南的規模不大，三兩日就可以將原先的 code 升級成 L4。首先要大改的就是將原先的 snake_case variable / function / class 名按照 PSR-1 標準替換。其次是將 class / function 名變成 L4 的（因為有不少常用的 function 都改了 signature）。之後再到 [Packagist](https://packagist.org/) 找尋 reCAPTCHA、RSS feed 和 Sitemap class 的組件取代原先的 L3 bundle，並用 Composer 加到 project 內。這個過程都算輕鬆，因為這些組件是一般常用的組件，很容易找到。

{{< figure src="laravel-composer.jpg" title="Laravel 4 改用 Composer 管理組件的安裝、更新及相依性" >}}

Laravel 3 和 Laravel 4 的用法上的改動可以參考 Jeffery Way 的 [Laravel 4 Mastery](http://net.tutsplus.com/tutorials/php/laravel-4-mastery/) 和 Jason Lewis 的 [Laravel 4: Illuminating Your Laravel 3 Applications](http://jasonlewis.me/article/laravel-4-illuminating-your-laravel-3-applications)。不過因為這些影片和文章都是在 Beta 版時寫的，和正式版會有小量出入。以下是我升級 CityU GE 指南時主要遇到的變化：

## HTML、Form、Blade、Asset Management、i18n

* L3 用來產生 HTML `<a>` tag 的 `HTML::link()`、`HTML::link_to_route()`、`HTML::link_to_secure()` 在 L4 變成了 global function `link_to()`、`link_to_action()`
* `Form` class 仍然存在，在 L4 beta 時曾傳出 L4 會取消
* `Form::open()` 的傳入參數改為一個 associative array
* `Asset` class 已經消失，要自己找替代品，或者乾脆不用
* Blade 的 `@include()` 易名為 `@extends()`
* Blade 的 `@endsection` 易名為 `@stop`（但不轉也可以用到，但上面的 `@include` 就一定要改）
* Blade 新增了`{{{ }}}` 語法，括在裏面的 variable 會做一次 HTML escape；而原有的 `{{ }}` 語法就和以前一樣不會 escape
* `e()` 這個 escape HTML entity 的 function 仍然存在
* `Lang` class 已經可以支援單數 / 眾數 / 個別數量時會套用不同款的翻譯字串
* L3 內 `__()` 這個模仿 gettext 的 functoin 在 L4 改為 `trans()` 和 `trans_choice()` 或者 `Lang::get()` 和 `Lang::choice()`

## Pagination

* Paginate 過的 variable 如果要取得本頁的行列的話 L3 是 `foreach ($courses->results as $course) {...}`；L4 則不用再加 `results`，即是 `foreach ($courses as $course) {...}`
* 因為上述的改動，如果要檢查 paginate 過的結果是否為空的話，要使用 `if (!$courses->getTotal()) {...}`

## Route

* 新增了 resource controller：`Route::resource()`（可以方便做 RESTful API，它會自動產生 RESTful URL，並且會有 route name）
* 新增了 sub-domain routing
* 新增了 route prefix（即是網址中段的 directory 名），例如要做一個 admin section，不用再每個 admin section 的 route 加 `/admin/`，可以用 `Route::group(array('prefix' => 'admin'), function() {...})` 包起所有 admin section 的 route

## URL、Request、Response

* `URL::base()` 這個 function 改為使用 `URL::to('')`
* L3 要取得目前這個 request 的 route name 是用 `Request::route()->action['as']` 而 L4 是用 `Route::currentRouteName()`
* 要送出 HTTP 404、500 等 error，L3 是用 `return Response::error('404')` 而 L4 是用 `return App::abort(404)`

## Raw Query、Eloquent、Migration

* L3 raw query 的 `DB::only()` 在 L4 已經取消
* L4 可以用到 transaction
* L3 要執行 raw query 是 `DB::query()` 而 L4 則分拆成 `DB::select()`、`DB::insert()`、`DB::update()`、`DB::delete()` 和 `DB::statement`
* Eloquent model 的 getter 和 setter function 的命名規則分別改為 `getFooAttribute()` 和 `setFirstNameAttribute()`，`Foo` 和 `FirstName` 是 database column 名（留意 column 名仍然是用 snake_case，所以 `setFirstNameAttribute()` 是對應 database table 的 `first_name` column）
* 新加了 database seed 功能，可以不用再將填測試或預設資料寫成 DB schema 的 migration，並且在有需要時才導入到資料庫中
* 經 Eloquent 取得的 `created_at`、`updated_at` 和 `deleted_at` column 不再是普通的 string，而是 extends 自 `DateTime` 的 [Carbon](https://github.com/briannesbitt/Carbon) object，它提供一些方便易用的 function 來切換顯示格式、運算和處理這些日期和時間

## Artisan

* 新增了 `php artisan serve`，只要讓 console 開着的話就可以用 PHP 內置的網頁伺服器運行這個 Laravel project，可以不用設定 VirtualHost
* 新增了 `php artisan controller:make`，用來產生 resource controller
* 新增了 `php artisan optimize`，用來產生最佳化過的 class loader，加快載入速度
* 新增了 `php artisan migrate:refresh`，可以還原所有 migration 並重新執行所有 migration

## 其他

* `dd()` 仍然可以使用
* L4 目前未有內置 profiler bar (Anbu)，但可以用 [loic-sharma/profiler](https://packagist.org/packages/loic-sharma/profiler) 代替
其實還有很多的改變，只是我在這次升級未有遇到。要查 L4 的說明文件可以到 [Laravel 官方網站](http://laravel.com/docs)，官方網站亦都有 L4 [API](https://laravel.com/api/4.2/)。說明文件並沒有將所有 function 列出，如果要找一些比較少用的 function 就要到 API documentation 查。如果已經有 L3 的 project 的話，其實可以繼續用 L3。因為升級不只是換 Laravel 的核心檔案，還要逐個檔案去改。但如果是要開始一個新 project 的話，就最好用 L4。因為 L3 將會淡出，日後未必再有新的 L3 bundle 可以用。相反，L4 是用 Packagist，愈來愈多 PHP framework 都開始支援 Packagist / Composer，這意味着日後會有更多現成組件可以用，節省自己開發的時間。
