# remind_app
登録してある、金言、名言などをポップしてくれるアプリです。Django練習のために作成しました。

# setting.pyの設定

settingファイルは本番(setting.py)と開発(local_setting.py)で分けるのがいいかもしれない。
を起動するときに

`$ python3 manage.py runserver 0.0.0.0:8000 config.local_settings`
でlocal_settings.pyを反映できる。
0.0.0.0:8000はアプリケーションサーバが受けるIP。アプリケーションサーバの8000番ポートがどこからのアクセスも受けるということ。
WebサーバのIPはまた別のはなし。

## setting.pyのSECRET_KEYを外部ファイルから読み込む
gitで管理するのであればSECRET KEYを外部から読み込むように設定しておく。
これは開発環境で使ったものを使いまわさず、本番環境では新たにランダムの文字列を生成し設定するのがいい。

外部から読み込む際には`django-environ`というパッケージを利用すると便利。
```
$ pip3 install django-environ
```
でインストール可能

（変更前）
```python:setting.py

〜中略〜

SECRET_KEY = 'SECRET KEYの文字列'
```
（変更後）
```python:setting.py
import os
import environ

〜中略〜
# SECURITY WARNING: keep the secret key used in production secret!

# SECRET_KEYを外部ファイルから読み込む
env = environ.Env()
# もし、.envファイルが存在したら設定を読み込む（ただし、同じ変数の値は上書きされない。）
env.read_env(os.path.join(BASE_DIR, '.env'))
SECRET_KEY = env('SECRET_KEY')
```

.envを作成し
```python:.env
#ここの「＝」については両端にスペースを入れると読み込まないという事例あり。
SECRET_KEY='SECRET KEYの文字列'
```

さらに、.gitignoreに.envを記載。
```python:.gitignore
.env
```

### 新しい文字列の作成

get_random_secret_key.pyなど適当な名前でファイルを作り

```python:get_random_secret_key.py
#新たにSECRET KEYを作成するためのファイル

from django.core.management.utils import get_random_secret_key

secret_key = get_random_secret_key()
text = 'SECRET_KEY = \'{0}\''.format(secret_key)
print(text)
```

このコマンドを実行する。
```
$ python3 get_random_secret_key.py
```

## 静的ファイルの設定
### PATHの設定

DEBUGがFalseの時（本番環境）ではrunserverが機能しないため、ベースディレクトリ配下(root/remind_appなど)やアプリケーション配下に存在する静的ファイルを読み込まない。
DEBUGがFalseの時は、リバースプロキシを使って自前でsetting.pyの`STATIC_ROOT`に書かれたディレクトリから静的ファイルを読み込むように設定する必要がある。
以下のように設定しておくといい。（あるいは初期設定でなっている。）
```python:local_settings.py

〜中略〜
STATIC_URL = '/static/'
# 開発環境で使うためのDIR（DEBUG = True）
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
# 本番環境で使うためのDIR（DEBUG = False）
STATIC_ROOT = ' /var/www/{}/static'.format(PROJECT_NAME)
```

なお、ここでは`BASE_DIR`、`PROJECT_NAME`という変数を扱うため、下記のように設定しておくと何かと便利。
```python:local_settings.py

〜中略〜
# Build paths inside the project like this: os.path.join(BASE_DIR, ...)
filepath = '自身ファイルpath（Macの場合、自身のHD以下のディレクトリ記載）'
BASE_DIR = os.path.dirname(filepath)
PROJECT_NAME = os.path.basename(BASE_DIR)
```

### 静的ファイルの集約

先に書いたように、DEBUGがFalseの時はwebサーバの挙動が変化する。
次のような手続きが必要になる。
#### リバースプロキシ側で`STATIC_ROOT`配下の静的ファイルを「https://<ホスト名>/<STATIC_URLの値>/...」とのURLリクエストで参照できるように設定

#### Djangoプロジェクト側で`STATIC_ROOT`配下に静的ファイルを集約する
`$ python3 manage.py collectstatic`
を実行する。静的ファイルが`STATIC_ROOT`配下にコピーされる。

### *メディアファイルの管理

