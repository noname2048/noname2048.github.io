---
layout: post
title: AWS + Ubuntu16 + Nginx + uWSGI + Django
data: 2018-09-29 01:33:00 +0900
categories: dev
---
꼭 한번 Nginx를 이용하여 배포해보고 싶었던 프로젝트입니다. 보안 및 제대로된 소규모 앱을 만들어 보고 싶었는데, 여러가지를 뒤적거리면서 해결하게 되었습니다.

[`client`]www - aws ec2[`nginx` - `uwsgi` - `django`]

이 포스트는 virtualenv를 사용하지 않고, 간단히 위에 적혀진 통로를 따라서 welcome Django 페이지를 보실분에게 적절합니다. virtualenv 글은 다른 블로그나, 이 페이지를 공부하신 이후에 좀 더 작업을 진행하시면 가능하실 듯 합니다.

참고한 사이트의 목록은 다음과 같습니다.
* http://brownbears.tistory.com/16
* https://blog.leop0ld.org/posts/use-python3-django-uwsgi-nginx/
* https://rainsound-k.github.io/deploy/2018/05/02/instance-setting-and-django-deploy-part2.html
* https://wayhome25.github.io/django/2018/03/04/django-deploy-04-uwsgi/

하나만 본다면 다음에서 많은 도움을 받았습니다.
* https://wikidocs.net/6611

아마존 EC2 와 Route 53 에 관련한 설정은 다른 설명글에도 많습니다.

다만 제가 놓쳐서 많이 해맸던 부분은

> `Hostzone에 의해 만들어진 DNS 4개`를 기억하여 `Resiter & edit Domain 부분에 기입`한다는 부분입니다.

> 또한 `보안 그룹` 설정 시에, 기본적으로 열려있는 22번은 자신의 IP로(나중에 언제든 변경 가능), 열려있지 않은 80번과 8000번도 열어주셔야 원활한 테스트를 진행 할 수 있음을 기억해주세요. 

>`Ubuntu 18.04 LTS`로 많이 시도했지만, 알 수 없는 오류가 해결되지 않아 다운버전을 시도하여 이를 해결하였습니다. 사람들이 `16.04 LTS`를 괜히 쓰는게 아니고, 또 저도 앞으로 최신버젼에 대한 욕심을 많이 줄일 것 같습니다.

제가 시도했던 때에 각각의 버젼정보는 다음과 같습니다.
* Ubuntu 16.04LTS (free tier version)
* Python3.5.2
* Django 2.1.1
* uWSGI 2.0.1
* Nginx 1.10.3
* `18.09.28`

또한 터미널로는 putty에서 mobaxterm으로 넘어와 진행하였습니다.(호불호에 맞게 사용하세요)

AWS 최초 접속 이후 Advanced Packaging tool 을 이용해 기본적인 서버를 유지 보수를 진행합니다.
```
sudo apt update
sudo apt upgrade
```
> `sudo apt-get update && sudo apt-get upgrade -y` 를 사용하셔도 무방합니다.

> apt와 apt-get의 차이는 apt-get이 좀더 low-level를 다룬다는 점입니다. 사용시에는 크게 차이가 없습니다.

파이썬3이 기본적으로 깔려있고, 파이썬2가 깔려있지 않습니다. pip도 깔려있지 않기 때문에 다음과 같이 설치합니다.
```
sudo -H install python3-pip
sudo -H pip3 install --upgrade pip
```
> 만일 파이썬2도 깔려있다거나, 서버환경이 다르다면 적절한 pip 설치 방법을 찾아주세요

> -H 플래그는 HOME의 의미이며, HOME의 변수를 이용하여 전역설치시 보안과 관련한 세팅을 한다고 합니다. -H flag 가 없으면 warning 및 제대로 작동하지 않을 수 있습니다. 

이후 관련된 패키지도 설치합니다.
```
sudo -H install uwsgi
sudo -H install Django==2.1.1
sudo apt-get install nginx
```

> 여기까지 진행하셨으면 설치가 끝났습니다. 만약 다른 경로로 설치하거나, 진행하신다면.. 알맞게 수정해 주세요

