---
layout: post
title: Načítání Facebook postů v Nette Framework
published: true
---

{{ page.title }}
================

<p class="meta">12 Jun 2014 - Prague (Bohemia / Czech Republic)</p>

``` DRAFT ```


Na jednom projektu jsem narazil na potřebu zobrazovat posty z Facebooku na klientově stránce. Inu začal jsem psát prototyp jak bych danou věc řešil. Prototyp jsem "spíchnul" za hodinku, ale bylo to uděláno tak trošku na hulváta. Tak jsem si řekl že to postupně přepíšu tak jak by to třeba napsal nějaký zkušený programátor s Nette frameworkem. Na http://srazy.info/nettefwpivo jsem danou věc přednesl a Nette guruové mi přislíbili odbornější konzultace, tak doufám že se nám podaří vytvořit návod jak by se takové věci nad Nette měli psát.

celý projekt je pak k naleznutí na Githubu https://github.com/chemix/Nette-Facebook-Reader

Pokud máte nápad jak danou ukázku vylepšit nebo článek upravit, pošlete pull request, nebo mi napište email.



321... START
=============

Začneme čistým projektem vycházející z Nette/Sandbox

`composer create-project nette/sandbox Nette-Facebook-Reader`

`cd Nette-Facebook-Reader`

zápis do "log" a "temp" folderu

`chmod -R a+rw temp log`

vyčistíme homepage šablonu a připravíme si nový Import Presenter se šablonou.

/app/templates/Import/default.latte
```html
{block #content}
	<h1>Import</h1>
```
a /app/presenters/ImportPresenter.php

```php
namespace App\Presenters;

use Nette,
	App\Model;

class ImportPresenter extends BasePresenter
{

	public function renderDefault()
	{

	}

}
```

Přidáme trošku Facebooku
========================

Přidáme si do composer.json závislost na Facebook SDK

```json
	"require": {
		"php": ">= 5.3.7",
		"nette/nette": "~2.2.0",
		"dg/adminer-custom": "~1.0",
		"facebook/php-sdk-v4" : "4.0.*"
	},
```

a stahneme si ho pomoci `composer update`.

Přihlásime se na Facebook Developers  http://developers.facebook.com a vytvoříme novou aplikaci.

![Facebook Developer](/image/nette-facebook-reader/facebook-developer.png)

Stačí nám vyplnit pouze její název

![Create new App](/image/nette-facebook-reader/create-new-app.png)

a zjistíme si Facebook App ID a App Secret

![Facebook App IDs](/image/nette-facebook-reader/facebook-app-ids.png)

Na hulváta si zkusíme načíst data skrze Facebook Graph Api. V našem Import Presenteru přidáme do hlavičky

```php
use Facebook\FacebookRequest;
use Facebook\FacebookRequestException;
use Facebook\FacebookSession;
use Tracy\Dumper;
```

a metodu přepíšeme podle ukázky z dokumentace k PHP Facebook SDK

```php
public function renderDefault()
{
	FacebookSession::setDefaultApplication('YOUR_APP_ID', 'YOUR_APP_SECRET');

	$session = FacebookSession::newAppSession();
	$data = array();

	try {
		$request = new FacebookRequest($session, 'GET', '/nettefw/feed');
		$response = $request->execute();
		$posts = $response->getGraphObject()->asArray();

		$data = $posts['data'];
	} catch (FacebookRequestException $ex) {
		// Session not valid, Graph API returned an exception with the reason.
		echo $ex->getMessage();
		exit();
	} catch (\Exception $ex) {
		// Graph API returned info, but it may mismatch the current app or have expired.
		echo $ex->getMessage();
		exit();
	}

	Dumper::dump($data);
}
```

po zkouknutí výstupu bychom měli vidět dump pole o 25 položkách

![dump array](/image/nette-facebook-reader/dump-array-25.png)

Malinko si zjednodušíme ošetření chyb jen na ukončení aplikace.

```
} catch (\Exception $ex) {
	throw $ex;
	$this->terminate();
}
```


Cache, ať při vývoji nečekáme
=============================

Přidáme si možnost cache pro požadavek. (Hlavně si tím urychlíme další rozšiřování, přeci jen čekat 10 sec na každý refresh mě nebaví).

O cachovaní se nám postara Nette Framework a jeho Nette\Caching\Cache;

Přidáme do use sekce

```
use Nette\Caching\Cache;
```