ユーザーがアップロードしたファイル、画像などをメディアファイルということにして、一応それらのファイルの管理も記載しておく。
本来アプリケーションサーバで処理されるアップロードファイルだが、リバースプロキシで処理することでアプリケーション側の負荷が少なくなる。
DEBUGがFalseのとき（本番環境）の設定。
```python:local_settings.py

〜中略〜
MEDIA_URL = '/media/'
MEDIA_ROOT = 'var/www/{}/media'.format(PROJECT_NAME)
```
ちなみにMEDIA_ROOTの設定を忘れると、ベースディレクトリ直下にアップロードされる。
開発環境時のメディアファイルの挙動を確認するために、`config/urls.py`に下記urlを書き加えておくと良い。
（あるいはsetting.pyとlocal_setting.pyで分ける）
```python:config/urls.py

〜中略〜
#DEBUGがfalseの時にメデァアファイルの挙動を確かめることができるよう記載
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
]
#DEBUGがfalseの時にメデァアファイルの挙動を確かめることができるよう記載
urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## DataBaseの設定（MySQLのとき）

デフォルトではおそらくSQLiteが指定されている。
今回はMySQLで設定をする。他にもOracleとPostgreが対応している。
```python:setting.py
# Database
# https://docs.djangoproject.com/en/3.0/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'remind_app_db',
        'USER': 'remindappuser',
        'PASSWORD': 'hogehoge',
        'HOST': 'localhost', #DBがあるPCのIPアドレスを入れる
        'PORT': '3306',
        'ATOMIC_REQUESTS':True, #トランザクションの有効範囲を、リクエストの開始から終了までにするかどうか。
        'OPTIONS': {
            'sql_mode': 'TRADITIONAL', 'NO_AUTO_VALUES_ON_ZERO',
        },
    }
}
```
本番環境と開発環境で異なる時はこれもそれぞれで書いておくといい。

ATOMIC_REQUESTSはトランザクションの有効範囲を決める。(デフォルトではFalse)
銀行の送金などでは、片方が送金（表示金額を減らす）、片方が受け取り（表示金額を増やす）処理を行うが
送金が失敗したら受け取りも失敗としないと錬金できてしまう。（あるいはお金が消える。）
したがってトランザクションの範囲をリクエスト開始から終了までにしておき、一連の処理を一つの処理として管理する。

また、オプションについては[こちら](https://docs.djangoproject.com/en/dev/ref/databases/#mysql-notes)に記載。
sql_modeは厳密モードである「STRICT_TRANS_TABLES」や「STRICT_ALL_TABLES」が推奨されてる。
>Django highly recommends activating a strict mode for MySQL to prevent data loss (either STRICT_TRANS_TABLES or STRICT_ALL_TABLES).
>引用元：DjangoDocument

## LOGの設定
ログ出力に関する設定はプロジェクト作成時には設定ファイルには記載がない。
setting.pyに本番用と開発用のログ設定を記載する。この様に本番、開発で大きく異なるときはやはりlocal_...で分けた方がよさそう。

```python:settings.py
#ロギングの設定全般、本番稼働時にファイルにログを保存する用の設定
#log設定
#開発
if DEBUG:
    LOGGING = {
        # バージョンは「1」固定
        'version': 1,
        # 既存のログ設定を無効にしない様に設定
        'disable_existing_loggers': False,
        # ログフォーマット
        'formatters': {
            #開発用
            'develop': {
                'format': '%(asctime)s [%(levelname)s] %(pathtime)s:%(lineno)d'
                          '%(message)s'
            },
        },

        # ハンドラ
        'handlers': {
            # ファイル出力用のハンドラ
            'console': {
                'level': 'DEBUG',
                'class': 'logging.StreamHandler',  # 開発用のハンドラ
                'formatter': 'develop',
            },
        },

        # ロガー
        'loggers': {
            # アプリケーション全般のログを拾うロガー
            '': {
                'handlers': ['console'],
                'level': 'INFO',
                'propagate': False,
            },

            # Django本体のログを拾うロガー
            'django': {
                'handlers': ['console'],
                'level': 'INFO',
                'propagate': False,
            },
            
            # 発行されるSQL文を出力するための設定
            'django.db.backends': {
                'handlers': ['console'],
                'level': 'DEBUG',
                'propagate': False,
            },
        }, 
    }
#本番
else:
    LOGGING = {
        #バージョンは「1」固定
        'version': 1,
        #既存のログ設定を無効にしない様に設定
        'disable_existing_loggers': False,
        #ログフォーマット
        'formatters': {
            #本番用
            'production': {
                'format': '%(asctime)s [%(levelname)s] %(process)d %(thread)d'
                          '%(pathtime)s:%(lineno)d %(message)s'
            },
        },
    
        #ハンドラ
        'handlers': {
            #ファイル出力用のハンドラ
            'file': {
                'level': 'INFO',
                'class': 'logging.handlers.TimeRotatingFileHandler', #ログファイルのローテーションが可能
                'filename': 'var/log/{}/app.log'.format(PROJECT_NAME),
                'formatter': 'production',
            },
        },
    
        #ロガー
        'loggers': {
            #アプリケーション全般のログを拾うロガー
            '': {
                'handlers': ['file'],
                'level': 'INFO',
                'propagate': False,
            },
        
            #Django本体のログを拾うロガー
            'django': {
                'handlers': ['file'],
                'level': 'INFO',
                'propagate': False,
            },
        },(DEBUG == False):
    }
```

ロガーでの設定は「''」で自作のアプリ全般のログ、「'django'」でDjango全体、「'django.db.backends'」でSQLのログになる。
DEBUGがFalseの時はSQLは出力されない。(パフォーマンスの問題)

詳しくは[こちら](https://docs.djangoproject.com/ja/2.2/topics/logging/#topic-logging-parts-loggers )などを参考に。

## Templatesに関する設定

「BACKEND」にはテンプレートエンジン（phpでいうsmartyやbaldeなど）を指定。デフォルトでは「Django Template Language」が設定されている。
「DIR」、「APP_DIR」はtemplatesのフォルダ検索をどこが優先するかの決め打ち。「APP_DIR」が「True」ならばアプリケーションディレクトリ配下のtemplatesが優先される。

## LANGUAGE_CODEの設定

「ja」にしておく。

## TIME_ZOMEの設定

「Asia/Tokyo」にしておく。
```
USE_TZ = True
```
これも確認しておく。

## MIDDLEWAREの設定

```python:setting.py
#HTTP→HTTPSへのリダイレクト機能
SECURE_SSL_REDIRECT = True
```

## ALLOWED_HOSTSの設定

これはセキュリティ対策のための設定でDjangoサイトを配信する際に、誰がホストになるかを指定するもの。
「DEBUG」が「True」となってい時はどうでもいいが、「False」の際は設定する。FQDN（ホスト名とドメイン名をつなげたもの、`www.example.com`）やIPアドレスなどをいれる。
ちなみに「'\*'」で「全てを許可」なので開発中とかはこのままでいい。
