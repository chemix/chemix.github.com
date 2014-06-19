---
layout: post
title: Načítání Facebook postů v Nette Framework
published: true
---

{{ page.title }}
================

<p class="meta">12 Jun 2014 - Prague (Bohemia / Czech Republic)</p>

``` DRAFT ```


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

<div class="commit">
commit: [Init from Nette Sandbox](https://github.com/chemix/Nette-Facebook-Reader/commit/dafae017f01730c79da1ede8b2ba6c295ac79f61)
</div>

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
		password: mojetajneheslo
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

```TODO```

commit: [Use kdyby/facebook](https://github.com/chemix/Nette-Facebook-Reader/commit/5dd7bed8fb30eb22284ee2e0fc92a726db0913fd)




### Díky
- Filip Procházka
- Jiří Zralý



### NEXT STEPS

* schvalovani zobrazeni na strance
* prihlasovani admina?

