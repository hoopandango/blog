---
title: "yet another deployment nightmare."
date: 2023-04-14T00:40:00+05:30
draft: false
series: "Deployment Diaries"
---


{{< figure src="./502_of_death.png"  align="center"
caption="an inescapable trap. image from Yume Nikki." >}}

## introduction
a little bit about myself. i don't have much formal knowledge about deploying web apps. i've learnt a bit from trial and error, stackoverflow, and some other online tutorials. so it's quite common for situations like this to become a mess.

anyway, i had a task yesterday. someone else's app was down, and they're no longer available to work on the project. seems that the website ceased working after an organisation-wide server migration. that's fine though, i'll restart some services and it'll work just right. it was already deployed, after all.

boy was i wrong. what follows is two hours of (minor, yet frustrating) troubles.


## there's a 502. now what?

nginx is somewhat unhelpful with it's memorable `502 Bad Gateway` error page. so, i began digging into the system to see what the problem possibly was.

let's see if the development server works.

```bash
source venv/bin/activate
python manage.py runserver
```

alright, that's active. some time later i checked the development server with a quick curl request `curl 127.0.0.1:8000`, which was functioning as well.

the application came prepacked with `run.sh` and `Procfile` scripts. upon inspection, i realised that those run the dev server as well. welp.

```sh
systemctl status gunicorn.socket
journalctl -u gunicorn.socket
```

that's all operational. socket's open where it should be. let's check the service next.


## a missing setting?
```sh
systemctl status gunicorn
journalctl -u gunicorn
```

```txt
Apr 13 11:19:27 <> gunicorn[788984]:     raise HaltServer(reason, self.WORKER_BOOT_ERROR)
Apr 13 11:19:27 <> gunicorn[788984]: gunicorn.errors.HaltServer: <HaltServer 'Worker failed to boot.' 3>
Apr 13 11:19:27 <> systemd[1]: gunicorn.service: Main process exited, code=exited, status=1/FAILURE
Apr 13 11:19:27 <> systemd[1]: gunicorn.service: Failed with result 'exit-code'.
```

alright, gunicorn's unable to boot up. that's going somewhere. this is a bit worrying though, because why is the dev server working alright then? let's scroll up a few more lines in the journal.

```txt
Apr 13 11:19:27 <> gunicorn[788996]:     raise ImproperlyConfigured(
Apr 13 11:19:27 <> gunicorn[788996]: django.core.exceptions.ImproperlyConfigured: Requested setting EMAIL_HOST_USER, but settings are not configured. You must either define the environment variable DJANGO_SETTINGS_MODULE or call settings.configure() before accessing settings.
```

well. this seems obvious. the `EMAIL_HOST_USER` setting is being searched for, but not found. so i search my `settings.py` file(s) for the relevant setting. however, it's actually present! i merged my settings file to ensure that there wasn't any problem in the imports, and restarted everything a couple times for good measure. still, same error.

google and stackoverflow were not quite helpful either. answers reported that the setting might be missing, misspelt, or there were some environment issues. i added the environment variable pointing to django's settings, via:

```sh
export DJANGO_SETTINGS_MODULE=app.settings
```

and it was still not working. the suggestions were of no help. after a few minutes of struggling, i took a closer look at the error log.

```txt
Apr 13 11:19:27 <> gunicorn[788996]:   File "<frozen importlib._bootstrap_external>", line 848, in exec_module
Apr 13 11:19:27 <> gunicorn[788996]:   File "<frozen importlib._bootstrap>", line 219, in _call_with_frames_removed
<!!!>
Apr 13 11:19:27 <> gunicorn[788996]:   File "<base_dir>/app/__init__.py", line 2, in <module>
Apr 13 11:19:27 <> gunicorn[788996]:     from . import utils
<!!!>
Apr 13 11:19:27 <> gunicorn[788996]:   File "<base_dir>/app/utils.py", line 24, in <module>
Apr 13 11:19:27 <> gunicorn[788996]:     def some_function(<other_params>, from_email=settings.EMAIL_HOST_USER):
<...>
Apr 13 11:19:27 <> gunicorn[788996]:     raise ImproperlyConfigured(
Apr 13 11:19:27 <> gunicorn[788996]: django.core.exceptions.ImproperlyConfigured: Requested setting EMAIL_HOST_USER, but settings are not configured. You must either define the environment variable DJANGO_SETTINGS_MODULE or call settings.configure() before accessing settings.
```

