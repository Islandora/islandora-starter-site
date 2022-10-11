![Islandora logo](https://cloud.githubusercontent.com/assets/2371345/25624809/f95b0972-2f30-11e7-8992-a8f135402cdc.png)

# Islandora Starter Site

[![Minimum PHP Version](https://img.shields.io/badge/php-%3E%3D%207.4-8892BF.svg?style=flat-square)](https://php.net/)
[![Contribution Guidelines](http://img.shields.io/badge/CONTRIBUTING-Guidelines-blue.svg)](./CONTRIBUTING.md)
[![LICENSE](https://img.shields.io/badge/license-GPLv2-blue.svg?style=flat-square)](./LICENSE)

Base configuration and module suite for a starter site.

Your mileage may vary. Depending on your particular use case, some Islandora
prerequisites may not be needed; for example, if you do not need Fedora,
Cantaloupe, or otherwise you can skip their installation and configuration. Note
that this may leave behind stray configurations that should be removed. This
is left as an exercise for the site admin, as documenting such advanced usage is
beyond the scope of this document.

## Prerequisites

1. PHP and [Composer](https://getcomposer.org/) installed
2. Database installation/configuration [for Drupal](https://www.drupal.org/docs/system-requirements/database-server-requirements)
    * Drivers for MySQL/MariaDB/Percona (`mysql`), PostgreSQL (`pgsql`), and
SQLite (`sqlite`) are installed. Using other (contrib) drivers would require
additional installation/configuration steps outside the scope of this document.
3. [Fedora Commons (FCRepo)](https://github.com/fcrepo/fcrepo) installation
    1. [Syn](https://github.com/Islandora/Syn/) installed and configured with a
key.
4. Triplestore installation
5. Cantaloupe installation
    1. `http://127.0.0.1:8080/cantaloupe/iiif/2` is expected to be resolvable,
and to accept full URLs as resource IDs.
6. ActiveMQ/Alpaca/Crayfish installation
    1. ActiveMQ expected to be listening for STOMP messages at `tcp://127.0.0.1:61613`
    2. Queues (and underlying (micro)services) configured appropriately:

| Queue Name                                | Destination                                                |
|-------------------------------------------|------------------------------------------------------------|
| `islandora-connector-homarus`             | Homarus (Crayfish ffmpeg transcoding microservice)         |
| `islandora-indexing-fcrepo-delete`        | FCRepo indexer                                             |
| `islandora-indexing-triplestore-delete`   | Triplestore indexer                                        |
 | `islandora-connector-houdini`             | Houdini (Crayfish imagemagick transformation microservice) |
 | `islandora-connector-ocr`                 | Hypercube (Crayfish OCR microservice)                      |
 | `islandora-indexing-fcrepo-file-external` | FCRepo indexer                                             |
 | `islandora-indexing-fcrepo-media`         | FCRepo indexer                                             |
 | `islandora-indexing-triplestore-index`    | Triplestore indexer                                        |
 | `islandora-indexing-fcrepo-content`       | FCRepo indexer                                             |
 | `islandora-connector-fits`                | CrayFits derivative processor                              |

7. A [Drupal-compatible web server](https://www.drupal.org/docs/system-requirements/web-server-requirements)
8. A [Matomo installation](https://matomo.org/)
    * Further details in the [`matomo` module's](https://www.drupal.org/project/matomo) documentation
9. [FITS Web Service](https://projects.iq.harvard.edu/fits/downloads#fits-servlet) and
[CrayFits](https://github.com/roblib/CrayFits) installations
    * Further details in the [`islandora_fits` module's](https://github.com/roblib/islandora_fits) README/documentation

## Usage

1. Create a project based on this repository:

    ```bash
    composer create-project islandora/islandora-starter-site
    ```

    This should:
    1. Grab the code and all PHP dependencies,
    2. Scaffold the site, and have the `default` site's `settings.php` point at
      our included configuration for the next step.

2. Configure Flysystem's `fedora` scheme in the site's `settings.php`:

    ```php
    $settings['flysystem'] = [
      'fedora' => [
        'driver' => 'fedora',
        'config' => [
          'root' => 'http://127.0.0.1:8080/fcrepo/rest/',
        ],
      ],
    ];
    ```

    Changing `http://127.0.0.1:8080` to point at your Fedora installation.

3. Install the site:

    ```bash
    composer exec -- drush site:install --existing-config
    ```

    After this step, you should configure your web server to serve `web/`
directory as its document root.

4. Add (or otherwise create) a user to the `fedoraadmin` role; for example,
giving the default `admin` user the role:

    ```bash
    composer exec -- drush user:role:add fedoraadmin admin
    ```

5. Make the Syn/JWT keys available to our configuration either by (or by some combination of):
    1. Symlinking the private key to `/opt/islandora/auth/private.key`; or,
    2. Configuring the location to the private key on the site at
`http://<your-web-server>/admin/config/system/keys/manage/islandora_rsa_key`
(where `<your-web-server>` accesses your web server which is configured to serve
from the starter site's `web/` directory).

6. Run the migrations in the `islandora` migration group to populate the base
taxonomies, specifying the `--userid` targeting the user with the `fedoraadmin`
role:

    ```bash
    composer exec -- drush migrate:import --userid=1 islandora_tags,islandora_defaults_tags
    ```

This should get you a starter Islandora site with:

* A basic node bundle to represent repository content
* A handful of media types presentable
* RDF and JSON-LD mappings for miscellaneous entities to support storage in
Fedora, Triplestore indexing and client requests.

### Post-installation cleanup

1. Uninstall the database driver modules you are not using; for example, if
you are using `mysql` to use a MySQL-compatible database, you should be clear to
uninstall the `pgsql` (PostgreSQL) and `sqlite` (SQLite) modules:

    ```bash
    composer exec -- drush pm:uninstall pgsql sqlite
    ```

### Known issues

#### Warnings/errors during installation

Some modules presently have some bad expectations as to the system state when
`hook_install()` is invoked and as such, some messages are emitted:

```
$ composer exec -- drush site:install --existing-config --db-url=mysql://user:***@localhost/db

 You are about to:
 * DROP all tables in your 'db' database.

 Do you want to continue? (yes/no) [yes]:
 >

 [notice] Starting Drupal installation. This takes a while.
 [notice] Performed install task: install_select_language
 [notice] Performed install task: install_select_profile
 [notice] Performed install task: install_load_profile
 [notice] Performed install task: install_verify_requirements
 [notice] Performed install task: install_settings_form
 [notice] Performed install task: install_verify_database_ready
 [notice] Performed install task: install_base_system
 [notice] Performed install task: install_bootstrap_full
 [error]  The Flysystem driver is missing.
 [warning] Could not find required jsonld.settings to add default RDF namespaces.
 [notice] Performed install task: install_config_import_batch
 [notice] Performed install task: install_config_download_translations
 [notice] Performed install task: install_config_revert_install_changes
 [notice] Performed install task: install_configure_form
 [notice] Performed install task: install_finished
 [success] Installation complete.  User name: admin  User password: ***
$
```

There are two "unexpected" messages there:

* `[error]  The Flysystem driver is missing.`
    * Appears to be from [the `flysystem` module's `hook_install()` implementation](https://git.drupalcode.org/project/flysystem/-/blob/cf46f90fa6cda0e794318d04e5e8e6e148818c9a/flysystem.install#L27-32)
      where it tries to ensure that all schemes defined are in a state ready
      to be used; however, all the modules are not yet enabled (in the
      particular case, `islandora` is not actually enabled, so the `fedora` driver is unknown),
      and so leading to this message being emitted. The `islandora` module
      _is_ enabled by the time the command exits, so this message should be
      ignorable.
* `[warning] Could not find required jsonld.settings to add default RDF namespaces.`
    * Appears to be from [the `islandora` module's `hook_install()` implementation](https://github.com/Islandora/islandora/blob/725b5592803564c9727e920b780247e45ecbc9a4/islandora.install#L8-L13)
      where it tries to alter the `jsonld` module's `jsonld.settings` config
      object to add some namespaces; however, because the configs are not yet
      installed when installing the modules with `--existing-config`, it fails
      to find the target configuration to alter it. As exported, the
      `jsonld.settings` already contains the alterations (at time of writing),
      so this warning should be ignorable.

In summary: These two messages seem to be ignorable.

#### Patches

There are currently no patches included with the Starter Site. If a patch (external or internal) is necessary, it can be applied automatically by composer by using the [composer-patches plugin](https://github.com/cweagans/composer-patches). Any patches included in the Starter Site should be described fully here (including when they should be removed).

### Ongoing Project Maintenance

It is anticipated that [Composer](https://getcomposer.org/) will be used to manage Drupal and its extensions,
including Islandora's suite of modules. The Drupal project has already documented many of the interactions in this
space, so we will just list and summarize them here:

[Using Composer to Install Drupal and Manage Dependencies](https://www.drupal.org/docs/develop/using-composer/manage-dependencies)
* The "install" describing:
    * Composer's `create-project` command, which we describe above being used to install
      using this "starter site" project; and,
    * The `drush site:install`/`drush si` command, described above being used to install Drupal via the command line.
* "manage[ment]", describing:
    * Composer's `require` command, used to add additional dependencies to your project
    * Updating, linking out to additional documentation for Drupal's core and modules:
        * [Updating Drupal core via Composer](https://www.drupal.org/docs/updating-drupal/updating-drupal-core-via-composer)
        * [Updating Modules and Themes using Composer](https://www.drupal.org/docs/updating-drupal/updating-modules-and-themes-using-composer)

    Generally, gets into using Composer's `update` command to update extensions according to the specifications in
    the `composer.json`, and Composer's `require` command to _change_ those specifications when necessary to cross
    major version boundaries.

It is also recommended to monitor and/or subscribe to Drupal's security advisories to know when it might be
necessary to update.

## Documentation

Further documentation for this ecosystem is available on the [Islandora documentation site](https://islandora.github.io/documentation/).

## Troubleshooting/Issues

Having problems or solved a problem? Check out the Islandora Google Groups for a solution.

* [Islandora Group](https://groups.google.com/forum/?hl=en&fromgroups#!forum/islandora)
* [Islandora Dev Group](https://groups.google.com/forum/?hl=en&fromgroups#!forum/islandora-dev)

## Development

If you would like to contribute, please get involved by attending our weekly [Tech Call](https://github.com/Islandora/islandora-community/wiki/Weekly-Open-Tech-Call). We love to hear from you!

If you would like to contribute code to the project, you need to be covered by an Islandora Foundation [Contributor License Agreement](https://github.com/Islandora/islandora-community/wiki/Onboarding-Checklist#contributor-license-agreements) or [Corporate Contributor License Agreement](https://github.com/Islandora/islandora-community/wiki/Onboarding-Checklist#contributor-license-agreements). Please see the [Contributor License Agreements](https://github.com/Islandora/islandora-community/wiki/Contributor-License-Agreements) page on the islandora-community wiki for more information.

We recommend using the [islandora-playbook](https://github.com/Islandora-Devops/islandora-playbook) to get started.

### General starter site development process

For development of this starter site proper, we anticipate something of a
particular flow being employed, to avoid having other features and modules creep
into the base configurations. The expected flow should go something like:

1. Provisioning an environment making use of the starter site
    * It _may_ be desirable to replace the environment's starter site installation with a repository clone of the
      starter site at this point to avoid otherwise manually copying changes out to a clone.
2. Importing the config of the starter site
    1. This should overwrite any configuration made by the provisioning process,
       including disabling any modules that should not be generally enabled, and
       installing those that _should_ be.
    2. This might be done with a command in the starter site directory such as:

        ```bash
        composer exec -- drush config:import sync
        ```

3. Perform the desired changes, such as:
    * Using `composer` to manage dependencies:
        * If updating any Drupal extensions, this should be followed by running
          Drupal's update process, in case there are update hooks to run which might
          update configuration.
    * Performing configuration on the site
4. Export the site's config, to capture any changed configuration:

    ```bash
    composer exec -- drush config:export sync
    ```

5. Copying the `config/sync` directory (with its contents) and `composer.json`
   and `composer.lock` files into a clone of the starter site git repository,
   committing them, pushing to a fork and making a pull request.
    * If the environment's starter site installation was replaced with a repository clone, you should be able to skip
      the copying, and just commit your changes, push to a fork and make a pull request to the upstream repository.

Periodically, it is expected that releases will be published/minted/tagged on the original repository; however, it is
important to note that automated updates across releases of this starter site is not planned to be supported. That
said, we plan to include changelogs with instructions of how the changes introduced since the last release might be
effected in derived site for those who wish to adopt altered/introduced functionality into their own site.

#### Development modules

A few modules are included in our `require-dev` section but left uninstalled from the Drupal site, as they may be of
general utility but especially development. Included are:
* [`config_inspector`](https://www.drupal.org/project/config_inspector/): Helps identify potential issues between
  the schemas defined and active configuration.
* [`devel`](https://www.drupal.org/project/devel): Blocks and tabs for miscellaneous tasks during development
* [`restui`](https://www.drupal.org/project/restui): Helper for configuration of the [core `rest` module](https://www.drupal.org/docs/8/core/modules/rest)

These modules might be
[enabled via the GUI or CLI](https://www.drupal.org/docs/extending-drupal/installing-modules#s-step-2-enable-the-module); however, they should be disabled before performing any kind of
config export, to avoid having their enabled state leak into configuration.

## License

[GPLv2](http://www.gnu.org/licenses/gpl-2.0.txt)
