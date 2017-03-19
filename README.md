# DVWA Docker image

This Docker image contains [DVWA](http://dvwa.co.uk/) which is a "web application that is damn vulnerable".
It's purpose is to demonstrate the most common web related vulnerabilities.


## Disclaimer

Since it includes SERIOUS ones, it's highly unrecommended to put it anywhere close to a production system.
(You have been warned)

To even more lower the risks when running it on your own computer, it is recommended to isolate it.
A Docker image or a VM should be fine, but no warranties.

Read the original [DVWA disclaimer](https://github.com/ethicalhack3r/DVWA#disclaimer) too.


## Usage

First of all pull the image from Docker Hub:

``` bash
$ docker pull sagikazarmark/dvwa
```

Then start it with the following command:

``` bash
$ docker run --rm -it -p 8080:80 sagikazarmark/dvwa
```

(Note: `-it` is required so that you can stop the container with SIGINT)

Now head to `http://localhost:8080` in your browser. Login with `admin` and `password` (hard to guess) credentials.
You should see an installation screen with all options green (except captcha, we will get back to that later).
Click on the **Create / Reset Database** button. When the process is finished,
click on the **login** link and login in again. That's it, you are ready for hacking.

When you are done with playing don't forget to stop the container!!!


## Environment

Since the purpose of this image is to model an average (vulnerable) application, I wanted to keep the environment
as close to a "usual" one as possible. Despite Docker suggests running one process per container,
this image is self-contained: it includes a whole LAMP stack.

The OS of the image is **Debian Jessie**, all softwares are installed from official repositories:

- PHP 5.6.30
- Apache 2.4.10
- MySQL 5.5.54

MySQL credentials are configured to be the DVWA defaults:

- User: **root**
- Password: **p@ssw0rd**
- Database: **dvwa**

Further changes:

- `allow_url_include` PHP option is turned `On` (required by DVWA)
- MySQL `bind_address` option is disabled as well as `root` user is allowed to login from any addresses so that you can access the database from your host computer
- MySQL `general_log` is turned on (default location: `/var/log/mysql/mysql.log`) so that queries can be monitored
- recaptcha is configured in the DVWA configuration to look for `RECAPTCHA_PUBLIC_KEY` and `RECAPTCHA_PRIVATE_KEY` environment variables


## Tips


### Start as daemon

This is **NOT** recommended as you might forget to stop the container (as I did),
but it can be useful if you don't want the container to consume your shell window.
You can start the container as a daemon by replacing `--rm` with `-d`:

``` bash
$ docker run -d -p 8080:80 sagikazarmark/dvwa
```

You might also want to give it a name, so that you can refer to it easily (instead of it's generated name or ID):

``` bash
$ docker run -d --name dvwa -p 8080:80 sagikazarmark/dvwa
```

When you are done, please **STOP** the container (or even delete it):

``` bash
$ docker stop dvwa
$ docker delete dvwa
```


### Expose MySQL to the host

The MySQL server running in the container can be exposed to the host so that you can access the database itself.
All you need to do is adding `-p 3336:3306` to the Docker run command,
where `3336` is the port which you can connect to on your `localhost`:

``` bash
$ docker run --rm -it -p 8080:80 -p 3336:3306 sagikazarmark/dvwa
```

After that you can easily connect to the server:

``` bash
$ mysql -h 127.0.0.1 -P 3336 -u root -pp@ssw0rd
```

This can be useful for example if you want to monitor the server with `mytop`:

``` bash
$ mytop --password="p@ssw0rd" --port=3346 --database="dvwa" --host="127.0.0.1"
```


### Recaptcha

Since recaptcha requires registration, unfortunately I couldn't build it into the image.
If you need captcha, head to `https://www.google.com/recaptcha/admin/create` and register a key.

After that start the container with the following command:

``` bash
$ docker run -d -p 8080:80 -e RECAPTCHA_PUBLIC_KEY=YOUR_KEY -e RECAPTCHA_PRIVATE_KEY=YOUR_KEY sagikazarmark/dvwa
```


### Check logs with lnav

[lnav](http://lnav.org) is an extremely powerful log analyzer which can easily become the developer's best friend.
It allows you to filter/search/highlight or save logs in the console. It knows all kinds of log formats including
Apache log and it can even highlight SQL queries, which makes it a perfect tool for us.

There are two kinds of logs which might be useful in our scenario:

- Apache access and error logs
- MySQL error and general query logs

Fortunately the container automatically tails the Apache logs to the STDOUT of the container, so we can use
the built-in Docker logging:

``` bash
$ docker logs -f dvwa | lnav
```

We are mostly interested in access logs for the vulnerability pages, so start with filtering out all non-relevant log entries.
Type the following (or at least the colon, you can copy-paste the rest):

`:filter-in GET /vulnerabilities`

It can also be useful to search for and highlight a certain part of the log (for example the posted form content in case of SQLi).
For that you can either use the highlight command, or if you want to go quickly, the search function.
As in the previous case, you need to type the slash, but you can copy-paste the rest:

`/(?<=id=)(.*)(?=&Submit=Submit HTTP)`

This will highlight the contents of the `id` field sent as a query parameter.

Sticking to the SQLi example, observing the injected code itself can be useful, but sometimes it's more expressive to
examine the executed SQL query itself. Unfortunately the MySQL query log is not tailed into the container's STDOUT,
but all is not lost, we can still use lnav and a bit of hack:

``` bash
$ docker exec dvwa sh -c "tail -f /var/log/mysql/mysql.log" | lnav
```

The general log contains all kinds of queries, we are mostly interested in `SELECT` ones, so let's filter them:

`:filter-in SELECT`

You can always go back to the unfiltered state with the `Ctrl+R` keystroke.


## Why this image? (aka. Do I suffer from the NIH syndrome?)

I usually try to fight against the [Not invented here](https://en.wikipedia.org/wiki/Not_invented_here) syndrome
and contribute to existing software In this particular case there are already 27 existing Docker images of the
very same DVWA, so it's a valid question to ask why I created another one.


### Quality and reliability

Besides being a NIH fighter, I am also allergic to low quality. This includes everything from code to documentation.
Although there are not too much code in this case that could be wrong, the existing images are highly underdocumented.
My purpose was to provide an image AND a guide with advanced tips, so that it can actually be useful.

In most of the cases the piece of software fulfills the needs of it's publisher,
but it may not be useful for other people. So as a thumb rule a published solution should either aim a very specific,
but common problem or it should aim to be useful for a larger audience with all kinds of use cases.

And a third point (which is very specific to Docker images): I am not a big fan of running images which
does not publish a Dockerfile as well.


### Environment

Most of the images either does not provide any information about their environment or use "special"
dockerized LAMP stacks (mostly `tutum/lamp`).

One of my goals was creating an image as close to "usual" production environments as possible.


### Size

The size of the image is around 360 MB which is relatively small compared to the already existing images.
It's also small in terms of image layers: it has only 5 layers compared to the most popular image which has 41
(mostly because of using `tutum/lamp` as a base image).


## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
