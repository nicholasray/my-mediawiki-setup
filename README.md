# My Mediawiki Setup

I use [boxwiki](https://github.com/niedzielski/boxwiki) (thanks Stephen!). Most
of the instructions on his page work as is but here are some clarifications if
you are on a Mac like me (his setup is written for linux and some of the
commands won't work):

## Prerequisites

First install
[docker-for-mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac)

You should be able to open up your terminal and have `docker` and
`docker-compose` versions printed out when you run the following command if you
have installed this successfully:

```bash
docker -v 
docker-compose -v
```
Now, let's install more dependencies:

```bash
# MediaWiki development dependencies.
brew install composer

# Download boxwiki to your favorite directory
git clone https://github.com/niedzielski/boxwiki.git
cd boxwiki

# Download Core.
repo_base=https://gerrit.wikimedia.org/r/p/mediawiki
time git clone --recursive "$repo_base/core.git"
cd core

# Download extensions.
time while read i; do git -C extensions clone --recursive "$i"; done << eof
$repo_base/extensions/Echo
$repo_base/extensions/EventLogging
$repo_base/extensions/PageImages
$repo_base/extensions/MobileFrontend
$repo_base/extensions/OAuth
$repo_base/extensions/Popups
$repo_base/extensions/QuickSurveys
$repo_base/extensions/RelatedArticles
$repo_base/extensions/WikimediaEvents
$repo_base/extensions/WikimediaMessages
eof

# Download skins.
time while read i; do git -C skins clone --recursive "$i"; done << eof
$repo_base/skins/MinervaNeue
$repo_base/skins/Vector
eof

# Install PHP dependencies.
time for i in . extensions/*/ skins/*/; do composer -d"$i" install; composer -d"$i" update; done

# Add your LocalSettings.php as LocalSettingsDev.php. See the bottom of this README
# for a copy of my LocalSettingsDev.php

# Symlink the core folder to the `w` directory (needed for friendly urls):
ln -s . w

# Important! Add your .htaccess file to your `core` folder. See the boxwiki
# README for a copy of a .htaccess file that will work.

# Install node and nvm: You can probably skip the following if you just want mediawiki to
# run and are not doing development work on mediawiki that require old versions
# of Node (like MobileFrontend):

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash

# Follow the nvm instructions that are printed during installation. You will
# have to add something like the following to your ~/.bash_profile, ~/.zshrc, or
# other profile of your choice:

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

# Then make sure you source it:
source ~/.bash_profile

# Install NPM dependencies. Note: you may get a prompt asking you to install a
# specific node version like `nvm install 6.11.0`. You should follow that. 
cd .. && . ~/.nvm/nvm.sh && nvm use 

cd core && time for i in . extensions/*/ skins/*/; do npm -C "$i" i; done
```
## Performance Tweaks

Docker was built for linux so the performance of docker-for-mac needs some
tweaking if you don't want your mediawiki instance to be slow as molasses.

I have modified the boxwiki docker-compose.yml (located in the boxwiki folder)
file to setup a memcached server and use docker's [cached
volume](https://docs.docker.com/docker-for-mac/osxfs-caching/#cached). The
latter makes a ton of difference in the page load speed.

Here is how my docker-compose.yml file looks:

```
version: '3'

services:
  boxwiki:
    build:
      context: .
      dockerfile: Dockerfile
    links:
      - mysql
    ports:
      - "8181:80"
    volumes:
      - ./core:/var/www/html:cached
    depends_on:
      - mysql
    networks:
      default:
        aliases:
          - boxwiki.svc
    environment:
      - DB_SERVER=mysql.svc:3306
      - MW_ADMIN_NAME=admin
      - MW_ADMIN_PASS=adminpass
      - MW_WG_SECRET_KEY=secretkey
      - DB_USER=wikiuser
      - DB_PASS=sqlpass
      - DB_NAME=boxwiki
      - MW_SITE_NAME=box
      - MW_SITE_LANG=en
  mysql:
    image: mariadb:10.3
    volumes:
      - ./mysql:/var/lib/mysql
    environment:
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
      MYSQL_DATABASE: 'boxwiki'
      MYSQL_USER: 'wikiuser'
      MYSQL_PASSWORD: 'sqlpass'
    networks:
      default:
        aliases:
          - mysql.svc
  memcached:
    image: memcached
```

## LocalSettingsDev.php Tweaks

If you use my docker-compose file, you will also want to make tweaks to your
LocalSettingsDev.php file (located in boxwiki/core) to take advantage of the
memcached server. Although it needs some cleaning and is always changing, here
is the one I currently use:

```php
<?php

# Protect against web entry
if ( !defined( 'MEDIAWIKI' ) ) {
  exit;
}

$wgMainCacheType = CACHE_MEMCACHED;
$wgParserCacheType = CACHE_MEMCACHED; # optional
$wgMessageCacheType = CACHE_MEMCACHED; # optional
$wgMemCachedServers = array( "memcached:11211" );

$wgDBserver = getenv( 'DB_SERVER' );
$wgDBname = getenv( 'DB_NAME' );
$wgDBuser = getenv( 'DB_USER' );
$wgDBpassword = getenv( 'DB_PASS' );

## Site Settings
$wgShellLocale = "en_US.utf8";
$wgLanguageCode = getenv( 'MW_SITE_LANG' );
$wgSitename = getenv( 'MW_SITE_NAME' );
$wgMetaNamespace = "Project";
# Configured web paths & short URLs
# This allows use of the /wiki/* path
## https://www.mediawiki.org/wiki/Manual:Short_URL
$wgScriptPath = "/w";        // this should already have been configured this way
$wgArticlePath = "/wiki/$1";

#Set Secret
$wgSecretKey = getenv( 'MW_WG_SECRET_KEY' );

## RC Age
# https://www.mediawiki.org/wiki/Manual:$wgRCMaxAge
# Items in the recentchanges table are periodically purged; entries older than
# this many seconds will go. The query service (by default) loads data from
# recent changes Set this to 1 year to avoid any changes being removed from the
# RC table over a shorter period of time.
$wgRCMaxAge = 365 * 24 * 3600;

## Logs
$wgDebugLogGroups = array(
  'resourceloader' => '/var/log/mediawiki/resourceloader.log',
  'exception' => '/var/log/mediawiki/exception.log',
  'error' => '/var/log/mediawiki/error.log'
);
$wgDebugLogFile = "/var/log/mediawiki/debug.log";

$wgDebugToolbar = true;
$wgShowExceptionDetails = true;
$wgShowDBErrorBacktrace = true;
$wgShowSQLErrors = true;

$wgEnableUploads = true;

$wgEventLoggingBaseUri = '/event.gif';
$wgEventLoggingFile = '/var/log/mediawiki/events.log';
wfLoadExtension( 'EventLogging' );

/* wfLoadExtension( 'TextExtracts' ); */
/* wfLoadExtension( 'PageImages' ); */
wfLoadExtension( 'Echo' );

$wgUseInstantCommons = true;
$wgEnableJavaScriptTest = true;

// Useful when testing language variants
$wgUsePigLatinVariant = true;

$wgMFAlwaysUseContentProvider = true;
$wgMFContentProviderClass = 'MobileFrontend\ContentProviders\MwApiContentProvider';
$wgMFMwApiContentProviderBaseUri = 'https://en.wikipedia.org/w/api.php';
$wgMFContentProviderScriptPath = 'https://en.wikipedia.org/w';
$wgMFContentProviderTryLocalContentFirst = true;
$wgMFEnableBeta = true;
$wgMFEnableMobilePreferences = true;
$wgMFLazyLoadImages = [ 'base' => true ];
$wgMFNearby = true;
$wgMFNearbyEndpoint = 'https://en.wikipedia.org/w/api.php';
$wgMFAdvancedMobileContributions = true;

$wgMFEnableWikidataDescriptions = [
  'base' => true
];

$wgMFDisplayWikibaseDescriptions = [
  'search' => true,
  'nearby' => true,
  'watchlist' => true,
  'tagline' => true
];

$wgMinervaPageIssuesNewTreatment = [
  "base" => true
];

/* // Visual Editor */
/* wfLoadExtension( 'VisualEditor' ); */

/* // Enable by default for everybody */
/* $wgDefaultUserOptions['visualeditor-enable'] = 1; */
/* $wgVirtualRestConfig['modules']['parsoid'] = array( */
/* 	// URL to the Parsoid instance */
/* 	// Use port 8142 if you use the Debian package */
/* 	'url' => 'https://en.wikipedia.org', */
/* 	// Parsoid "domain", see below (optional) */
/* 	'domain' => 'en.wikipedia.org', */
/* 	// Parsoid "prefix", see below (optional) */
/* 	'prefix' => 'localhost' */
/* ); */
/* $wgVisusalEditorFullRestbaseURL = 'https://en.wikipedia.org/api/rest_'; */

wfLoadExtension( 'MobileFrontend' );

/* wfLoadExtension( 'OAuth' ); */
/* $wgGroupPermissions['sysop']['mwoauthproposeconsumer'] = true; */
/* $wgGroupPermissions['sysop']['mwoauthmanageconsumer'] = true; */
/* $wgGroupPermissions['sysop']['mwoauthviewprivate'] = true; */
/* $wgGroupPermissions['sysop']['mwoauthupdateownconsumer'] = true; */

$wgPopupsGateway = 'restbaseHTML';
$wgPopupsRestGatewayEndpoint = 'https://en.wikipedia.org/api/rest_v1/page/summary/';
wfLoadExtension( 'Popups' );

// wfLoadExtension( 'QuickSurveys' );

// wfLoadExtension( 'RelatedArticles' );

wfLoadExtension( 'Echo' );

wfLoadExtension( 'WikimediaEvents' );
$wgWMEReadingDepthEnabled = true;
$wgWMEReadingDepthSamplingRate = 1;

$wgMinervaDownloadIcon = true;
$wgMinervaApplyKnownTemplateHacks = true;
$wgMinervaABSamplingRate = 1;
$wgMinervaErrorLogSamplingRate = 1;
wfLoadSkin( 'MinervaNeue' );

wfLoadSkin( 'Vector' );
```

Note that some extensions like VisualEditor are commented out. I only activate
these when necessary as, in general, the more extensions you enable, the slower
your mediawiki will be.

## Execution

### Start

To start just run:

```bash
docker-compose up
```

Note, you will have to wait a bit for docker to boot up and build the image,
etc. When it is ready, you will be able to visit  your wiki homepage at
http://localhost:8181
