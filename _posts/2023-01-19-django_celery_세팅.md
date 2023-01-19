---
title: django celery 세팅
categories:
- django
# layout: single
date: 2023-01-19
toc: true
toc_label: CATEGORY
toc_sticky: true
# author_profile: true
---

지난번 celery정리에 이어서 django에 celery세팅법을 정리하겠습니다.
{: .notice--info}

### Rabbitmq

메시지 브로커인 rabbitmq는 도커를 사용하여 실행합니다.

```bash
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 --restart=unless-stopped rabbitmq:management
```

[http://localhost:15672/](http://localhost:8080/#/nodes/rabbit%4024bf4ed5c54d)  에 들어가서 rabbitmq가 실행되는지 확인합니다.

rabbitmq:3.7-management

<p align = "center"><img src='/assets/images/posts/2023-01-19/4.png' width="500"/></p>

rabbitmq의 초기 id password는 guest, guest입니다. 로그인을 하고 저 화면이 나오면 성공적으로 실행된 것입니다.

`django-admin startproject [프로젝트 이름]` 명령어로 장고 프로젝트를 만들거나 기존의 프로젝트의  settings.py와 같은 경로에 celery.py를 만듭니다.

[프로젝트이름]/celery.py

```python
import os
from celery import Celery
# celery 디폴트 설정을 settings파일을 참조한다.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', '[프로젝트 이름].settings')

# celery 인스턴스를 설정합니다. broker로 메시지 브로커를 rabbitmq로 설정합니다. 
app = Celery('[프로젝트이름]', broker="amqp://[id]:[password]@rabbitmq//")
  
# 뒤에 namespace='CELERY'는 모든 셀러리 관련 설정 키는 'CELERY_' 라는 접두어를 가져야 한다고 
# 알려줍니다. broker_url => CELERY_BROKER_URL
app.config_from_object('django.conf:settings', namespace='CELERY')

# 다른 앱에 들어있는 tasks.py를 자동으로 찾아준다.
app.autodiscover_tasks()
```

`os.environ.setdefault` 를 코드가  [프로젝트.settings](http://프로젝트.settings.py) 파일을 참조하여 celery를 설정하겠다는 의미입니다. setting 파일의 밑부분에 celery관련 설정을 추가합니다.

[프로젝트이름]/settings.py

```python
INSTALLED_APPS = [
    'django.contrib.admin',
		,,,
    'celery',
    ,,,
]
```

```python
# celery setting
CELERY_BROKER_URL = 'amqp://guest:guest@rabbitmq:5672//'
CELERY_RESULT_BACKEND = 'rpc://guest:guest@rabbitmq:5672//'

CELERY_RESULT_SERIALIZER = 'json'
CELERY_TASK_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Asia/Seoul'
```

`CELERY_BROKER_URL` 로 메시지브로커를 rabbitmq로 설정합니다. guest:guest는 rabbitmq의 초기 id와 password입니다. rabbitmq에서 변경이 가능합니다. 

[프로젝트이름]/__init__.py

```python
# 장고가 시작할 때 @shared_task를 사용가능하게 보장해준다.
# 장고가 시작할 때 celery를 인식한다.
from .celery import app as celery_app

__all__ = ['celery_app']
```

`django-admin startapp [이름]` 명령어를 사용하여 앱을 만들고 경로에 tasks.py를 만들어서 celery가 하는 일을 합니다.

[앱이름]/tasks.py

```python
from .inference.result import ai
from backend.celery import app

# 어노테이션으로 task선언
@app.task()
def imageAi(imageUrl):
    result = ai(imageUrl, "./plants/inference/model.pt")
    return result
```

프로젝트에서 사용했던 함수로 view에서 이미지 경로나 url를 받고 이미지에 대한 간단한 정보를 반환해줍니다.

어노테이션 @app.task로  imageAi()가 celery로 실행되야 한다는 것을 알려줍니다.

이제 모든 준비가 되었습니다. 밑의 명령어를 사용해서 celery를 실행시킬 수 있습니다. 

```bash
celery -A backend worker --loglevel=info --pool=solo
```

밑의 사진처럼 나오면 celery가 실행된 것입니다. 우리가 설정했던 인스턴스 앱이나 메시지 브로커를 확인할 수 있습니다.

<p align = "center"><img src='/assets/images/posts/2023-01-19/5.png' width="500"/></p>

이런 에러가 발생하면 rabbitmq서버를 확인하거나 celery설정에 문제가 없는지 확인합니다.

```python
[2023-01-13 20:06:01,454: ERROR/MainProcess] consumer: Cannot connect to amqp://guest:**@127.0.0.1:5672//: [Errno 61] Connection refused.
Trying again in 2.00 seconds... (1/100)
```

메시지 브로커와 성공적으로 연결이 되면 사진처럼 celery가 일할 준비가 완료 된 것입니다. 이제 `python [manage.py](http://manage.py) runserver` 명령어를 사용하여 비동기동작을 수행할 수 있습니다.

<p align = "center"><img src='/assets/images/posts/2023-01-19/6.png' width="500"/></p>


더 공부하고 싶으면 밑의 깃허브로 공부해보세요. 저도 밑의 깃허브 예제와 유튜브로 celery를 사용해보고 동작을 이해했습니다!
{: .notice--danger} 

[https://github.com/bennett39/celery39](https://github.com/bennett39/celery39)