> 서버시간을 설정할 수 도 있습니다.

> 18.04의 경우 여기까지 진행하면 django-admin이 자동으로 경로에 추가 되지 않습니다. 해결방법은 /usr/local/bin에 django-admin.py를 넣는것인데, `django tutorial part1` 부분의 작은 링크에 해결법이 자세히 있습니다.

`Django tutorial part1` 과 `uwsgi django tutorial`을 한번씩 진행하고 나서 아래 코드를 작성하였습니다.

[uwsgi 튜토리얼](https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html)

> 둘다 공식 홈페이지에 설명이 잘 나와있는 편입니다!.. 물론 uwsgi는 잘 정리가 안되서 똑같이 따라했는데도 고생을 좀 하였습니다.

설정파일 목록은 다음과 같습니다.
* /home/ubuntu/project/project/settings.py
* /home/ubuntu/project/project_uwsgi.ini
* /home/ubuntu/project/uwsgi_params
* /etc/nginx/sites-available/project.conf
* /etc/nginx/sites-enable/project.conf


`/home/ubuntu` 에서
```
django-admin startproject project
cd project
vim uwsgi_params
```

```
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;

uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;

uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
```
> uwsgi_params는 uwsig가 기본으로 가지는 설정등을 포함한 것인데,  튜토리얼에 편의를 위해서 그냥 복붙하라고 나와있습니다.
```
vim project_uwsgi.ini
```

```
# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /home/ubuntu/project
# Django's wsgi file
module          = django.core.handlers.wsgi:WSGIHandler()
# the virtualenv (full path)
# home            = /usr/local/bin ... # import django; print(django.__file__)

# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 2
# the socket (use the full path to be safe
socket          = /tmp/project.sock
# ... with appropriate permissions - may be needed
chown-socket    = www-data:www-data
chmod-socket    = 664
# clear environment on exit
vacuum          = true
uid             = www-data
gid             = www-data

module          = project.wsgi:application
env             = DJANGO_SETTNGS_MODULE=project.settins

```
> uwsgi.ini 파일은 명령줄로 실행할 것을 파일로 만들어서 파일로 대체하는데 의의가 있습니다. 편하기도 하고, 자동화하기 위함입니다.

> uid와 gid는 sudo 명령어로 실행하여야 옵션이 먹히는 걸 확인했습니다. (튜토리얼은 그냥 명령을 칩니다. 아마 root 계정인가 봅니다. - 리눅스 배포판 중에는 첫로그인부터 root인 배보판도 있습니다.)

> 664로 했을때, Permision deny가 뜬다면, socket 위치에서 파일 권한이 잘 설정되었는지 확인해야합니다.

> module과 env는 django와 연합하기 위한 설정으로, 튜토리얼을 그대로 따라하였습니다.

```
sudo vim /etc/nginx/sites-available/project.conf
```
```
# project_nginx.conf

# the upstream component nginx needs to connect to
upstream django {
    server unix:///tmp/project.sock; # for a file socket
    # server 127.0.0.1:8001; # for a web port socket (we'll use this first)
}

# configuration of the server
server {
    # the port your site will be served on
    listen      80;
    # the domain name it will serve for
    server_name 서버이름-혹은-주소를-넣으세요; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 125M;   # adjust to taste

    # Django media
    location /media  {
        alias /home/ubuntu/project/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /home/ubuntu/project/static; # your Django project's static files - amend as required
        access_log off;
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /home/ubuntu/project/uwsgi_params; # the uwsgi_params file you installed
    }
}
```
> 전형적인 nginx 설정 중 쉬운것만 골라왔습니다. 마지막 location의 uwsgi_pass를 통해서 nginx의 설정을 그대로 uwsgi로 넘겨줍니다.

> 이 파일의 indentation은 반드시 space 4개로 해주세요. tab을 쓰면 안먹히는 경우가 있다고 들었습니다. (직접 겪은 사실은 아닙니다.)

enable은 available의 파일은 심볼릭 링크만 걸겠습니다.
```
sudo ln -s /etc/nginx/sites-available/project.conf /etc/nginx/sites-enable/
sudo rm /etc/nginx/sites-availalbe/default /etc/nginx/sites-enable/default
```


