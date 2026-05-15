---
title: "FastAPIでREST（その１）"
emoji: "🏎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastapi", "rest", "postgresql", "python", "sqlalchemy"]
published: true
publication_name: "psc"
---

# はじめに

当社では、クラウドネイティブなSaaSの開発をしており、いつもはWebAPIの実装ならサーバーレスで。となるのですが、今回は、サーバーレスでないWebAPIの実現方法として、NestJSに続き、FastAPIを動かしてみましたという内容です。

https://zenn.dev/psc/articles/76d4ec767b72ae

上記のNestJSの記事と同じお題を実装してみます。

# 環境構築
## Python

FastAPIはPythonの型ヒントを活用していて、Pythonのバージョンが異なると型ヒントの書き方や範囲がかわるようです。このため、Pythonのバージョン管理ができたほうが良さそうなので、pyenv+venvで構築していきます。

```bash
$ git clone https://github.com/pyenv/pyenv.git ~/.pyenv
$ cd ~/.pyenv && src/configure && make -C src
```
```bash: .bash_profile
export PYENV_ROOT=~/.pyenv
command -v pyenv >/dev/null || export PATH=$PYENV_ROOT/bin:$PATH
eval "$(pyenv init -)"
```
```bash
$ pyenv --version
pyenv 2.3.17-2-g20189ff0
$ pyenv install 3.10
```

私のAmazon Linux2では、ここでエラーになったので、既に成功している人の情報を参考にさせていただいて
（/tmpに作ってくれるエラーファイルの中身を見ましょう。今回の場合は、末尾ではなく、途中で「あれがない。これがない」と言っていましたが導入記事ではないのでとばします）

```bash
$ sudo yum install -y openssl11 openssl11-devel
$ sudo yum install -y gcc zlib-devel bzip2 bzip2-devel readline-devel sqlite sqlite-devel tk-devel libffi-devel xz-devel
$ pyenv install 3.10
pyenv install 3.10
Downloading Python-3.10.11.tar.xz...
-> https://www.python.org/ftp/python/3.10.11/Python-3.10.11.tar.xz
Installing Python-3.10.11...
Installed Python-3.10.11 to /home/ec2-user/.pyenv/versions/3.10.11
$ pyenv global 3.10
$ pyenv version
3.10.11 (set by /home/ec2-user/.pyenv/version)
$ python --version
Python 3.10.11
```

## venv
```bash
$ mkdir fastapi-sample
$ cd fastapi-sample
$ python -m venv venv
$ source venv/bin/activate
(venv) $ pip list
Package    Version
---------- -------
pip        23.0.1
setuptools 65.5.0
```

## FastAPI

上記の続きで、公式サイトのとおり

```bash
(venv) $ pip install fastapi
(venv) $ pip install "uvicorn[standard]"
(venv) $ pip list
Package           Version
----------------- -------
anyio             3.6.2
click             8.1.3
fastapi           0.95.0
h11               0.14.0
httptools         0.5.0
idna              3.4
pip               23.0.1
pydantic          1.10.7
python-dotenv     1.0.0
PyYAML            6.0
setuptools        65.5.0
sniffio           1.3.0
starlette         0.26.1
typing_extensions 4.5.0
uvicorn           0.21.1
uvloop            0.17.0
watchfiles        0.19.0
websockets        11.0.1
```

## VSCode

fastapi-sampleフォルダを開いて、main.pyファイルを作ると「Flake8拡張しますか？」聞いてくれたので、おとなしく従います。ステータスラインのPythonのバージョンをvenvのものに切り替えます。

```py: main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

VSCodeのターミナルを開くと、venvが反映されているので、

```bash
(venv) $ uvicorn main:app --reload
INFO:     Will watch for changes in these directories: ['/home/ec2-user/work/fastapi-sample']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [25032] using WatchFiles
INFO:     Started server process [25034]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

別のターミナルを開いて、

```bash
(venv) $ curl http://127.0.0.1:8000/items/5?q=somequery
{"item_id":5,"q":"somequery"}
```

ということで、ルーティングまでは簡単に実現できそうです。
また、この時点で、OAS3形式のドキュメントにもアクセスできます。

# 次回は

データベースにアクセスするREST Serverを実装していきます。

https://zenn.dev/psc/articles/bb17fd07739519

