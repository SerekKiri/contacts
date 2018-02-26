# Nextcloud Contacts

![Downloads](https://img.shields.io/github/downloads/nextcloud/contacts/total.svg)
[![irc](https://img.shields.io/badge/IRC-%23nextcloud--contacts%20on%20freenode-blue.svg)](https://webchat.freenode.net/?channels=nextcloud-contacts)
[![Build Status](https://travis-ci.org/nextcloud/contacts.svg?branch=master)](https://travis-ci.org/nextcloud/contacts)
[![Code coverage](https://img.shields.io/codecov/c/github/nextcloud/contacts.svg?maxAge=2592000)](https://codecov.io/gh/nextcloud/contacts/)

**A contacts app for [Nextcloud](https://nextcloud.com). Easily sync contacts from various devices with your Nextcloud and edit them online.**

![](https://raw.githubusercontent.com/nextcloud/screenshots/master/apps/Contacts/contacts.png)

## Why is this so awesome?

* :rocket: **Integration with other Nextcloud apps!** Currently Mail and Calendar – more to come.
* :tada: **Never forget a birthday!** You can sync birthdays and other recurring events with your Nextcloud Calendar.
* :busts_in_silhouette: **Sharing of Adressbooks!** You want to share your contacts with your friends or coworkers? No problem!
* :see_no_evil: **We’re not reinventing the wheel!** Based on the great and open SabreDAV library.

## Installation

In your Nextcloud, simply navigate to »Apps«, choose the category »Organization«, find the Contacts app and enable it.
Then open the Contacts app from the app menu.

## Support

If you need assistance or want to ask a question about Contacts, you are welcome to [ask for support](https://help.nextcloud.com) in our Forums or the [IRC-Channel](https://webchat.freenode.net/?channels=nextcloud-contacts). If you have found a bug, feel free to open a new Issue on GitHub. Keep in mind, that this repository only manages the frontend. If you find bugs or have problems with the CardDAV-Backend, you should ask the guys at [Nextcloud server](https://github.com/nextcloud/server) for help!

## Maintainers:

- [Hendrik Leppelsack](https://github.com/Henni)
- [Jan-Christoph Borchardt](https://github.com/jancborchardt)
- [John Molakvoæ](https://github.com/skjnldsv)

If you'd like to join, just go through the [issue list](https://github.com/nextcloud/contacts/issues) and fix some. :)

### Nightly builds

Instead of setting everything up manually, you can just [download the nightly builds](https://nightly.portknox.net/contacts/?C=M;O=D) instead. These builds are updated every 24 hours, and are pre-configured with all the needed dependencies.

1. Download
2. Extract the tar archive to 'path-to-nextcloud/apps'
3. Navigate to »Apps«, choose the category »Productivity«, find the Contacts app and enable it.

The nightly builds are provided by [Portknox.net](https://portknox.net)

## Building the app

The app can be built by using the provided Makefile by running:

    make

This requires the following things to be present:
* make
* which
* tar: for building the archive
* curl: used if phpunit and composer are not installed to fetch them from the web
* npm: for building and testing everything JS

## Running tests
You can use the provided Makefile to run all tests by using:

    make test

This will run the PHP unit and integration tests and if a package.json is present in the **js/** folder will execute **npm run test**

Of course you can also install [PHPUnit](http://phpunit.de/getting-started.html) and use the configurations directly:

    phpunit -c phpunit.xml

or:

    phpunit -c phpunit.integration.xml

for integration tests