Abychom mohli vytvořit instanci třídy Cache tak jí musíme předat nějaký storage kam si bude ukládat data. Viz API dokumentace

```
public function __construct(IStorage $storage,
```

A ten si necháme poslat (injectnout) do třídy pomoci Nette Dependenci Injection. Jediné co musíme udělat je definovat public property $cacheStorage typu \Nette\Caching\IStorage a pomocí anotace @inject nám framework zařídí vše potřebné.

```
class ImportPresenter extends BasePresenter
{
	/**
	 * @var \Nette\Caching\IStorage @inject
	 */
	public $cacheStorage;
```

hup, a v našich metodách se ke storage dostaneme snadno pomocí `$this->cacheStorage`

```
$cache = new Cache($this->cacheStorage, 'facebookWall');
$data = $cache->load("stories");
```

Více o cache si nastudujete v dokumentaci Nette\Caching.

V našem případě pokud se nám nepodaří načíst data z cache (a to se nám napoprvé určitě nepovede) tak si je načteme z Facebooku a do té cache si je uložíme:

```
$cache->save("stories", $data, array(
	Cache::EXPIRATION => '+30 minutes',
	Cache::SLIDING => TRUE
));
```

Výsledkem je

```
public function renderDefault()
{
	FacebookSession::setDefaultApplication('YOUR_APP_ID', 'YOUR_APP_SECRET');

	$session = FacebookSession::newAppSession();
	$cache = new Cache($this->cacheStorage, 'facebookWall');
	$data = $cache->load("stories");

	if (empty($data)) {
		try {
			$request = new FacebookRequest($session, 'GET', '/nettefw/feed');
			$response = $request->execute();
			$posts = $response->getGraphObject()->asArray();

			$data = $posts['data'];
			$cache->save("stories", $data, array(
				Cache::EXPIRATION => '+30 minutes',
				Cache::SLIDING => TRUE
			));

		} catch (\Exception $ex) {
			throw $ex;
			$this->terminate();
		}
	}

	Dumper::dump($data);
}
```

Nyní by nám druhý request měl trvat výrazně kratší dobu.

![with cache](/image/nette-facebook-reader/with-cache.png)

Ukládáme posty do databáze
==========================

Dalším krokem je uložit si získána data do databáze. Pro práci s databází použijeme třídu Nette\Database. Vytvoříme si databázi a uživatele (díky klonování Nette Sandbox máme výborný nástroj Adminer přímo u projektu /adminer/).

Uživatel bude facebookwall a s heslem 'tajneheslo' bude mít přístup ke všem databázím začínající facebookwall_ v našem vývojovém případě konkrétně k databázi facebookwall_devel

```
CREATE DATABASE `facebookwall_devel` COLLATE 'utf8_czech_ci';
CREATE USER 'facebookwall'@'localhost' IDENTIFIED BY 'tajneheslo';
GRANT USAGE ON * . * TO 'facebookwall'@'localhost' IDENTIFIED BY 'tajneheslo' WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0 ;
GRANT ALL PRIVILEGES ON `facebookwall\_%` . * TO 'facebookwall'@'localhost';
FLUSH PRIVILEGES;
```

a vytvoříme si tabulku kam si posty budeme ukládat.

```
CREATE TABLE `facebook_wallposts` (
  `id` varchar(100) CHARACTER SET ascii NOT NULL,
  `created_time` datetime NOT NULL,
  `updated_time` datetime NOT NULL,
  `type` varchar(100) CHARACTER SET ascii NOT NULL,
  `status_type` varchar(250) COLLATE utf8_czech_ci NOT NULL,
  `name` varchar(250) COLLATE utf8_czech_ci NOT NULL,
  `link` varchar(250) COLLATE utf8_czech_ci NOT NULL,
  `message` text COLLATE utf8_czech_ci NOT NULL,
  `caption` text COLLATE utf8_czech_ci NOT NULL,
  `picture` varbinary(250) NOT NULL,
  `status` char(1) COLLATE utf8_czech_ci NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  KEY `type` (`type`),
  KEY `created_time` (`created_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_czech_ci;
```

V presenteru si řekneme ať nám framework předá objekt Nette\Database\Context

```
/**
 * @var \Nette\Database\Context @inject
 */
