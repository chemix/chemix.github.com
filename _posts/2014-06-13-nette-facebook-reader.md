---
layout: post
title: Načítání Facebook postů v Nette Framework
published: true
---

{{ page.title }}
================

<p class="meta">12 Jun 2014 - Prague (Bohemia / Czech Republic)</p>

`DRAFT`


Na jednom projektu jsem narazil na potřebu zobrazovat posty z Facebooku na klientově stránce. Inu začal jsem psát prototyp jak bych danou věc řešil. Prototyp jsem "spíchnul" za hodinku, ale bylo to uděláno tak trošku na hulváta. Tak jsem si řekl že to postupně přepíšu tak jak by to třeba napsal nějaký zkušený programátor s Nette frameworkem. Na [http://srazy.info/nettefwpivo](http://srazy.info/nettefwpivo) jsem danou věc přednesl a Nette guruové mi přislíbili odbornější konzultace, tak doufám že se nám podaří vytvořit návod jak by se takové věci nad Nette mohli psát.

celý projekt je pak k naleznutí na Githubu [https://github.com/chemix/Nette-Facebook-Reader](https://github.com/chemix/Nette-Facebook-Reader)

Pokud máte nápad jak danou ukázku vylepšit nebo článek upravit, pošlete pull request, nebo mi napište email.

Článek je psán stylem od nuly k prototypu a nasledné zlepšování. Ne všechen kód je zde uváděn, ale každý kus kódu je zakončen odkazem na daný commit na githubu. 


Vytvoření projektu z Nette/Sandbox
----------------------------------

Začneme čistým projektem vycházející z Nette/Sandbox

`composer create-project nette/sandbox Nette-Facebook-Reader`

`cd Nette-Facebook-Reader`

zápis do "log" a "temp" folderu

`chmod -R a+rw temp log`

commit: [Init from Nette Sandbox](https://github.com/chemix/Nette-Facebook-Reader/commit/dafae017f01730c79da1ede8b2ba6c295ac79f61)

Pročištění pískoviště
---------------------

vyčistíme homepage šablonu a připravíme si nový Import Presenter se šablonou.

/app/templates/Import/default.latte

{% highlight html %}
{block #content}
	<h1>Import</h1>
{% endhighlight %}

a /app/presenters/ImportPresenter.php

{% highlight php startinline %}
namespace App\Presenters;

use Nette,
	App\Model;

class ImportPresenter extends BasePresenter
{

	public function renderDefault()
	{

	}

}
{% endhighlight %}

commit: [clear homepage template and presenter](https://github.com/chemix/Nette-Facebook-Reader/commit/a420a7af22e6ecba6d6d7d5c65e5fae44679ce22)

commit: [added blank Import Presenter](https://github.com/chemix/Nette-Facebook-Reader/commit/668308c23c3e2334b7ee07062228183f80938161)


Přidáme trošku Facebooku
------------------------

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

{% highlight php startinline %}
use Facebook\FacebookRequest;
use Facebook\FacebookRequestException;
use Facebook\FacebookSession;
use Tracy\Dumper;
{% endhighlight %}

a metodu přepíšeme podle ukázky z dokumentace k PHP Facebook SDK

{% highlight php startinline %}
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
{% endhighlight %}

po zkouknutí výstupu bychom měli vidět dump pole o 25 položkách

![dump array](/image/nette-facebook-reader/dump-array-25.png)

Malinko si zjednodušíme ošetření chyb jen na ukončení aplikace.

{% highlight php startinline %}
} catch (\Exception $ex) {
	throw $ex;
	$this->terminate();
}
{% endhighlight %}

commit: [add dependence on Facebook SDK to composer and small typo updates](https://github.com/chemix/Nette-Facebook-Reader/commit/09420de69abc424fe3b06a6a201dd23597465967)

commit: [load data from Facebook - dirty version](https://github.com/chemix/Nette-Facebook-Reader/commit/f7abe600372f3e77cb7486dbb393bb85c300e7ed)

commit: [only one catch and application->terminate()](https://github.com/chemix/Nette-Facebook-Reader/commit/17cf92295d9b10da4502d34ea870933ac499983e)




Cache, ať při vývoji nečekáme
-----------------------------

Přidáme si možnost cache pro požadavek. (Hlavně si tím urychlíme další rozšiřování, přeci jen čekat 10 sec na každý refresh mě nebaví).

O cachovaní se nám postara Nette Framework a jeho Nette\Caching\Cache;

Přidáme do use sekce

{% highlight php startinline %}
use Nette\Caching\Cache;
{% endhighlight %}

Abychom mohli vytvořit instanci třídy Cache tak jí musíme předat nějaký cache storage kam si bude ukládat data. Viz API dokumentace [Nette\Caching\Cache](http://api.nette.org/2.2.1/Nette.Caching.Cache.html#___construct)

A ten si necháme poslat (injectnout) do třídy pomoci Nette Dependenci Injection. Jediné co musíme udělat je definovat public property $cacheStorage typu \Nette\Caching\IStorage a pomocí anotace @inject nám framework zařídí vše potřebné.

{% highlight php startinline %}
class ImportPresenter extends BasePresenter
{
	/**
	 * @var \Nette\Caching\IStorage @inject
	 */
	public $cacheStorage;
{% endhighlight %}

hup, a v našich metodách se ke storage dostaneme snadno pomocí `$this->cacheStorage`

{% highlight php startinline %}
$cache = new Cache($this->cacheStorage, 'facebookWall');
$data = $cache->load("stories");
{% endhighlight %}

Více o cache si nastudujete v dokumentaci [http://doc.nette.org/cs/2.1/caching]( Cache).

V našem případě pokud se nám nepodaří načíst data z cache (a to se nám napoprvé určitě nepovede) tak si je načteme z Facebooku a do té cache si je uložíme:

{% highlight php startinline %}
$cache->save("stories", $data, array(
	Cache::EXPIRATION => '+30 minutes',
	Cache::SLIDING => TRUE
));
{% endhighlight %}

Výsledkem je

{% highlight php startinline %}
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
{% endhighlight %}

Nyní by nám druhý request měl trvat výrazně kratší dobu.

![with cache](/image/nette-facebook-reader/with-cache.png)

commit: [added cache for request](https://github.com/chemix/Nette-Facebook-Reader/commit/e471528d4265d6ee9487a0fbb0401a59a04a3d4a)


Ukládáme posty do databáze
---------------------------

Dalším krokem je uložit si získána data do databáze. Pro práci s databází použijeme třídu Nette\Database. Vytvoříme si databázi a uživatele (díky klonování Nette Sandbox máme výborný nástroj Adminer přímo u projektu /adminer/).

Uživatel bude facebookwall a s heslem 'tajneheslo' bude mít přístup ke všem databázím začínající facebookwall_ v našem vývojovém případě konkrétně k databázi facebookwall_devel

{% highlight sql startinline %}
CREATE DATABASE `facebookwall_devel` COLLATE 'utf8_czech_ci';
CREATE USER 'facebookwall'@'localhost' IDENTIFIED BY 'tajneheslo';
GRANT USAGE ON * . * TO 'facebookwall'@'localhost' IDENTIFIED BY 'tajneheslo' WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0 ;
GRANT ALL PRIVILEGES ON `facebookwall\_%` . * TO 'facebookwall'@'localhost';
FLUSH PRIVILEGES;
{% endhighlight %}

a vytvoříme si tabulku kam si posty budeme ukládat.

{% highlight sql startinline %}
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
{% endhighlight %}

V presenteru si řekneme ať nám framework předá objekt Nette\Database\Context

{% highlight php startinline %}
/**
 * @var \Nette\Database\Context @inject
 */
public $database;
{% endhighlight %}

v konfiguračním souboru /app/config/config.local.neon si doplňíme připojení k databázi.

```
nette:
	database:
		dsn: 'mysql:host=127.0.0.1;dbname=facebookwall_devel'
		user: facebookwall
		password: tajneheslo
		options:
			lazy: yes
```

Pozor na zápis, liší se od Dibi a občas mě to dokáže zabrzdit ;-)


V presenteru si pak na hulváta doplníme ukládání jednotlivých řádku, připadne po opakovaném importu aktualizaci postit

{% highlight php startinline %}
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
{% endhighlight %}

můžeme se přesvědčit, že se nám vše uložilo

![save posts to database](/image/nette-facebook-reader/save-to-database.png)

commit: [save posts to database](https://github.com/chemix/Nette-Facebook-Reader/commit/2550d2d70c36720526f88bea448864cffda592df)


Zobrazení postů z importu 
-------------------------

Předáme si výpis práve přidaných postů do šablony a tam si je vypíšeme.

{% highlight php startinline %}
// send data to template
$this->template->wallPosts = $data;
{% endhighlight %}

a

{% highlight smarty %}
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
{% endhighlight %}

commit: [show post in template](https://github.com/chemix/Nette-Facebook-Reader/commit/7e7ad1456b6a6767fb543dacf5404738e9e40c69)


Zobrazení postů na homepage
---------------------------

V HomepagePresenteru si načteme posty co jsme načetly importem. Jelikož, ale nechcem zobrazovat všechny posty nastavíme si u některých v databázi status na 1 a budem zobrazovat pouze tyto.

{% highlight php startinline %}
public function renderDefault()
{
	$facebookWallPosts = $this->database->table('facebook_wallposts')->where('status','1')->limit(5)->fetchAll();
	$this->template->wallPosts = $facebookWallPosts;
}
{% endhighlight %}

a nesmíme zapomenout na přidání public property $database;

{% highlight php startinline %}
/**
 * @var \Nette\Database\Context @inject
 */
public $database;
{% endhighlight %}

šablona pak může vypadat nějak takto:

{% highlight smarty %}
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

commit: [show posts on homepage](https://github.com/chemix/Nette-Facebook-Reader/commit/7fed0d0a70ac7472a1b70d696f18fae4e2993e52)

tuto verzi najdete pod tagem [:prototype](https://github.com/chemix/Nette-Facebook-Reader/tree/prototype)

Zapouzdření do modelu
---------------------

Pokud se nad úkolem zamyslíme tak je to taková věc co by se nám mohla hodit i na jiném projektu. Připravíme si tedy modelovou vrstvu. Do ktere přepíšeme náš prototyp.

Všiměte si jak si v konstruktoru řekneme o Nette\Database\Context

app/model/FacebookWallpost.php

{% highlight php startinline %}
namespace App\Model;

use Nette\Database\Context;
use Nette\Object;

class FacebookWallposts extends Object
{
	/**
	 * @var \Nette\Database\Context
	 */
	protected $database;

	function __construct(Context $database)
	{
		$this->database = $database;
	}

	public function getLastPosts($count = 5)
	{
		return $this->database->table('facebook_wallposts')
			->where('status', '1')
			->order('created_time DESC')
			->limit($count)
			->fetchAll();
	}
}
{% endhighlight %}

a v presenteru přepíšeme vypisování postů na

{% highlight php startinline %}
/**
 * @var \App\Model\FacebookWallposts @inject
 */
public $wallposts;

public function renderDefault()
{
	$this->template->wallPosts = $this->wallposts->getLastPosts();
}
{% endhighlight %}

Property $database jsme nahradili za $wallpost a změnili typ třídy co chceme po frameworku aby nám předal. Aby to celé fungovalo musíme ješte danou servisu zaregitrovat v config.neon

```
services:
	- App\Model\UserManager
	- App\RouterFactory
	router: @App\RouterFactory::createRouter
	- App\Model\FacebookWallposts
```

commit: [model - section for read from db](https://github.com/chemix/Nette-Facebook-Reader/commit/fa96deed64d74de7ff0da782e2eb1d1e4080f9c4)

Import do modelu
----------------

To samé uděláme i s částí pro načítání dat z Facebooku.

Při přesunu odstraním používání cache, jelikož už jí při vývoji nepotřebuji ba naopak pokud chci zadat import tak chci aby se provedl vždy.

{% highlight php startinline %}
public function importPostFromFacebook()
{
	FacebookSession::setDefaultApplication('YOUR_APP_ID', 'YOUR_APP_SECRET');
	$session = FacebookSession::newAppSession();

	try {
		$request = new FacebookRequest($session, 'GET', '/nettefw/feed');
		$response = $request->execute();
		$posts = $response->getGraphObject()->asArray();
		$data = $posts['data'];

	} catch (\Exception $ex) {
		throw $ex;
		$this->terminate();
	}

	// save data to database
	...
{% endhighlight %}

z Import presenteru přemístíme use sekci do modelu a presenter se nám rázem zjednodušil na

{% highlight php startinline %}
class ImportPresenter extends BasePresenter
{
	/**
	 * @var \App\Model\FacebookWallposts @inject
	 */
	public $wallposts;

	public function renderDefault()
	{
		$this->template->wallPosts = $this->wallposts->importPostFromFacebook();
	}

}
{% endhighlight %}

commit: [model - section for import data from Facebook](https://github.com/chemix/Nette-Facebook-Reader/commit/dce5d9bc22449ea094ac12cfdc880c5718444cc4)

A co to heslo v kódu? Pryč s nim
--------------------------------

Jako pěkný, ale. Ale nám se ještě nelíbí

{% highlight php startinline %}
FacebookSession::setDefaultApplication('YOUR_APP_ID', 'YOUR_APP_SECRET');
$session = FacebookSession::newAppSession();
{% endhighlight %}

hesla chceme v konfiguraci a zde si jen řekneme o funkční session. Nahradíme tedy za

{% highlight php startinline %}
$session = $this->facebookSessionManager->getAppSession();
{% endhighlight %}

a do konstruktoru přidáme předání závislosti. plus nezapomene na deklarovaní property.

{% highlight php startinline %}
/**
 * @var \App\Model\FacebookSessionManager
 */
protected $facebookSessionManager;

/**
 * @param Context $database
 * @param FacebookSessionManager $facebook
 */
function __construct(Context $database, FacebookSessionManager $facebook)
{
	$this->database = $database;
	$this->facebookSessionManager = $facebook;
}
{% endhighlight %}

a našeho "hloupoučkého" managera definujeme v /app/model/FacebookSessionManager.php

{% highlight php startinline %}
namespace App\Model;

use Facebook\FacebookSession;

class FacebookSessionManager {

	protected $appId;

	protected $appSecret;

	function __construct($appId, $appSecret)
	{
		$this->appId = $appId;
		$this->appSecret = $appSecret;

		FacebookSession::setDefaultApplication($this->appId, $this->appSecret);
	}

	public function getAppSession()
	{
		return FacebookSession::newAppSession();
	}
}
{% endhighlight %}

registrujeme ho v config.neon

```
services:
	- App\Model\UserManager
	- App\RouterFactory
	router: @App\RouterFactory::createRouter
	- App\Model\FacebookWallposts
	- App\Model\FacebookSessionManager(%facebook.appId%, %facebook.appSecret%)
```

a v config.local.neon přidáme sekci s kódem a heslem k aplikaci

```
parameters:
	facebook:
		appId: APP_ID
		appSecret: APP_SECRET
```
commit: [extract session generator to new service FacebookSessionManager](https://github.com/chemix/Nette-Facebook-Reader/commit/1f6302dff1a140b2f39898560ad7ac8f170933bf)

tuto verzi najdete pod tagem [:model-di](https://github.com/chemix/Nette-Facebook-Reader/tree/model-di)

Korektury od Nette Guru 
-----------------------
Jako první poslal pull request [Filip Prochazka](http://filip-prochazka.com/). 

commit: [removed useless folder](https://github.com/chemix/Nette-Facebook-Reader/commit/c4a3db274328eec1e99e050d4c40b37945d6a29a) odebírá zbytečné kontrolování složky kterou nepoužíváme

commit: [Refactored default configuration for database](https://github.com/chemix/Nette-Facebook-Reader/commit/ee682fd66f4303ce1ac591cdb57b17fbfbe775b7) Upravuje jak se zapisuje přihlašování k databázi. Nyní jsou parametry (jméno, heslo, server, databáze) vytáhnuté do sekce "parameters". Proto si ze souboru config.local.neon odeberte sekci nette - database a nahraďte ji 

```
parameters:
	database:
		dbname: facebookwall_devel
		user: facebookwall
		password: tajneheslo
```

mnohem čitelnější. 

commit: [export db schema to project](https://github.com/chemix/Nette-Facebook-Reader/commit/743a911744625e62aa1eb4eace190f35eb4bf8b8) přidává zmíněný SQL table creator do kódu

teď ale příjde ta zajímavější část. 

Použití Kdyby/Facebook
----------------------

Jako první nahradíme v *composer.json* Facebook/SDK za [Kdyby/Facebook](https://github.com/Kdyby/Facebook)

```
	"require": {
		"php": ">= 5.3.7",
		"nette/nette": "~2.2.0",
		"dg/adminer-custom": "~1.0",
		"kdyby/facebook" : "dev-master"
	},
```

a aktualizujeme composer pomocí `composer update`. Smažeme náš "hloupoučký" *FacebookSessionManager.php* a odebereme i jeho registraci do services v *config.neon*, zde naopak přidáme sekci extensions a do ní registrujeme Kdyby Facebook

```
extensions:
	facebook: Kdyby\Facebook\DI\FacebookExtension

services:
	- App\Model\UserManager
	- App\RouterFactory
	router: @App\RouterFactory::createRouter
	- App\Model\FacebookWallposts
```

Tato extension vyžaduje v configu Facebook App ID a Facebook Secret. Proto do lokálního *config.local.neon* přemístíme tyto informace ze sekce params (kam jsme si je uložili) do sekce facebook.

```
facebook:
	appId: "APP_ID"
	appSecret: "APP_SECRET"
```

(APP_ID dejte do uvozovek, jinak je brán jako integer a Kdyby/Facebook vyhodí excaption.)

A ve FacebookWallpost.php uděláme pár změn. Prvně si upravíme use sekci. Odebereme Facebook SDK a nahradíme ho za Kdyby\Facebook a přidáme Tracy\Debugger.

{% highlight php startinline %}
use Kdyby\Facebook\Facebook;
use Kdyby\Facebook\FacebookApiException;
use Nette\Database\Context;
use Nette\Object;
use Tracy\Debugger;
{% endhighlight %}

Pak nahradíme inicializaci session managera za Kdyby\Facebook

{% highlight php startinline %}
/**
 * @var Facebook
 */
protected $facebook;

/**
 * @param Context $database
 * @param Facebook $facebook
 */
function __construct(Context $database, Facebook $facebook)
{
	$this->database = $database;
	$this->facebook = $facebook;
}
{% endhighlight %}

a zjednoduší se nám i volání samotného požadavku. Plus si budeme logovat případnou chybu. 

{% highlight php startinline %}
public function importPostFromFacebook()
{
	try {
		$posts = $this->facebook->api('/nettefw/feed');
		$data = $posts['data'];

	} catch (\Exception $ex) {
		Debugger::log($ex->getMessage(), 'facebook');
		return array();
	}

	// save data to database
    ...
{% endhighlight %}

> Q: Proč `return array();` namísto `$this->terminate();`
> 
> A: ???

Teď, když si znovu spustíme import, tak bychom měli v Tracy vidět novou ikonku Facebooku a u ní základní informace o volání jeho API. Krása.

![Kdyby Facebook - Tracy extension](/image/nette-facebook-reader/kdyby-facebook-tracy.png)

Další vychytávkou co Kdyby\Facebook má je metoda iterate. Pokud jste si všimli tak volání Facebook API vrací cca 25 záznamů a adresu pro další (paging) toho tato metoda využívá umožnuje nad výsledkem iterovat třeba ve foreach a donačíst tak úplně všechny posty.

Nahradíme tedy volání *api* za *iterate*. Zde už dostáváme čisté "pole" všech postů tak poupravíme i samotné procházení výsledků.

{% highlight php startinline %}
try {
	$posts = $this->facebook->iterate('/nettefw/feed');

} catch (\Exception $ex) {
	Debugger::log($ex->getMessage(), 'facebook');
	return array();
}

$imported = array();
// save data to database
foreach ($posts as $rowPost) {
  ...
}
return $imported;
{% endhighlight %}

Když teď, zkusíme import, v Tracy panelu uvidíme že se Facebook Api volalo vícekrát a v panelu je vidět detail každého volání.

Filip pak přepsal mé ifové peklíčko do mnohem čitelnější podoby pomocí [ternárního operátora "?:"](http://php.vrana.cz/ternarni-operator.php). Nahradil datum ve stringu za DateTime obalené v [Nette\Utils\DateTime](http://api.nette.org/2.2.1/Nette.Utils.DateTime.html) a vrácený záznam je přetypován na [ArrayHash](http://api.nette.org/2.2.1/Nette.Utils.ArrayHash.html) (nezapomenout definovat v use)

{% highlight php startinline %}
use Nette\Utils\DateTime;
use Nette\Utils\ArrayHash;
{% endhighlight %}

a pak ve foreach

{% highlight php startinline %}
$post = array(
	'id' => $rowPost->id,
	'type' => $rowPost->type,
	'created_time' => DateTime::from($rowPost->created_time)->format('Y-m-d H:i:s'),
	'updated_time' => DateTime::from($rowPost->updated_time)->format('Y-m-d H:i:s'),
	'name' => isset($rowPost->name) ? $rowPost->name : NULL,
	'link' => isset($rowPost->link) ? $rowPost->link : NULL,
	'status_type' => isset($rowPost->status_type) ? $rowPost->status_type : NULL,
	'message' => isset($rowPost->message) ? $rowPost->message : NULL,
	'caption' => isset($rowPost->caption) ? $rowPost->caption : NULL,
	'picture' => isset($rowPost->picture) ? $rowPost->picture : NULL,
);

// post 'status' use story, we need message
if ($rowPost->type == 'status' && isset($rowPost->story)) {
	$post['message'] = $rowPost->story;
}
// add to return array 
$imported[$post['id']] = ArrayHash::from($rowPost);
{% endhighlight %}

Latte filter
-------------------

Filip do kódu přidal i ukázku jak se poprat s if peklem v šablonách pomocí Latte filtru.

V *HomepagePresernter* definujeme metodu [createTemplate](https://github.com/chemix/Nette-Facebook-Reader/blob/5dd7bed8fb30eb22284ee2e0fc92a726db0913fd/app/presenters/HomepagePresenter.php#L27), která vrací presenteru template object, který se použije pro render šablon. K této šabloňe přidáme filtr, který se bude starat o odkazy na posty.

{% highlight php startinline %}
protected function createTemplate()
{
	/** @var Nette\Bridges\ApplicationLatte\Template $template */
	$template = parent::createTemplate();
	$template->addFilter('fbPostLink', function ($fbPost) {
		if (!empty($fbPost->link)) {
			return $fbPost->link;
		}

		if ($m = Nette\Utils\Strings::match($fbPost->id, '~^(?P<pageId>[^_]+)_(?P<postId>[^_]+)\\z~')) {
			return 'https://www.facebook.com/nettefw/posts/' . urlencode($m['postId']);
		}

		return NULL;
	});

	return $template;
}
{% endhighlight %}

* filter se jmenuje fbPostLink
* pokud daný post má definovaný link vrací tento link
* pokud se podaří z post id (které obsahuje {pageId}\_{postId} ) získat postId vrátí link na konkrétní post na facebooku
* voláme ho nad objektem $post

šablona se nám pak zjednoduší na 

{% highlight smarty %}
<div n:foreach="$wallPosts as $post" class="post {$post->type}">
	<h3 n:if="$post->name">{$post->name}</h3>
	<img n:if="$post->picture" src="{$post->picture}" />
	<p>
		{if $post->message}{$post->message|truncate:250:'...'}{/if}
		<a href="{$post|fbPostLink}">více</a>
	</p>
</div>
{% endhighlight %}


commit: [Use kdyby/facebook](https://github.com/chemix/Nette-Facebook-Reader/commit/5dd7bed8fb30eb22284ee2e0fc92a726db0913fd)


tuto verzi najdete pod tagem [:kdyby](https://github.com/chemix/Nette-Facebook-Reader/tree/kdyby)

Drobné vylepšováky
--------------------

Na doporučení Davida jsem odebral z projektového *.gitignore* soubory Sublime editoru a PHPStormu a zapsal jsem si je do globálního gitignore [podle návodu na githubu](https://help.github.com/articles/ignoring-files#create-a-global-gitignore)

Administrace postů
------------------

Jako poslední úkol jsem si nechal administraci na povolování postů. Vytvoříme si nový presenter *AdminPresenter* a u vypsaných jednotlivých položek přidáme tlačítko enable, disable. To celé pak z "AJAXujem". 

Než začnem s php úpravama, provedem pár drobných změn na frontendu ať se na náš výtvor dá aspoň trošku koukat. Osobně mám rád [Zurb Foundation](http://foundation.zurb.com), ale tu samou práci, pro tento případ i možná vhodnější, odvede [Bootstrap](http://getbootstrap.com)

commit: [added zurb foundation 5 and updated templates](https://github.com/chemix/Nette-Facebook-Reader/commit/0e0d4494bf3cd8d853ecaea380381fa78716d444)

commit: [tabs indent](https://github.com/chemix/Nette-Facebook-Reader/commit/0cabe53852463d83ebbb45226ec811438ae5a967)

Výkop administrace
------------------

Začneme úpravou modelu, kam si přidáme metodu co nám vrátí všechny posty.

{% highlight php startinline %}
public function getAllPosts()
{
	return $this->database->table('facebook_wallposts')
		->order('created_time DESC')
		->fetchAll();
}
{% endhighlight %}

a následně si je načteme presenterem *AdminPresenter* a pošleme do šablony

{% highlight php startinline %}
/**
 * Admin presenter.
 */
class AdminPresenter extends BasePresenter
{

	/**
	 * @var \App\Model\FacebookWallposts @inject
	 */
	public $wallposts;

	public function renderDefault()
	{
		$this->template->wallPosts = $this->wallposts->getAllPosts();
	}

}
{% endhighlight %}

šablonu je možno vidět v commitu.

commit: [basics for admin init](https://github.com/chemix/Nette-Facebook-Reader/commit/66ac9176ea8ae548d8e5c0b658d9e655a811613f)

Akce disable, enable
--------------------

Následuje vytvoření v modelu metod co se nám postarají o samotnou editaci postu.

{% highlight php startinline %}
/**
 * enable post
 *
 * @param $postId string
 * @return bool
 */
public function enablePost($postId)
{
	$this->database->table('facebook_wallposts')
		->where('id', $postId)
		->update(array('status' => '1'));
	return TRUE;
}
/**
 * disable post
 *
 * @param $postId string
 * @return bool
 */
public function disablePost($postId)
{
	$this->database->table('facebook_wallposts')
		->where('id', $postId)
		->update(array('status' => '0'));
	return TRUE;
}
{% endhighlight %}

a v presenteru si vytvořím dvě akce co funkcionalitu budou obsluhovat 

{% highlight php startinline %}
public function actionEnablePost($postId)
{
	if ($this->wallposts->enablePost($postId)){
		$this->flashMessage('Post enabled');
	}
	$this->redirect('default');
}
public function actionDisablePost($postId)
{
	if ($this->wallposts->disablePost($postId)){
		$this->flashMessage('Post disabled');
	}
	$this->redirect('default');
}
{% endhighlight %}

v šabloně si upravím odkazy ať fungují

{% highlight smarty %}
{if $post->status == '1'}
	<li><a n:href="Admin:disablePost $post->id" class="button alert">disable</a></li>
{else}
	<li><a n:href="Admin:enablePost $post->id" class="button">enable</a></li>
{/if}
{% endhighlight %}

a fungujeme. 

commit: [enable and disable post](https://github.com/chemix/Nette-Facebook-Reader/commit/2c92ac3844a94fd671d339b3ee8d396969a63031)

První verze zajaxovní
---------------------

není to žádná hitparáda, ale funguje, a to se počítá ;-) Začnem úpravou presenteru. Ten pokud se bude jednat o ajaxový požadavek, tak na místo flashMessage nastavíme do proměnné *payload* message co se stalo, a následně pošleme uživateli tento payload. Metoda sendPayload je ulehčení ať se nemusíme starat o posílání JSON Response, vše je čitélne z obsahu [metody sendPayload](http://api.nette.org/2.2.2/source-Application.UI.Presenter.php.html#603) 

{% highlight php startinline %}
public function actionEnablePost($postId)
{
	if ($this->wallposts->enablePost($postId)){
		$this->flashMessage('Post enabled');
	if ($this->wallposts->enablePost($postId)) {
		if ($this->isAjax()) {
			$this->payload->message = 'Post enabled';
			$this->sendPayload();
		} else {
			$this->flashMessage('Post enabled');
			$this->redirect('default');
		}
	}
	$this->redirect('default');
}
{% endhighlight %}

to samé uděláme i pro druhou metoru *actionDisablePost*. Upravíme si šablonu tak že budem zobrazovat obě tlačítka a jen skrze css budem schovávat to, které zrovna nebudem potřebovat.

{% highlight smarty %}
<li n:class="disable, !$post->status ? hide"><a n:href="Admin:disablePost $post->id" class="ajax button alert">disable</a></li>
<li n:class="enable, $post->status ? hide"><a n:href="Admin:enablePost $post->id" class="ajax button">enable</a></li>
{% endhighlight %}

a pak celé to rozhejbání v JavaScriptu. Pokud kliknem na odkaz co má třídu *.ajax* tak stopnem klasické volání, a zavoláme XHR požadavek. Pokud se nám vrátí message, že je post disablován, tak prohodíme zobrazení tlačítek. (v opačném případě také) Plus pár visuálních drobností (disablování butonu po kliknutí, změna kurzoru na hodinky)

{% highlight javascript %}
// Ajax click in admin
$('body').on('click', 'a.ajax', function (event) {
	event.preventDefault();
	event.stopImmediatePropagation();
	var link = $(this);
	if (link.hasClass('disabled')) {
		return false;
	}
	link.css('cursor', 'wait');
	link.addClass('disabled');
	$.post(this.href, function (data) {
		if (data.message == 'Post disabled') {
			link.parent().parent().find('.disable').addClass('hide');
			link.parent().parent().find('.enable').removeClass('hide');
		} else {
			// enabled
			link.parent().parent().find('.disable').removeClass('hide');
			link.parent().parent().find('.enable').addClass('hide');
		}
		link.removeClass('disabled');
		link.css('cursor', 'default');
	});
});
{% endhighlight %}

commit: [ajax version of enable and disable post](https://github.com/chemix/Nette-Facebook-Reader/commit/c9b5a8fce0a63c3e0c3f69e11b0549d860d871db)

Úprava JSON komunikace
----------------------

Porovnávat message co se stalo není moc "profi", tak si zavedem nějakou proměnou s akcí a status zdali se provedla správně. To s použitím payload proměnné je vcelku snadné 

{% highlight php startinline %}
if ($this->isAjax()) {
	$this->payload->message = 'Post enabled';
	$this->payload->action = 'enable';
	$this->payload->status = '1';
	$this->sendPayload();
	
} else {
{% endhighlight %}

v JavaScriptu se pak zeptáme co se dělo a jak to dopadlo a zobrazíme dynamicky flash zprávu.

{% highlight javascript %}
var flashMessage = function(message)
{
	$($('body')[0]).prepend($('<div class="flash info">'+message+'</div>'));
}

...

if (payload.action == 'disable' && payload.status == '1') {
	// disabled
	link.parent().parent().find('.disable').addClass('hide');
	link.parent().parent().find('.enable').removeClass('hide');
	flashMessage(payload.message);
}
{% endhighlight %}

commit: [better payload status and javascript behaviour](https://github.com/chemix/Nette-Facebook-Reader/commit/a13d6a0c37667278b0acfcd0a712683fb4afe0a0)

Použij signály než action
-------------------------

Další radou od zkušených bylo použití signálů (handle). 

> Handle je na změnu stavu aktuálního view. Tj na smazání, zaktivnění položky etc. (Většinou totiž po provedení chceš znova vykreslit tu samou stránku). 
>
> Patrik Votoček

nebo 

> Handle je „subsignál“ aktuální akce, je to jako když odešleš formulář. Když máš akci, tak většinou by měla něco zobrazovat, nebo připravovat data pro formulář. Zpracování formuláře taky nedáváme do akce, ale napíšeme na to metodu, kterou dáš formuláři jako callback. Tak přesně to je signál, zpracování nějaké operace (třeba smazání řádků, nebo označení řádku jako hidden) pro aktuální akci (což je třeba výpis jednotlivých řádků).
>
> Filip Procházka

Přepracování bylo snadné. Přejmenoval jsem metody z *actionDisablePost* na *handleEnablePost* a volání z 

{% highlight php startinline %}
<a n:href="Admin:enablePost $post->id" class="ajax button">enable</a>
{% endhighlight %}

na vykřičníkový signál

{% highlight php startinline %}
<a n:href="enablePost! $post->id" class="ajax button">enable</a>
{% endhighlight %}

#### TIP: piš méně 
> pri odkazovani na akci ve stejnem presenteru staci uvest nazev akce, nemusis jiz uvadet presenter
>
> Matej21

commit: [use signals for enable/disable posts](https://github.com/chemix/Nette-Facebook-Reader/commit/24c0c4154122f8a35f283a4fdf16b083ef51d264)

Snippety a nette.ajax.js
------------------------

Teď přichází pořádné kladivo. Představme si že nechceme ručně ošetřovat ajaxové volání. Prostě ať se udělá co se udělat má a změní se jen potřebné. K tomu slouží [Snippety](http://doc.nette.org/cs/2.1/ajax). Snippet chápu jako pojmenovaný prvek na stránce, který v případě potřeby je možné nahradit za jeho aktuální verzi. V našem případě si pro začátek označíme dva snippety. Prvním bude blok kódu co se nám stará o výpis flash messsages

{% highlight smarty %}
{snippet flashes}
	<div n:foreach="$flashes as $flash" class="flash {$flash->type}">{$flash->message}</div>
{/snippet}
{% endhighlight %}

druhým bude tabulka wallpostů

{% highlight smarty %}
{snippet wallposts}
	{foreach $wallPosts as $post}
		<div class="row">
		...
	{/foreach}
{/snippet}
{% endhighlight %}

v tuto chvíli se stali tzv. controllem který v případě, že víme že se změnil tak ho necháme překreslit *redrawControl*. V našem případě pokud chceme změnit status postu tak necháme překreslit snippet *flashes* a *wallposts*

{% highlight php startinline %}
public function handleEnablePost($postId)
{
	if ($this->wallposts->enablePost($postId)) {
		$this->flashMessage('Post enabled');
		$this->redrawControl('flashes');
		$this->redrawControl('wallposts');
	}
}
{% endhighlight %}

kód se nám dosti zjednodušil. Ještě dáme pryč celý náš JavaScript mechanismus co se staral o zpracování požadavku a použijeme knihovnu [nette.ajax.js od Vojty Dobeše](http://addons.nette.org/vojtech-dobes/nette-ajax-js), která umí pracovat automaticky právě se snippety a s jejich přenosovým "JSON protokolem"

jediné co potřebujeme je zavolat její inicializaci. 

{% highlight javascript %}
$(function () {
    $.nette.init();
});
{% endhighlight %}

pěkné zjednodušení, že? Kabelama se nám přenáší jen co se "opravdu" změnilo

![json snippets response](/image/nette-facebook-reader/snippets-response.png)

commit: [use snippets and nette.ajax.js](https://github.com/chemix/Nette-Facebook-Reader/commit/151202a775d65adeccb6a8a991fc759545d935c7)


Bacha na F5
-----------

Zpracování formulářu v Nette funguje na bázi signálu (handle) a tam abychom se vyhnuli problému s refreshem použijeme *redirect()*. Stejně je tomu i v našem případě se signály na disablování a enablování postu. Pokud se nejedná o ajaxový požadavek, tak přesměrujem.

{% highlight php startinline %}
// F5 protection without JS
if (!$this->isAjax()){
	$this->redirect('this');
}
{% endhighlight %}

commit: [redirect after handle signal without JS](https://github.com/chemix/Nette-Facebook-Reader/commit/dfbb0431ab2ff2225f8fba55ae79ee0ece7fabf2)


Posílání opravdu jen toho co je třeba
-------------------------------------

Ajaxové požadavky sviští o 106 jen se nám v každém requestu ajaxem posílá celá tabulka postů. Ale my změnili jen jeden, co kdyby se tedy posílal jen tenhle jeden spolu s flash message? Lze. Technika se nazývá [dynamické snippety](http://doc.nette.org/cs/2.1/ajax#toc-dynamicke-snippety)

každý řádek zabalíme do jednoznačne identifikovatelného snippetu (použijeme n makro)

{% highlight smarty %}
{snippet wallposts}
	{foreach $wallPosts as $post}
		<div class="row" n:snippet="item-$post->id">
{% endhighlight %}

a přidáme trochu logiky do handle. V případě že se jedná o ajaxový požadavek, načteme jen aktuálně zpracovávaný řádek a do šablony ho pošleme jako "seznam všech postů", v normálním požadavku pošleme do šablony posty všechny.

{% highlight php startinline %}
public function handleEnablePost($postId)
{
	if ($this->wallposts->enablePost($postId)) {
		$this->template->wallPosts = $this->isAjax()
			? array($this->wallposts->getOne($postId))
			: $this->wallposts->getAllPosts();
		$this->flashMessage('Post enabled');
		$this->redrawControl('flashes');
		$this->redrawControl('wallposts');
{% endhighlight %}

jelikož se handle zpracovává dříve než render viz [životní cyklus presenteru](http://doc.nette.org/cs/2.1/presenters#toc-zivotni-cyklus-presenteru), tak pokud uživatel bez JavaScriptu změnil viditelnost postu, tak už do šablony poslal seznam všech postů a render už tuto věc dělat nemusí, tak si to ošetříme.

{% highlight php startinline %}
public function renderDefault()
{
	if (!isset($this->template->wallPosts)) {
		$this->template->wallPosts = $this->wallposts->getAllPosts();
	}
}
{% endhighlight %}

commit: [Add method getOne to Model\FacebookWallposts](https://github.com/chemix/Nette-Facebook-Reader/commit/229592e47887ba5160f627041a9f2698ddaa3e08)

commit: [use dynamic snippets](https://github.com/chemix/Nette-Facebook-Reader/commit/d4e73defeb07033c4da59f5c3ee4892abbeb63e6)

Zničíme duplicitní kód
----------------------

Metody *handleEnablePost* a *handleDisablePost($postId)* mají dost kódu úplně stejného. Proto mě napadlo že bych je nějak předělal.

**První** nápad byl mít metodu *handleChangePostStatus($postId, $actionType)*, kde by jako druhý parametr byl typ akce, disable nebo enable. Dva parametry se mi nakonec nelíbily.

#### TIP: rezervovaná slova
zde jsem původně měl parametr pojmenová pouze *$action* a ouhle nějak to nefungovalo. Narazil jsem na pojem rezervovaných proměnných. Tak bacha na ně ;-) Proto i submit button ve formuláři by neměl mít jméno *action*. Další slova jsou: *$do*, *$_fid*, (TODO) .. a *$action*

**Druhým** nápadem bylo mít metodu *handleTogglePostStatus($postId)*, která by si zjistila zda je článek povolen a zakázala by ho nebo opačně. Zjistil jsem, že by status ani zjištovat nemusela jen by SQL update otočil hodnotu (nezkoušel jsem). Toto řešení jsem zavrhl kvůli zobrazení ve dvou oknech současně. Chování by mohlo být nelogické.

**Třetím** nápadem bylo vytáhnout společnou logiku do vlastní metody a u něj jsem zůstal.

{% highlight php startinline %}
protected function afterTogglePostStatus($status, $postId, $message)
{
	if ($status) {
		$this->template->wallPosts = $this->isAjax()
			? array($this->wallposts->getOne($postId))
			: $this->wallposts->getAllPosts();

		$this->flashMessage($message);
		$this->redrawControl('flashes');
		$this->redrawControl('wallposts');
	}
	// F5 protection without JS
	if (!$this->isAjax()){
		$this->redirect('this');
	}
}

public function handleEnablePost($postId)
{
	$status = $this->wallposts->enablePost($postId);
	$this->afterTogglePostStatus($status, $postId, 'Post enabled');
}

public function handleDisablePost($postId)
{
	$status = $this->wallposts->disablePost($postId);
	$this->afterTogglePostStatus($status, $postId, 'Post disabled');
}
{% endhighlight %}

commit: [refactor handleDisable and handleEnable](https://github.com/chemix/Nette-Facebook-Reader/commit/8cc54bf7e2898fffa6512ccb89be3ebd12ab722a)

Drobnosti
---------

Dobré je mít v repozitáři šablonu pro *config.local.neon*

commit: [add config.local.neon template](https://github.com/chemix/Nette-Facebook-Reader/commit/5fa51fbedc03677f20357282c6106a9a21c6f65f)


Chtěl jsem oku lahodící výpis postů na homepage

commit: [better homepage render with masonry plugin](https://github.com/chemix/Nette-Facebook-Reader/commit/d379b0d77c81c9718eadd32fc10b5da8c0181d98)

A pomocí CSS animované zmizení flash message

commit: [css autohide flash messages via](https://github.com/chemix/Nette-Facebook-Reader/commit/c7e45f59d17bab2b45d81792ff34a3e953253011)



### Díky
- [Filip Procházka](https://github.com/fprochazka)
- [Jiří Zralý](https://github.com/medhi)
- [David Matějka](https://github.com/matej21)
- [Patrik Votoček](https://github.com/Vrtak-CZ)