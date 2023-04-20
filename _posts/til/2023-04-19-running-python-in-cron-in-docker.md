---
title: Running a Python script periodically in a Docker container using cron
categories:
- til
tags:
- cron
- python
- docker
- docker-compose
- kindle
---

Recently, my partner gave a great idea for utilising my old Kindle: generate a "newspaper" each morning from a bunch of RSS feeds, and email it to the Kindle using "Send-to-Kindle" feature (a blog post about this project is in the works).

I loved this idea, and thought it would be no problem to get a Python script up and running periodically on my Raspberry Pi home server using `cron`. However, I ran into various issues along the way (some of which were not so easy to resolve), so I'm collating all the configuration changes I made in the hopes that it will be useful to someone one day. You can find the full repo for this project [here](https://github.com/andrwcnln/watchman), and I have also included my Dockerfile, docker-compose.yml and crontab at the end of this TIL.

## 1. Double check the user

A lot of problems with `cron` come down to user privileges. Each user has their own `crontab`, and then there is the system-wide *root* `crontab`. The first issue I ran into with creating a `cron` job inside a container was that Docker created the crontab as a non-root user. This issue presented itself to me when I tried to run the following command, to  list the current cronjobs in the Docker container:
```
docker-compose exec container-name crontab -l
```
This returned the following output:
```
no crontab for root
```
Now, it is not necessarily a problem to have non-root `cron` jobs, but just make absolutely certain that you are creating the jobs with the user you expect. For me, I wanted to run as `root`, so I added to following line to my docker-compose.yml:
```
user: root
``` 
Now, the `root` user will be used when building your Docker image and the created `crontab` will be where you expect.

## 2. Missing dependencies
When `cron` calls your Python script, you may run into issues with `ModuleNotFoundError` or `ImportError`, where Python cannot find your installed packages. This is because `cron` does not have access to your system environment variables, including the Python path. You can resolve most of these errors with imports by adding the `PYTHONPATH` environment variable to your `crontab`. This should be the path to your `site-packages` folder, something like this:
```
PYTHONPATH=/usr/bin/local/python3
```
You may also need to add a shebang (`#!`) to your Python script to direct `cron` to the correct version. You can find the Python location with one of the following commands:
```
which python
which py
which python3
```
*NOTE*: These commands must be performed in your Docker container when it is up and running. In `docker-compose` syntax this would be the following (with the name of your container instead of `container-name`):
```
docker-compose exec container-name which python3
```
You can then add this to the top of your Python script, as follows:
```
#!/usr/bin/local/python3
```
## 3. Still missing dependecencies
Some modules will still run into errors even when the PYTHONPATH variable has been set. In particular, I ran into problems with `reportlab` and `Pillow/PIL`:
```
ImportError: cannot import name '_imaging' from 'PIL'
```
This was solved by adding the system PATH to the `crontab` as well. The system path is included in the default `crontab` that is created when you first run `crontab -e`:
```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
Therefore, it is a good idea to include it if you are making a new `crontab` to make sure `cron` can find everything it needs to. 

## 4. Check relative paths in Python
By default, `cron` runs from the default root path. Therefore, both your call to Python in your `crontab` and the filepaths within Python should either be relative to `root` (i.e `/main.py` rather than `main.py`) or just use full paths instead.

## 5. "Failed to build wheel" and related errors
This error is related to Python inside a Docker container rather than `cron`. However, someone might still find it useful. When you install your `requirements.txt`, you may encounter errors such as 
```
legacy-install-failure
error: command '/usr/bin/gcc' failed with exit code 1
fatal error: Python.h: No such file or directory
```
I was able to resolve these by adding `python3-dev`, `wheel` and `Cmake` to my `requirements.txt`. These are sometimes required when packages include other binaries or need to compile other code when installed.

## 6. Other useful tips
- [crontab.guru](crontab.guru) is a great resource for checking `cron` syntax
- Installing vim/nano in your Docker container to make the debugging stage easier. This is especially useful for changing your crontab to run much more frequently, or adding debugging messages etc., when the container is up.

I hope this helped you resolve some errors! I've included my Dockerfile, docker-compose.yml and crontab below if you want to set up a similar project or adjust your own files. The full repo is also available [here](https://github.com/andrwcnln/watchman).

Dockerfile:
```
FROM python:3

COPY . .
RUN python3.11 -m pip install --no-cache-dir -r requirements.txt

RUN touch /var/log/cron.log

RUN apt-get update \  
&& apt-get install cron -y

RUN chmod +x main.py

RUN crontab crontab 

CMD cron -f
```
docker-compose.yml:
```
version: "2.4"

services:
  watchman:
    platform: "linux/arm64/v8"
    image: watchman:latest
    container_name: watchman
    restart: always
    user: root
    build:
      context: build
      dockerfile: Dockerfile
```
crontab:
```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PYTHONPATH=/usr/bin/local/python3
15 7 * * * python3 /main.py >> /var/log/cron.log 2>&1

```