Dcycle Drupal Starterkit
=====

[![CircleCI](https://circleci.com/gh/dcycle/starterkit-drupalsite/tree/master.svg?style=svg)](https://circleci.com/gh/dcycle/starterkit-drupalsite/tree/master)

Contents
-----

* About
* Quickstart
* Local self-signed HTTPS quickstart
* Let's Encrypt on a server
* Initial installation on Docker
* Incremental deployment (updating) on Docker
* ./scripts/docker-compose.sh instead of docker-compose
* Environment types
* Local and environment-specific settings
* Power up/power down the Docker environment
* Uninstalling the Docker environment
* Prescribed development process
* Starter data
* Private and public files
* Database schema versions and project versions
* hook_update_N() and configuration
* Patches
* Development design patterns
* Removing modules
* Running automated tests
* Security and maintenance updates
* Theming
* Acquia
* Login links for Acquia environments
* Pantheon (experimental)
* Linting
* Getting a local version of the database
* Logging
* Profiling (or "Why does it take so long?") part 1: PHP
* Profiling (or "Why does it take so long?") part 2: Database
* Logging emails during development
* Troubleshooting
* Running automated functional tests

About
-----

A starterkit to build a Drupal 10 project.

### Features

* **Docker-based**: only Docker is required to build a development environment.
* **One-click install**: a full installation should be available simply by running `./scripts/deploy.sh`.
* **Dependencies and generated files are not included in the repo**: modules, libraries, etc. should not be included in the repo. If you need to generate a directory with all dependencies and generated files, run `./scripts/create-build.sh`.
* **Sass is not used**: to keep the workflow simple, this project does not use SASS; we encourage direct modification of CSS. See the section "No SASS", below.
* **FPM and multi-container**: instead of bundling Apache and Drupal on the same container, we use different containers as described in [PHP and Apache (or Nginx) in separate Docker containers using Docker Compose, March 25, 2022, Dcycle blog](https://blog.dcycle.com/blog/2022-03-25/php-apache-different-containers/).
* **Nginx**: we switch to the Nginx server because Apache was causing [image styles to not generate](https://github.com/docker-library/drupal/issues/215) in a multi-container Docker setups.

### Where to find the code

* The [code lives on GitHub](https://github.com/dcycle/starterkit-drupalsite).
* The [issue queue is on GitHub](https://github.com/dcycle/starterkit-drupalsite/issues).
* If you fork of copy this directory for your own project, enter other environments here (production, stage, secondary git origins).

Quickstart
-----

Step 1: Install [Docker](https://www.docker.com/get-docker) (nothing else required)

Step 2:

    cd ~/Desktop && git clone https://github.com/dcycle/starterkit-drupalsite.git
    cd ~/Desktop/starterkit-drupalsite && ./scripts/deploy.sh

Step 3: Click on the login link at the end of the command line output and enjoy a fully installed Drupal environment.

You can SSH into your container by running:

    ./scripts/ssh.sh

Local self-signed HTTPS quickstart
-----

The following instructions are for local self-signed HTTPS for local development and should not be used on a server. See the section "Let's Encrypt on a server", below, to use HTTPS certificates on a live server.

    cd ~/Desktop/starterkit-drupalsite && ./scripts/https-deploy.sh

See the article [Local development using Docker and HTTPS, Dcycle Blog, Oct. 27, 2018](https://blog.dcycle.com/blog/2018-10-27) for details on how this works.

Let's Encrypt on a server
-----

(This does not apply to local development, only to publicly-accessible servers.)

We will follow the instructions in the following blog posts:

* [Letsencrypt HTTPS for Drupal on Docker, October 03, 2017, Dcycle Blog](https://blog.dcycle.com/blog/170a6078/letsencrypt-drupal-docker/)
* [Deploying Letsencrypt with Docker-Compose, October 06, 2017, Dcycle Blog](https://blog.dcycle.com/blog/7f3ea9e1/letsencrypt-docker-compose/)

Here are the exact steps:

* Figure out the IP address of your server, for example 1.2.3.4.
* Make sure your domain name, for example example.com, resolves to 1.2.3.4. You can test this by running:

    ping example.com

You should see something like:

    PING example.com (1.2.3.4): 56 data bytes
    64 bytes from 1.2.3.4: icmp_seq=0 ttl=46 time=28.269 ms
    64 bytes from 1.2.3.4: icmp_seq=1 ttl=46 time=25.238 ms

Press control-C to get out of the loop.

* Run your instance (./scripts/deploy.sh)
* edit the file .env
* replace the line VIRTUAL_HOST=localhost with VIRTUAL_HOST=example.com
* Run ./scripts/deploy.sh again

Now set up Let's Encrypt as per the above blog posts:

    mkdir "$HOME"/certs
    docker run -d -p 80:80 -p 443:443 \
      --name nginx-proxy \
      -v "$HOME"/certs:/etc/nginx/certs:ro \
      -v /etc/nginx/vhost.d \
      -v /usr/share/nginx/html \
      -v /var/run/docker.sock:/tmp/docker.sock:ro \
      --label com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy \
      --restart=always \
      jwilder/nginx-proxy
    docker run -d \
      --name nginx-letsencrypt \
      -v "$HOME"/certs:/etc/nginx/certs:rw \
      -v /var/run/docker.sock:/var/run/docker.sock:ro \
      --volumes-from nginx-proxy \
      --restart=always \
      jrcs/letsencrypt-nginx-proxy-companion

Figure out the network name

    docker network ls

It is something like "starterkit_drupalsite_default".

Connect your network and restart the Let's Encrypt container:

    docker network connect starterkit_drupalsite_default nginx-proxy
    docker restart nginx-letsencrypt

After 120 seconds the security certificate should work, but you will get "The provided host name is not valid for this server". This means you need to add your host to the allowed hosts. See the "Troubleshooting" section for details.

Now your site should work with LetsEncrypt.

Initial installation on Docker
-----

To install or update the code, install Docker and run `./scripts/deploy.sh`, which will give you a login link to a local development site.

Incremental deployment (updating) on Docker
-----

Updating your local installation is the same command as the installation command:

    ./scripts/deploy.sh

This will bring in new features and keep your existing data.

Environment types
-----

When you run ./scripts/deploy.sh, you can specify one of these **environment types**. If you do not specify an environment type, the "dev" environment type is used.

**dev** (default): used for local development.

* Opcache is turned off: PHP parses your code at every call, insteading of using optimization cache.
* Volumes are shared between your host and your container, meaning changes to your code on your host machine are used by your container.

**build**: used to build an image, potentially for storage on a Docker registry.

* Opcache is turned on because "build" mode is not designed to do development.
* Code is copied to the container, so your container will have everything it needs to run.

./scripts/docker-compose.sh instead of docker-compose
-----

The `docker-compose` command will, by default, use `docker-compose.yml`, but in our setup, `docker-compose.yml` by itself is not valid, because we have several environment types (dev, build, ...). Using `./scripts/docker-compose.sh` will find the right environment type based on the contents of the unversioned `.env` file. For example, if the environment type is "dev", we will use, automatically, `docker-compose.yml` but also `docker-compose.dev.yml` as described in ./scripts/dev/env.txt.

Local and environment-specific settings
-----

If you need environment-specific settings, or need to track sensitiv api keys,
you can create a file named:

    ./drupal/settings/local-settings/unversioned.php

Use ./drupal/settings/local-settings/unversioned.example.php as an example.

Power up/power down the Docker environment
-----

To shut down your containers but _keep your data for next time_:

    ./scripts/docker-compose.sh down

To power up your containers:

    ./scripts/docker-compose.sh up -d

Uninstalling the Docker environment
-----

To shut down your containers and _destroy your data_:

    ./scripts/destroy.sh

If your project is the only project using the local https via the [Nginx Proxy], you might want to destroy the nginx-proxy container

    docker kill nginx-proxy
    docker rm nginx-proxy

You might also want to remove, from /etc/hosts, the line which contains your local development domain (use `sudo vi /etc/hosts` to edit that file); and also remove the certificate for your domain from `~/.docker-compose-certs`.

Prescribed development process
-----

The development cycle is as follows:

* Download the latest code and run `./scripts/deploy.sh`.
* Perform your local development on a branch `my-feature`.
* Export your config changes to code using `./scripts/export-config.sh`.
* Merge the latest version of master and run `./scripts/deploy.sh`.
* Push to Github and open a pull request.
* Self-review or have a colleague review code in the pull request.
* Make sure automated tests and linting is passing using `./scripts/test.sh`
* Use Github's GUI to **Squash and merge** your branch for clean git history.
* Use Github's GUI to delete your branch.
* Delete your branch locally and switch back to master.
* Pull the latest version of master.

Developers should also read this entire ./README.md file and add to it any information which may be useful for other developers.

Starter data
-----

Our approach is to include everything necessary for development within our codebase, meaning that, if you use this project as a basis for your own projects, and follow the same approach, developers will almost never need to obtain a staging database to do work.

Here is how this works:

* In `./drupal/starter-data`, you will find a starter database and starter files such as images.
* When you run `./scripts/deploy.sh`, the code will use the starter data to populate the environment (see also the section "Database schema versions and project versions", below).
* Developers who create, for example, a new content type xyz and a new view at, for example, /listing, will run `./scripts/export-config.sh` to export the content type and view.

The above exports the configuration into code at `./drupal/config`, but not in the starter database. This does not matter, because any time `./scripts/deploy.sh` is run, configuration in code will be imported into the running database.

However the next developer to run `./scripts/deploy.sh` on a new environment, or a testbot, will not have any dummy (starter) data associated with this new configuration. The following steps can be added to a development workflow to remedy this:

* Developers can now create a few dummy nodes of type xyz, along with images in image fields.
* They will then run `./scripts/update-starter-data.sh` which will take this new data and make it part of the codebase.
* This new data will now be available to new developers (and existing developers if they run `./scripts/destroy.sh`, then `./scripts/deploy.sh` again).
* This new data will be available to automated test code at `./tests/browser-tests/test01.js` (which is called by `./scripts/end-to-end-tests.sh` in the continuous integration process).
* You will also be able to test for accessibility of your new code, with dummy data, at `./scripts/a11y-tests.sh` (also called during the continuous integration process).

With this approach, functionality and configuration is deeply integrated with dummy data in the same codebase.

Private and public files
-----

./docker-compose.yml defines two volumes, `drupal-files` (the public files) and `drupal-private-files`, the private files.

The public files volume maps to `/var/www/html/sites/default/files` on the container; the private files volume maps to `/drupal-private-files` on the container. The latter is also set in ./drupal/settings/local-settings/versioned.php.

This means that both public files and private files will persist beyond the life of the drupal container.

Database schema versions and project versions
-----

This project (Dcycle Drupal Starterkit) decouples module versions from the project code. Concretely, in ./Dockerfile-drupal-base, for example, we download, via composer, modules such as _the latest version_ (not _a specific version_) of Webform.

In addition, because Drupal code requires a database to do anything useful, and because we want the benefits of version control on our code, we include, as stated above in the "Starter data" section, in `./drupal/starter-data/`, a minimal database and files which work with our code.

The UUID defined in ./drupal/config/system.site.yml is (and needs to be) identical to the UUID in the starter database.

Because, as mentioned, module versions are decoupled from project code, new versions of modules might implement hook_update_N() to modify the database. So concretely, if you create an instance of this project, it will use the starter database, then update it using the update hooks of the latest versions of all modules, which might be (and probably are) different from the versions used to create the starter database.

If you would like to update the starter database dump based on the current (post-hook_update_N()-)database, you can do so by running ./scripts/update-starter-data.sh (which updates the starter database and also the starter files such as images, based on what is in the running environment).

We include, in ./drupal/config, minimal configuration which works with our code. There are two ways in this config can be updated:

* If module update hooks update the configuration, the code is updated as part of the deployment process (see "hook_update_N() and configuration", below) but only for newly installed sites.
* Developers can run ./scripts/export-config-sh to update the configuration based on new features such as fields, content types, newly-installed modules...

Combined, the above approach allows you to check out any version of the starterkit code, and run the deployment script, ending up with a useful website.

hook_update_N() and configuration
-----

When an update hook modifies the database, our process works seamlessly for developers, even if the starter database is in a different schema than what is expected from the modules: `./scripts/deploy.sh` imports the outdated started database, then updates it using `drush updb`.

Except in the following case:

Drupal modules sometimes use update hooks to modify configuration. For more details see [hook_update_N(), a powerful and dangerous tool to use sparingly, January 29, 2021, Dcycle blog](https://blog.dcycle.com/blog/2021-01-29/hook_update_n/).

Because of [#3110362 If an update hook modifies configuration, then old configuration is imported, the changes made by the update hook are forever lost.](https://www.drupal.org/project/drupal/issues/3110362), and our decoupling of code from and module versions, we will, as part of our deployment script, update the configuration in code if update hooks update the configuration in the database.

Here is an example of how it works:

Let's say our starter database has been created with Webform 5.23. Its default "default_page_base_path" config is "form" without a leading slash:

    drush config:get webform.settings settings.default_page_base_path
    # 'webform.settings:settings.default_page_base_path': form

Running ./scripts/deploy.sh -- on a CI server or on a new installation or after having called ./scripts/destroy.sh -- will do the following: import the starter database normally, then run the ./drupal/scripts/update-config-in-code-if-updb-modifies-config-in-db.sh script (which on the container exists at /scripts/update-config-in-code-if-updb-modifies-config-in-db.sh):

* import the configuration
* run hook updb
* then export the configuration

**Simply running ./scripts/deploy.sh thus has the potential to actually modify your configuration in code. This is necessary to prevent configuration from becoming outdated, which can lead to [#3110362 If an update hook modifies configuration, then old configuration is imported, the changes made by the update hook are forever lost.](https://www.drupal.org/project/drupal/issues/3110362) on production**.

Patches
-----

If you need to apply a patch, make sure it is published on Drupal.org in an issue, and apply it in the Dockerfiles. See ./Dockerfile for an example.

Development design patterns
-----

If you are looking at the code for the first time, here are a few design patterns and approaches which explain some of the choices which were made during the development process:

### Do not commit code which can be dowloaded or generated

We are not committing code such as downloaded modules, keeping the codebase rather small.

### No SASS

We do not use SASS in this project to avoid complexity. If you want to implement SASS support, consider the following questions:

* Generated CSS should be created in the build phase by `./scripts/create-build.sh`. Because we only allow Docker as a dependency of this project, you might consider a technique such as the one outlined in ["Compass Sass to CSS using Docker", Feb. 9, 2016, Dcycle blog](http://blog.dcycle.com/blog/107/compass-sass-css-using-docker/).
* You should not commit generated CSS to this git repo. You might need to `.gitignore` certain files, or make sure they reside on the container, but not your local computer.
* Do you need a real-time compiler directly embedded in your container?

### Docker-based one-click development setup

./scripts/deploy.sh is designed to use Docker to set up everything you need to have a complete environment.

### Common code as traits

We make use of traits to allow classes to implement common code and the Singleton design pattern (see ./drupal/custom-modules/my_custom_module/src/App.php and ./drupal/custom-modules/my_custom_module/my_cystom_module.module for usage).

Removing modules
-----

During development, you might decide that a module needs to be disabled and replaced by some other module or approach.

Your pull request **should not** remove the module from the Dockerfile. The reason is: the production environment needs to have the module available in order to disable it.

If you like, you might put a comment in your Dockerfile stating when the module can be deleted, perhaps 3 months from now, something like:

    # On 2021-10-19, remove block_class
    RUN docker-compose require \
      ...
      drupal/block_class \
      ...

Running automated tests
-----

Developers are encouraged to run `./scripts/test.sh` on their machine. If you have Docker installed, this will work without any further configuration.

This lints code (that is, it checks code for stylistic errors) and runs automated unit PHPUnit tests. If you get linting errors you feel do not apply, see a workaround in the "Linting" section, below.

We are not using Drupal's automated testing framework, rather:

* PHPUnit is used to test the code;
* Any Drupal-specific code is wrapped in methods `./drupal/custom-modules/my_custom_module/src/traits/Environment.php`, then mocked in the unit tests. See `./drupal/custom-modules/my_custom_module/test/AppTest.php` for an example.
* Running tests is done using `./scripts/test.sh` (which includes linting) or `./scripts/unit.sh` (which does not include linting), requiring only Docker and no additional setup.

When submitting a core or contrib patch, debugging failing tests
-----

Let's use an example:

On Dec. 4th, 2020, [this comment was submitted to a core issue, along with a patch](https://www.drupal.org/project/drupal/issues/2273889#comment-13926796). The test failed with 21 failures. Here is how I reproduced the failure locally for debugging purposes:

* From the issue comment, click on the failing test, in this case [PHP 7.3 & MySQL 5.7 27,916 pass, 21 fail](https://www.drupal.org/pift-ci-job/1904405).
* In [the resulting page](https://www.drupal.org/pift-ci-job/1904405), note what you'd like to debug. In this example, we can the class `Drupal\Tests\locale\Functional\LocalePluralFormatTest` seems to fail.
* Make sure you have a running local environment; this can be done using `./scripts/deploy 9`.
* Make sure the patch in question is applied to the Drupal codebase on the container (note that a current limitation of this approach is that we are using the latest stable code, not the latest HEAD):

    ./scripts/shell.sh
    > curl -O https://www.drupal.org/files/issues/2020-12-04/2273889-42-core-plural-index.diff
    > patch -p1 < 2273889-42-core-plural-index.diff
    > exit

* Now run the failing test locally:

```
./scripts/test-class.sh "Drupal\Tests\locale\Functional\LocalePluralFormatTest"
```

Voilà, you will now get the same results locally as you do on Drupal's testbot, making it easier to work on patches.

Security and maintenance updates
-----

Because our git repo does not contain actual Drupal or module code, it downloads these each time ./scripts/deploy.sh is run, so simply running ./scripts/deploy.sh will update everything to the latest version.

Theming
-----

We are using the [CDN Drupal Bootstrap subtheme](https://drupal-bootstrap.org/api/bootstrap/starterkits%21cdn%21README.md/group/sub_theming_cdn/8).

Acquia
-----

This project provides some integration with Acquia Cloud.

### Prerequisites on the Acquia server

* We need to install the same version of PHP as can be found on the containers by typing `./scripts/docker-compose.sh exec -T drupal /bin/bash -c 'php -v'`.
* Make sure you are periodically calling "drush cron" from the command line using [this technique](https://docs.acquia.com/acquia-cloud/manage/cron) on your Acquia site.

### Notes about cron on Acquia

Acquia suggests [this technique](https://docs.acquia.com/acquia-cloud/manage/cron) for setting up cron.

    drush @MY-ACQUIA-ACCOUNT -dv -l http://my-site-on-acquia.example.com cron &>> /var/log/sites/${AH_SITE_NAME}/logs/$(hostname -s)/drush-cron.log

### Do not modify the Acquia git code directly, "build" it instead

Run ./scripts/create-build.sh

Login links for Acquia environments
-----

    ./scripts/acquia/uli.sh

Pantheon (experimental)
-----

If you are using Pantheon as a host, please refer to ./scripts/pantheon/README.md.

Docker on production
-----

If you are deploying to a Docker host, please refer to ./scripts/docker-host/README.md.

Linting
-----

We use a [PHP linter](https://github.com/dcycle/docker-php-lint) and other linters in ./scripts/lint.sh. If your code is not commented correctly or structured correctly, Circle CI continuous integration will fail. See the linter documentation on how to ignore certain blocks of code, for example `t()` is ignored in ./drupal-server/custom-modules/my_custom_module/src/traits/Utilities.php.

    // @codingStandardsIgnoreStart
    ...
    // @codingStandardsIgnoreEnd

Getting a local version of the database
-----

If you have a .sql file with the database database, you can run:

./scripts/docker-compose.sh exec -T drupal /bin/bash -c 'drush sqlc' < /path/to/db.sql

You can set up the Stage File Proxy module to fetch files from the live server instead of importing the files.

Logging
-----

To view logs, you can run:

    ./scripts/ssh.sh

Then use Drush to view logs:

    drush watchdog:show

Profiling (or "Why does it take so long?") part 1: PHP
-----

If you use the "dev" environment type (see "Environment types", above), we include the [XDebug 3](https://xdebug.org) profiler to figure out where your slow code is.

To use it, first run `./scripts/uli.sh`, this will show you the location of the profiling visualizer, for example:

    => Drupal: http://0.0.0.0:50001/...
    ...
    => View profiling results (see README): http://0.0.0.0:50175

**Your port numbers may differ**. If you don't see the "View profiling results" section, make sure you are using the dev environment type (see "Environment types", above).

Next, visit a page on Drupal, but with ?XDEBUG_PROFILE=1, for example:

    => http://0.0.0.0:50001?XDEBUG_PROFILE=1

This will show your web page as normal, but will also track all your PHP code so you can find slow code.

Now visit your profiler at (in this example): http://0.0.0.0:50175/webgrind.

We have included the required dependencies to generate a nice call graph.

If you want to profile a call from the command line, you can do this:

    docker-compose exec -T drupal /bin/bash
    ...
    export XDEBUG_TRIGGER=1
    drush php
    >>> \Drupal\node\Entity\Node::load(1);
    >>> exit

Again, visit your profiler at (in this example): http://0.0.0.0:50175/webgrind.

Profiling (or "Why does it take so long?") part 2: Database
-----

Every time you do anything on your site, you will trigger any number of database operations, some of which can be quite slow.

If you use XDebug, as described above, to debug a slow page load, you might realize that the only slow PHP function is:

    php::PDOStatement->execute

This is not very helpful. To further debug this, you might want to try monitoring slow queries in PHP.

To do this, you can enable and observe slow queries.

### Log into your Drupal container

    docker-compose exec -T drupal /bin/bash

### Get a database command line prompt

    drush sqlc

### Enable slow query logging

    SET GLOBAL slow_query_log=1;

(you can also log all queries by running `SET GLOBAL general_log=1;`.)

### Output data to the database and decide what you mean by "slow query"

In this case you are monitoring all queries taking more than 1.2 seconds.

    SET GLOBAL log_output='TABLE';
    SET GLOBAL long_query_time=1.2;

Now you can visit your slow page, then see your slow queries with the special backslash-G formatting:

    SELECT * FROM mysql.slow_log\G

(You can also view all queries, if you set the general_log to 1, above, by running: `SELECT * FROM mysql.general_log\G`.)

### When you're done, disable slow query logging

    SET GLOBAL slow_query_log=0;

Source: [MariaDB Slow Query Log Overview](https://mariadb.com/kb/en/slow-query-log-overview/).

Logging emails during development
-----

We are using a dummy email server with a full GUI, [Mailhog](https://github.com/mailhog/MailHog), as a destination for our emails during development. This is described in [Debug outgoing emails with Mailhog, a dummy mailserver with a GUI, March 14, 2019, Dcycle Blog](https://blog.dcycle.com/blog/2019-03-14/mailhog/).

Here is how it works:

When you create or update your environment ./scripts/deploy.sh, or at any time by running ./scripts/uli.sh, you will see something like:

    => Drupal: http://0.0.0.0:32783/...
    => Dummy email client: http://0.0.0.0:32781

The ports will be different each time, but using the above ports as an example, you can:

(1) visit http://0.0.0.0:32783/user/password
(2) enter "admin"
(3) you should see "Further instructions have been sent to your email address."
(4) to view the actual email in a GUI, visit (in this example) http://0.0.0.0:32781

Troubleshooting
-----

Start by making sure you have the latest version of your operating system, Docker, and Docker-Compose. Make sure you have at least 8Gb or RAM, and that Docker is assigned at least 6Gb or RAM. Restart your computer or server.

### The provided host name is not valid for this server

This means you are using a custom hostname or IP address to access your site.

For security reasons Drupal does not allow traffic from unknown domains or IP addresses.

Edit ./drupal/settings/local-settings/versioned.php, and add your domain to the list in the "Add your own domain(s) here" section, like this:

    $settings['trusted_host_patterns'] = [
      // Add your own domain(s) here!
      '^example\.com$',
      // These are used for local development.
      '^localhost$',
      '^127\.0\.0\.1$',
      '^0\.0\.0\.0$',
      // This is used for browser testing by ./scripts/a11y-tests.sh and
      // ./scripts/end-to-end-tests.sh (which iteself accesses our site via the
      // host "webserver" at ./tests/browser-tests/testLogInAndEdit.js, for
      // example at http://webserver/user).
      '^webserver$',
    ];

### Unable to connect to MySQL

This is known to happen if you are using an outdated version of Docker, for example on DigitalOcean's Docker image at the time of this writing. The solution is to create a Ubuntu image, then install Docker from scratch as per https://docs.docker.com/engine/install/ubuntu.

### If you cannot import translations

After enabling config_translation, also change the owner and group of /var/www/html/sites/default/files/translations:

    drush en config_translation
    chown www-data:www-data /var/www/html/sites/default/files/translations

### Do not use "docker-compose", use "./scripts/docker-compose.sh"

We construct our docker-compose environment based on an _environment type_ (see above), therefore the `docker-compose.yml` file, on its own, is invalid and will produce:

    ERROR: The Compose file is invalid because ...

The solution is to use, for example, `./scripts/docker-compose.sh ps` instead of `docker-compose ps`.

### First steps if anything goes wrong

Make sure you completely delete your environment using:

    ./scripts/docker-compose.sh down -v

Make sure you have the latest stable version of your OS and of Docker.
If you're really in a bind, you can do a factory reset of Docker (which will kill your local data).
Make sure Docker has 6Gb of RAM. (On Mac OS, for example, this can be done using Docker > Preferences > Advanced, then restarting Docker).

Rerun `./scripts/deploy.sh`, and reimport the stage database if you need it (see "Getting a local version of the database", above).

### ./scripts/docker-compose.sh down -v results in a network error

If you run `docker network rm starterkit_drupalsite_default` and you get "ERROR: network starterkit_drupalsite_default id ... has active endpoints", you might need to disconnect the Drupal 7 site from the Drupal 8 network first. This should be done in the migration process, but it might not have worked. If such is the case, type:

    docker network disconnect starterkit_drupalsite_default $(./scripts/docker-compose.sh ps -q database)

### Beware case-sensitivity

Docker is meant to mimic the Acquia environment relatively well, but there can be certain differences, for instance:

* if you `use path/to/class` instead of `path/To/Class`, it will work in Docker (local) and in automated tests, but fail on Acquia.

### ./scripts/docker-compose.sh build --no-cache

Docker will, by default, used cached versions of each step of the build process for images. For example:

    FROM ubuntu
    RUN apt-get install something

If you _know_ "something" has changed, you might want to run:

    ./scripts/docker-compose.sh build --no-cache
    ./scripts/deploy.sh

Running automated functional tests
-----

Let's say you're developing a pull request for a module, for example [masquerade](http://drupal.org/project/masquerade) you can put that module in ./drupal/custom-modules/, then run automated functional tests by running:

    ./scripts/deploy.sh
    docker-compose exec -T drupal /bin/bash

Then once in the container:

    composer require --dev phpspec/prophecy-phpunit:^2
    mkdir -p sites/simpletest
    chown www-data:www-data sites/simpletest
    php ./core/scripts/run-tests.sh --url http://webserver/ --verbose --sqlite /tmp/test.sqlite masquerade

https://www.drupal.org/project/drupal/issues/2958813#comment-12559056