`/ubuntu/project` 에서
```
sudo /etc/init.d/nginx restart
sudo uwsgi --ini project_uwsgi.ini
```
로 `테스트` 하여, 잘 작동하는지 확인합니다.

> sudo를 붙이지 않으면 uid gid 옵션이 안먹습니다. ㅠㅠ

> 기본적으로 nginx는 www-data(user)로 로그인하여 root와 통신합니다. 그렇지 않으면, 꼭 그렇게 만들어 주세요 `/etc/nginx/nginx.conf`

> www-data를 user혹은 group으로 삼는 socket이 만들어져야 nginx가 원할 하게 읽고 쓰기(접속) 할 수 있습니다. 만약 Permision Deny가 뜬다면,  소켓의 위치 `/tmp` 에서 `ls -al`옵션으로 파일 권한이 uid, gid를 통해 잘 되었는지 확인하세요

> 기타 에러는 파일에 오타가 있거나 파일 위치가 달라지는 경우가 많습니다. 저는 ubuntu를 ubnutu로 쓰거나, 예제를 복붙하다가 다른 위치를 가져온 경우가 종종 있었습니다.

> 만약 encodings 모듈이 없거나, django 모듈이 없다면, 파이썬 패스와 관련된 것 오류입니다. 현 예제는 virtualenv 없이 사용하여 그런 문제가 없었지만, 만약 사용하실 때에는 경로를 잘 정해주세요. `AWS Ubuntu 18.04` 에는 경로와 상관없이 이런 문제가 해결이 안되어 `16.04`로 내려서 해결하였습니다.

참고가능한 에러확인법
```
sudo systemctl status nginx
tail -f /var/log/syslog
tail -f /var/log/nginx/error.log
```

이제 자동화를 하겠습니다. 자동화와 관련된 파일은 두개입니다.
* /etc/systemd/system/uwsgi.service
* /etc/uwsgi/vassals/project_uwsgi.ini

[1] 먼저 uwsgi의 emperor 모드로 관리하도록, ini 파일을 모아둘 저장소를 만듭니다.
```
sudo mkdir /etc/uwsgi
sudo mkdir /etc/uwsgi/vassals
```
그리고는 프로젝트 파일을 링크시킵니다.
```
sudo ln -s /home/ubuntu/project/project_uwsgi.ini /etc/uwsgi/vassals
```
[2] 이제 emperor 모드를 24시간 할 수 있도록 서비스에 등록합니다.
```
sudo vim /etc/systemd/system/uwsgi.service
```
```
[Unit]
Description=uWSGI Emperor service

[Service]
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/vassals
Restart=on-failure
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

이제 서비스를 시작해 봅니다.
```
sudo systemctl enable uwsgi
sudo systemctl start uwsgi
```

에러는 
```
tail -f /var/log/syslog
```
로 확인합니다.


만약 에러가 해결되지 않아 멈추고 싶다면
```
sudo systemctl stop uwsgi
```

혹은 다시 시작하려면
```
sudo systemstl daemon-reload
sudo systemctl restart uwsgi
```

이후 인터넷브라우저로 접속했을때 정상적으로 창을 띄우면 성공입니다.


> 방화벽은 ufw로 설정합니다. 다만 서버 구동시 disable이 기본값이라 그대로 두었습니다.

> 만일 프로세스나 기타등등을 확인하고 싶다면
> ```
> ps -ef | grep nginx
> sudo netstat -ntlp
> ```
> 를 사용하세요
>
> <https://medium.com/@taeyeolkim/aws-ec2%EC%97%90-%EC%9B%B9%EC%84%9C%EB%B2%84-nginx-%EC%84%A4%EC%B9%98%ED%95%98%EA%B3%A0-%EA%B5%AC%EB%8F%99%ED%95%98%EA%B8%B0-a46a6e9484a8>

> 기본적으로 uwsgi 튜토리얼을 토대로 했지만, Django에도 관련 튜토리얼이 존재합니다.
> <https://docs.djangoproject.com/en/2.1/howto/deployment/wsgi/uwsgi/>

이상입니다.