## the hitting of the ceiling.

`app/__init__.py`. now that's a new one.

the cause of the error was a circular dependency. `__init__.py` loads before `wsgi.py`, `__init__.py` is calling `utils.py`, `utils.py` uses `settings.py`, and `settings.py` is only configured / activated in `wsgi.py`.

alright, but the elepahant in the room is, /why/? what purpose would such a script serve?


```py
import utils
from .settings import TIME_ZONE

utils.init(application_name="app", time_zone=TIME_ZONE)
```

ok. hold on, but where does this ask for the EMAIL_HOST_USER. if anything, you'd expect TIME_ZONE to throw the error. well, going into `utils.py`...

```py
def some_function(<other_params>, email=settings.EMAIL_HOST_USER):
```

yup. that's it. because the function header loads during import, it demands that the EMAIL_HOST_USER be ready first. however, these settings are imported from `django.conf` instead of `app`, so they aren't ready until the app starts executing.

_good fucking god._

this still doesn't answer the primary question. _how_ was this whole mess running _before_ the server restart? and why was it running on gunicorn but not nginx?

## now that everything's fixed

who am i kidding. when was it ever this easy?

after another batch of restarts, the gunicorn service is working. also, the nginx service is working. yet, the same old 502 appears. so, what's the problem?

```sh
systemctl status nginx
>>> # no problems here.
journalctl -u nginx
>>> # nothing here of concern as well.
```

alright then, let's check the nginx logs. first, `access.log`.

```txt
192.168.xxx.xxx - - [13/Apr/2023:11:46:39 +0000] "GET / HTTP/1.1" 502 166 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/112.0"
```

then, `error.log`.

```txt
2023/04/13 11:24:04 [error] 789202#789202: *14 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.xxx.xxx, server: <app.somedomain.tld> request: "GET / HTTP/1.1", upstream: "http://192.168.yyy.yyy:8000/", host: "<>"
```

now, i would've saved myself a lot of time if i understood what `upstream` meant exactly. some googling, tinkering, and tutorial-reading later i decided to check my nginx configuration file.


```txt
...
proxy-pass http://192.168.yyy.yyy:8000/
...
```

ok. ok. ok.

what.

hold up.

where's gunicorn writing? into a unix socket. where's nginx listening? on _port 8000_? and it's not even _localhost_ port 8000?

## it's all coming together

seems that the run scripts were honest. this app was only run on dev mode in production. this stopped the `__init__` stuff from breaking everything, and that's where nginx is listening as well. 

also, gunicorn was _still running_ into the socket at the same time. it was simply never listened to by nginx, or anyone else. 

because it's on `192.168.yyy.yyy:8000`, this means that anyone could access the web app directly over port 8000. to be fair i haven't checked the firewall yet, but i can't expect much.

(**EDIT:** they actually block port 8000 via ufw. i'm at a loss for words.)

(**EDIT 2:** ufw is inactive.)

ok. let's fix this.

```txt
...
proxy-pass http://unix:/run/gunicorn.sock/
...
```

there we go. then, an nginx restart followed. and then, voila, everything's working again. nginx listens to gunicorn over a socket, gunicorn serves the app as desired.

## lessons

- read logs, thoroughly and completely.
- verify people's competence before taking over projects.

## postscript: reading ~/.bash_history

this ended up being a bit shorter than expected. regardless, while writing this, i decided to check the bash history for any other hijinx i could locate. here's what i found.

- **apache commands** apparently there was an attempt to run this server on apache before nginx. it seems to be a failed attempt, which only perplexes me more.

```sh
sudo apache.service restart
sudo apac2he.service restart
sudo apac2he.sverce restart
sudo apac2he.server restart
```
_keep trying till you succeed!_

- **chmod -R 777** - the classic.
- **VSCode CLI** usage, which is a thing, apparently.
- cloning of two entirely unrelated repositories into the machine.
- mysqldump with the password in plaintext in the command - wait that one's on me XP

## resources

- the digitalocean tutorial that i follow during server setup: https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-22-04