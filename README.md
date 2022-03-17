# Demo of issue with Paketo and Composer autoloading

This project demos a problem I'm having with Paketo on a PHP application,
in particular with autoloading of classes outside of the vendor folder.

To demonstrate the problem, I've created a class outside of the vendor
folder (`Application\NonVendorClass`), and am checking if that class
exists. This class should be autoloaded via PSR-4 autoloading, and 
its namespace is defined for autoloading in `composer.json`. 
This file will silently fail to be found under Paketo.

I've also created a class in a different folder, and have told Composer
to load this as a classmap (a way of loading individual classes which
are not in a folder that's pointed to by a namespace known to the 
autoloader). This class is called `ClassMap`, and it will also fail
under Paketo (but will generate log output that may help in diagnostics).

For comparison, I also check if a class exists from the vendor
folder (`Doctrine\Common\Annotations\Annotation`).

## Running outside of Paketo

If I run a local PHP web server and load up this site, I get this as output
in the browser:

```
NonVendorClass exists
ClassMap exists
Annotation exists
```

In other words, PHP is able to access a class outside of the vendor folder,
the file loaded via classmap autoloading, and also a class inside the 
vendor folder.

There is no log output from this activity.

## Running via Paketo

I'm first running:

```
pack build paketo-composer-demo --builder paketobuildpacks/builder:full
```

Then I'm running this to get the container running:

```
docker run --interactive --tty --env PORT=8080 --publish 8080:8080 paketo-composer-demo
```

When I browse to this site, I get this output in the browser:

```
Can't find NonVendorClass
Can't find ClassMap
Annotation exists
```

PHP is unable to access any of the files which are outside of the vendor
folder.

The output from the web server seems to give a clue to the problem,
although only for the file loading via a classmap (the one loading via PSR-4
fails silently):

``` 
[Thu Mar 17 11:08:54 2022] PHP Warning:  
 include(/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/composer/../../classmap/ClassMap.php):
 failed to open stream: No such file or directory in 
 /layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/composer/ClassLoader.php
 on line 571
[Thu Mar 17 11:08:54 2022] PHP Warning:  include(): Failed opening 
 '/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/composer/../../classmap/ClassMap.php'
  for inclusion (include_path='/layers/paketo-buildpacks_php-dist/php/lib/php:/workspace/lib')
  in /layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/composer/ClassLoader.php on line 571
```

I think the issue is that Composer, when trying to autoload files outside of
the vendor folder, use `vendor/composer/../../` expecting to get to the
application's root folder. However, because the vendor folder is actually a
symlink to `/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor`
(rather than vendor actually being a real folder in the root of the application),
when it applies `vendor/composer/../../` to the real location of the vendor
folder, it actually ends up in 
`/layers/paketo-buildpacks_php-composer/php-composer-packages` rather than
in the application root. When it then tries to go down into an application
folder (such as `classmap` or `src` in my application) it can't find them.
