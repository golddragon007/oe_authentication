# OpenEuropa Authentication

[![Build Status](https://drone.fpfis.eu/api/badges/openeuropa/oe_authentication/status.svg?branch=master)](https://drone.fpfis.eu/openeuropa/oe_authentication)
[![Packagist](https://img.shields.io/packagist/v/openeuropa/oe_authentication.svg)](https://packagist.org/packages/openeuropa/oe_authentication)

The OpenEuropa Authentication module allows to authenticate against EU Login, the European Commission login service.

**Table of contents:**

- [Requirement](#requirement)
- [Installation](#installation)
- [Configuration](#configuration)
- [Enable the module](#enable-the-module)
- [Development](#development)
- [Troubleshooting](#troubleshooting)
  
## Requirement

This module requires the following modules: 
 - [Cas](https://www.drupal.org/project/cas) 

## Installation

The recommended way of installing the OpenEuropa Authentication module is via [Composer][2].

```bash
composer require openeuropa/oe_authentication
```

## Configuration

EU Login service parameters are already set by default when installing the module. Please refer to the EU Login
documentation for the available options that can be specified. You can see Project setup section on how to override 
these parameters.

### Settings overrides

In the Drupal `settings.php` you can override CAS parameters such as the ones below, corresponding to the
`cas.settings` and `oe_authentication.settings` configuration objects.

```php
$config['cas.settings']['server']['hostname'] = 'authentication';
$config['cas.settings']['server']['port'] = '7002';
$config['cas.settings']['server']['path'] = '/cas';
$config['oe_authentication.settings']['register_path'] = 'register';
$config['oe_authentication.settings']['validation_path'] = 'TicketValidationService';
```

By default, the development setup is configured via Task Runner to use the demo CAS server provided in the
`docker-compose.yml.dist`, i.e. `https://authentication:7002`.

If you want to test the module with the actual EU Login service, comment out all the lines above in your `settings.php`
and clear the cache.

### Account Handling & Auto Registration

The module enables the option that a user attempts to login with an account that is not already
registered, they will automatically be created.

See [Cas module](https://www.drupal.org/project/cas) for more information.

### Forced Login

The module enables the Forced Login feature to force anonymous users to
authenticate via CAS when they hit all or some of the pages on your site.

See [Cas module](https://www.drupal.org/project/cas) for more information.

### SSL Verification Setting

The EULogin Authentication server must be accessed over HTTPS, so be sure that the drupal site
needs to know how to verify the SSL/TLS certificate of the Authentication server to be
sure it is authentic.

For development, you can configure the module to disable this verification:
```php
$config['cas.settings']['server']['verify'] = '2';
```
_NOTE: DO NOT USE IN PRODUCTION!_

See [Cas module](https://www.drupal.org/project/cas) for more information.

### Proxy

You can configure the module to "Initialize this client as a proxy" allows 
to make authenticated requests to 3rd services (e.g. ePOETRY). 

```php
$config['cas.settings']['proxy']['initialize'] = TRUE;
```

This option is not enable by default, if you want to use itm please refer you to 
[Enable HTTPS PROXY for the drupal site for development.](#enable-https-proxy-for-the-drupal-site-for-development)
to be sure that your site are available over HTTPS and have good certificates.

See [Cas module](https://www.drupal.org/project/cas) for more information.

## Enable the module

In order to enable the module in your project run:

```bash
./vendor/bin/drush en oe_authentication
```

## Development

The OpenEuropa Authentication project contains all the necessary code and tools for an effective development process,
such as:

- All PHP development dependencies (Drupal core included) are required by [composer.json](composer.json)
- Project setup and installation can be easily handled thanks to the integration with the [Task Runner][3] project.
- All system requirements are containerized using [Docker Composer][4]
- A mock server for testing.

### Project setup

Download all required PHP code by running:

```bash
composer install
```

This will build a fully functional Drupal test site in the `./build` directory that can be used to develop and showcase
the module's functionality.

Before setting up and installing the site make sure to customize default configuration values by copying [runner.yml.dist](runner.yml.dist)
to `./runner.yml` and overriding relevant properties.

To set up the project run:

```bash
./vendor/bin/run drupal:site-setup
```

This will:

- Symlink the theme in  `./build/modules/custom/oe_authentication` so that it's available for the test site
- Setup Drush and Drupal's settings using values from `./runner.yml.dist`. This includes adding parameters for EULogin
- Setup PHPUnit and Behat configuration files using values from `./runner.yml.dist`

After a successful setup install the site by running:

```bash
./vendor/bin/run drupal:site-install
```

This will:

- Install the test site
- Enable the OpenEuropa Authentication module

### Using Docker Compose

#### Requirements

- [Docker][8]
- [Docker-compose][9]

#### Procedure

The setup procedure described above can be sensitively simplified by using Docker Compose.

Copy docker-compose.yml.dist into docker-compose.yml.

You can make any alterations you need for your local Docker setup. However, the defaults should be enough to set the project up.

Run:

```bash
# Initialize all containers as a daemon.
docker-compose up -d
# Download all dependencies of the module.
docker-compose exec web composer install
# Setup the project before the installation. 
docker-compose exec web ./vendor/bin/run drupal:site-setup
# Install the site
docker-compose exec web ./vendor/bin/run drupal:site-install
```

Your test site will be available at [http://localhost:8080/build](http://localhost:8080/build).

Run tests as follows:

```bash
docker-compose exec web ./vendor/bin/phpunit
docker-compose exec web ./vendor/bin/behat
```

#### Access to OpenEuropa Authentication mock server - EULogin

To be able to interact with the OpenEuropa Authentication mock container you need to add the internal container
hostname to the hosts file _of your host OS_.

```bash
echo "127.0.1.1       authentication" >> /etc/hosts
```

To configure the container with User's structures and with some examples of User, you can use files present on the
folder `tests/fixtures/mock-server-config/`. 

The container docker that provides the EULogin Mock Service `ecas-mock-server:4.6.0` are available on a private repo
`registry.fpfis.tech.ec.europa.eu`, please contact [DEVOPS team](DIGIT-NEXTEUROPA-DEVOPS@ec.europa.eu) to have credential.

See [Docker login](https://docs.docker.com/engine/reference/commandline/login/) to connect to the repository.

#### Enable HTTPS PROXY for the drupal site for development.

To enable the https proxy, you should add the service `secureweb` to `docker-compose.yml`

```dockerfile
services:
  secureweb:
    image: aheimsbakk/https-proxy:4
    ports:
      - 80:80
      - 443:443
    links:
      - <name of web container>:http
    restart: always
    volumes:
      - ./tests/fixtures/certs/secureweb:/etc/ssl/private
    environment:
      - SERVER_NAME=secureweb
      - SERVER_ADMIN=webmaster@mydomain.com
      - PORT_REDIRECT=8080
      - SSL_CERT_FILE=/etc/ssl/private/MyKeystore.crt
      - SSL_PRIVKEY_FILE=/etc/ssl/private/MyKeystore.key
      - SSL_CHAIN_FILE=/etc/ssl/private/MyKeystore.p12
```

To be able to interact with the https proxy container you need to add the internal container hostname to the hosts
file _of your host OS_.

```bash
echo "127.0.1.1       secureweb" >> /etc/hosts
```

Your test site will be available at [https://secureweb/build](https://secureweb/build).

### Troubleshooting

#### Disable Drupal 8 caching

Manually disabling Drupal 8 caching is a laborious process that is well described [here][10].

Alternatively you can use the following Drupal Console commands to disable/enable Drupal 8 caching:

```bash
./vendor/bin/drupal site:mode dev  # Disable all caches.
./vendor/bin/drupal site:mode prod # Enable all caches.
```

Note: to fully disable Twig caching the following additional manual steps are required:

1. Open `./build/sites/default/services.yml`
2. Set `cache: false` in `twig.config:` property. E.g.:
```yaml
parameters:
 twig.config:
   cache: false
```
3. Rebuild Drupal cache: `./vendor/bin/drush cr`

This is due to the following [Drupal Console issue][11].

[1]: https://github.com/openeuropa/oe_theme
[2]: https://www.drupal.org/docs/develop/using-composer/using-composer-to-manage-drupal-site-dependencies#managing-contributed
[3]: https://github.com/openeuropa/task-runner
[4]: https://docs.docker.com/compose
[5]: https://github.com/openeuropa/oe_theme#project-setup
[6]: https://nodejs.org/en
[7]: https://www.drupal.org/project/config_devel
[8]: https://www.docker.com/get-docker
[9]: https://docs.docker.com/compose
[10]: https://www.drupal.org/node/2598914
[11]: https://github.com/hechoendrupal/drupal-console/issues/3854
[12]: https://www.drupal.org/docs/8/extending-drupal-8/installing-drupal-8-modules
[13]: https://www.drush.org/
