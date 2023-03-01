### First error - "could not find driver"

I was tring to rescan the files in my Nextcloud server (running on Raspberry Pi 4 with DietPi) using the following syntax:

```
sudo -u www-data php /path/to/nextcloud/occ files:scan --all
```

but I kept running into a PHP error. Specifically this error:

```
Doctrine\DBAL\Exception: Failed to connect to the database: An exception occurred in the driver: could not find driver in /path/to/nextcloud/lib/private/DB/Connection.php:139
```

followed by a long, verbose stack trace.

It took me a decent amount of time to diagnose the exact issue, but eventually I found [this list](https://docs.nextcloud.com/server/20/admin_manual/installation/source_installation.html#prerequisites-for-manual-installation) of required PHP modules in the Nextcloud admin manual.

Running `php -m` will print out the list of currently installed PHP modules. I noticed I was missing quite a few of the required modules, but the one that was causing my issue was the missing `pdo_mysql` module.
This can be installed by running:

```
sudo apt-get install php7.4-mysql
```
**Note: This command will change based on your OS, PHP version and database type**

This resolved the error! However (as is always the case), this only meant I got a shiny new error instead:  

### Second error - "Name or service not known"

```
Doctrine\DBAL\Exception: Failed to connect to the database: An exception occurred in the driver: SQLSTATE[HY000] [2002] php_network_getaddresses: getaddrinfo failed: Name or service not known in /path/to/nextcloud/lib/private/DB/Connection.php:139
```

From first glance, this looks like something wrong in the DNS name resolution. This sent me a long way down the wrong path, changing a whole bunch of things in my docker-compose.yml file.
Eventually however, after a long and perilous journey over the high seas of Nextcloud forums and StackOverflow, I found [this example](https://techoverflow.net/2020/07/17/how-to-run-nextcloud-php-occ-in-a-docker-compose-configuration/) of running `php occ` in a docker-compose configuration.
This led me to running this command:

```
docker-compose exec -u www-data nextcloud-app php occ files:scan --all
```
**Note: replace nextcloud-app with the name of your Nextcloud container. Also, this command must be run from the directory of your Nextcloud docker-compose.yml**

....aaaaaand, *voila!* The command runs, the files are scanned and everything is up to date.
