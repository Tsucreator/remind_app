# remind_app
登録してある、金言、名言などをポップしてくれるアプリです。Django練習のために作成しました。

# setting.pyの設定

## setting.pyのSECRET_KEYを外部ファイルから読み込む
安全面からSECRET KEYを外部から読み込むように設定しておく。

（変更前）
```python:setting.py

〜中略〜

SECRET_KEY = 'SECRET KEYの文字列'
```
（変更後）
```python:setting.py

〜中略〜

try:
    from .local_settings import *
except ImportError:
    pass
```

local_settings.pyを作成し
```python:local_settings.py
SECRET_KEY = 'SECRET KEYの文字列'
```

さらに、.gitignoreにlocal_setting.pyを記載。
```python:.gitignore
local_setting.py
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
