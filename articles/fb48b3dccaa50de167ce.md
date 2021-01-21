---
title: "PHP Carbonで月の足し引きオーバーフローにはまった"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php"]
published: true
---

PHPで日付操作をする時は[Carbon](https://carbon.nesbot.com/docs/)によくお世話になっていますが、いい感じにやってくれるだろうと殴り書きしていたら月の日付がオーバーフローしてしまいました。

## 環境

```
$ php --version
PHP 7.4.13 (cli) (built: Jan 20 2021 00:29:11) ( NTS )
Copyright (c) The PHP Group
Zend Engine v3.4.0, Copyright (c) Zend Technologies
    with Zend OPcache v7.4.13, Copyright (c), by Zend Technologies
    with Xdebug v2.9.6, Copyright (c) 2002-2020, by Derick Rethans
```

```
$ composer --version
Composer version 1.10.15 2020-10-13 15:59:09
```

composerのバージョンについて叱られますが。

## 月と日数

人類の常識ですが、月の日数は各月毎に同じとは限らないので日付から月を足し引きするといった操作をするときは、足した（引いた）先の月の日数が足す（引く）前の月の日数以下であった場合の動作を決めなければなりません。
```
e.g 2021-01-31 + 1 months = 2021-02-28 or 2021-03-01
e.g 2021-03-31 - 1 months = 2021-02-28 or 2021-03-01
※2020年2月31日は存在しない
```

## Carbonの動作

PHP Carbonには便利なメソッドがたくさんありますが、ここでドキュメントをよく見てみます。

>By default, Carbon relies on the underlying parent class PHP DateTime behavior. As a result adding or subtracting months can overflow
>>デフォルトでは、Carbonは基礎となる親クラスのPHPDateTime動作に依存しています。その結果、月の加算または減算がオーバーフローする可能性があります

https://carbon.nesbot.com/docs/#api-addsub


だそうです。(google translate)
次にPHP DateTimeのドキュメントを確認してみます。

>月を加算するときの注意点

https://www.php.net/manual/ja/datetime.add.php

ちゃんと書かれています。

実際にやってみます。

```
composer require nesbot/carbon
```

```php
<?php

require './vendor/autoload.php';

$month = new DateInterval('P1M'); // 1 month of period

echo (new DateTime('2020-01-31'))->add($month)->format('Y-m-d').PHP_EOL;
echo (new DateTime('2020-03-31'))->sub($month)->format('Y-m-d').PHP_EOL;
echo Carbon\Carbon::create(2020, 1, 31)->addMonth().PHP_EOL;
echo Carbon\Carbon::create(2020, 3, 31)->subMonth().PHP_EOL;

/**
 *  出力
 *
 * 2020-03-02
 * 2020-03-02
 * 2020-03-02 00:00:00
 * 2020-03-02 00:00:00
 */
```

2020年の2月は29日間なので差分がドキュメントの例より1日短くなります。

## オーバーフローしないようにする

PHP Carbonのドキュメントにはしっかり対策(?)が書かれています。

- インスタンスのオーバーフロー設定
```php
<?php

$dt = Carbon\Carbon::create(2020, 1, 31);
$dt->settings([
    'monthOverflow' => false
]);

echo $dt->addMonth();

/**
 * 出力
 *
 * 2020-02-29 00:00:00
 */
```

- オーバーフローしない月の加算、減算メソッド

`addMonthNoOverflow()` `addMonthWithoutOverflow()` `subMonthNoOverflow()` `subMonthWithoutOverflow()`
他にも`xxxNoOverflow()` `xxxWithoutOverflow()`といったメソッドが複数存在するようです。

```php
<?php
echo Carbon\Carbon::create(2020, 1, 31)->addMonthNoOverflow().PHP_EOL;
echo Carbon\Carbon::create(2020, 1, 31)->addMonthWithoutOverflow().PHP_EOL;
echo Carbon\Carbon::create(2020, 3, 31)->subMonthNoOverflow().PHP_EOL;
echo Carbon\Carbon::create(2020, 3, 31)->subMonthWithoutOverflow().PHP_EOL;

/**
 * 出力
 *
 * 2020-02-29 00:00:00
 * 2020-02-29 00:00:00
 * 2020-02-29 00:00:00
 * 2020-02-29 00:00:00
 */
```

最終的にどちらの設定にするかは業務内容に依ると思いますが、直感的にはオーバーフローしない方じゃないでしょうか?。

## X月前、X月後という言葉

月の足し引き、少し考えずとも注意しなければならないのは明らかなのにはまってしまいました。
普段なにも考えずに何ヶ月後の何日とか言ったり考えたりしてるのも、よく考えればふわっとしている気がします。

月の足し引きには気をつけましょう！