public $database;
```

v konfiguračním souboru /app/config/config.local.neon si doplňíme připojení k databázi.

```
nette:
	database:
		dsn: 'mysql:host=127.0.0.1;dbname=facebookwall_devel'
		user: facebookwall
		password: mojetajneheslo
		options:
			lazy: yes
```

Pozor na zápis, liší se od Dibi a občas mě to dokáže zabrzdit ;-)


V presenteru si pak na hulváta doplníme ukládání jednotlivých řádku, připadne po opakovaném importu aktualizaci postit

```php
// save data to database
if (is_array($data) && !empty($data)) {
	foreach ($data as $rowPost) {

		$post = array(
			'id' => $rowPost->id,
			'type' => $rowPost->type,
			'created_time' => $rowPost->created_time,
			'updated_time' => $rowPost->updated_time,
		);

		if (isset($rowPost->name)) {
			$post['name'] = $rowPost->name;
		}
		if (isset($rowPost->link)) {
			$post['link'] = $rowPost->link;
		}
		if (isset($rowPost->status_type)) {
			$post['status_type'] = $rowPost->status_type;
		}
		if (isset($rowPost->message)) {
			$post['message'] = $rowPost->message;
		}
		if (isset($rowPost->caption)) {
			$post['caption'] = $rowPost->caption;
		}
		if (isset($rowPost->picture)) {
			$post['picture'] = $rowPost->picture;
		}

		// post 'status' use story, we need message
		if ($rowPost->type == 'status') {
			if (isset($rowPost->story)) {
				$post['message'] = $rowPost->story;
			}
		}

		try {
			$this->database->table('facebook_wallposts')->insert($post);
		} catch (\PDOException $e) {
			if ($e->getCode() == '23000') {
				$this->database->table('facebook_wallposts')->where('id', $rowPost->id)->update($post);
			} else {
				throw $e;
			}
		}
	}
}
```

můžeme se přesvědčit, že se nám vše uložilo

![save posts to database](/image/nette-facebook-reader/save-to-database.png)

Předáme si výpis práve přidaných postů do šablony a tam si je vypíšeme.

```
// send data to template
$this->template->wallPosts = $data;
```

a
```
{foreach $wallPosts as $post}
	type: <strong>{$post->type}</strong> <br>
	<small>
		id: {$post->id}<br>
		created_time: {$post->created_time} <br>
		updated_time: {$post->updated_time} <br>
	</small>

	{ifset $post->name}
		name: <h1>{$post->name}</h1>
	{/ifset}

	{ifset $post->status_type}
		status_type: {$post->status_type} <br>
	{/ifset}

	{ifset $post->message}
		message: {$post->message} <br>
	{/ifset}

	{ifset $post->picture}
		piscture: {$post->picture} <br>
		<img src="{$post->picture}" />
	{/ifset}

	{ifset $post->link}
		link: <a href="{$post->link}">{$post->link}</a> <br>
	{/ifset}

	{ifset $post->caption}
		caption: {$post->caption} <br>
	{/ifset}
	<hr>
{/foreach}
```

Zobrazení postů na homepage
===========================

V HomepagePresenteru si načteme posty co jsme načetly importem. Jelikož, ale nechcem zobrazovat všechny posty nastavíme si u některých v databázi status na 1 a budem zobrazovat pouze tyto.

```
public function renderDefault()
{
	$facebookWallPosts = $this->database->table('facebook_wallposts')->where('status','1')->limit(5)->fetchAll();
	$this->template->wallPosts = $facebookWallPosts;
}
```

a nesmíme zapomenout na přidání public property $database;

```
	/**
	 * @var \Nette\Database\Context @inject
	 */
	public $database;
```

šablona pak může vypadat nějak takto:

{% highlight html %}
<div n:if="$wallPosts" class="facebook-posts">
	<div n:foreach="$wallPosts as $post" class="post {$post->type}">
		<h3 n:if="$post->name">{$post->name}</h3>
		<img n:if="$post->picture" src="{$post->picture}" />

		{if $post->message}
			<p>
				{$post->message|truncate:250:'...'}
				<a n:if="$post->link" href="{$post->link}">více</a>
			</p>
		{else}
			{if $post->link}
				<a href="{$post->link}">více</a>
			{else}
				<a href="http://www.facebook.com/nettefw">více</a>
			{/if}
		{/if}
	</div>
</div>
{% endhighlight %}

tuto verzi najdete pod tagem [:prototype](https://github.com/chemix/Nette-Facebook-Reader/tree/prototype)

