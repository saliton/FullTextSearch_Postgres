[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/FullTextSearch_Postgres/blob/main/FullTextSearch_Postgres.ipynb)

# Colabで全文検索（その２：PostgreSQL編）

各種全文検索ツールをColabで動かしてみるシリーズです。全7回の予定です。今回はPostgreSQLです。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。以前のデータが残っている場合は取得せず再利用します。）

Google Driveのマウント


```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式に変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試す場合は wiki.json.bz2 を捨てずに残しておくことをおすすめします。


```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

    /content
    CPU times: user 1.58 ms, sys: 75 µs, total: 1.65 ms
    Wall time: 9.63 ms


json形式に変換されたデータを確認


```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## PostgreSQLのインストール

pg_bigmをビルドするのにPostgreSQLのソースコードが必要なようなので、ソースコードからインストールします。


```shell
%cd /content
!wget https://ftp.postgresql.org/pub/source/v14.1/postgresql-14.1.tar.gz
!tar xzf postgresql-14.1.tar.gz
%cd /content/postgresql-14.1/
!./configure
!make install
```

全文検索用のモジュールpg_bigmをインストールします。


```shell
%cd /content
!wget https://ja.osdn.net/projects/pgbigm/downloads/72448/pg_bigm-1.2-20200228.tar.gz
!tar xzf pg_bigm-1.2-20200228.tar.gz
%cd /content/pg_bigm-1.2-20200228
!make USE_PGXS=1 PG_CONFIG=/usr/local/pgsql/bin/pg_config install
```

    /content
    --2022-03-26 08:55:48--  https://ja.osdn.net/projects/pgbigm/downloads/72448/pg_bigm-1.2-20200228.tar.gz
    Resolving ja.osdn.net (ja.osdn.net)... 44.240.119.141, 35.167.170.34
    Connecting to ja.osdn.net (ja.osdn.net)|44.240.119.141|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://ja.osdn.net/frs/redir.php?m=nchc&f=pgbigm%2F72448%2Fpg_bigm-1.2-20200228.tar.gz [following]
    --2022-03-26 08:55:49--  https://ja.osdn.net/frs/redir.php?m=nchc&f=pgbigm%2F72448%2Fpg_bigm-1.2-20200228.tar.gz
    Reusing existing connection to ja.osdn.net:443.
    HTTP request sent, awaiting response... 302 Found
    Location: https://free.nchc.org.tw/osdn//pgbigm/72448/pg_bigm-1.2-20200228.tar.gz [following]
    --2022-03-26 08:55:49--  https://free.nchc.org.tw/osdn//pgbigm/72448/pg_bigm-1.2-20200228.tar.gz
    Resolving free.nchc.org.tw (free.nchc.org.tw)... 140.110.240.80, 2001:e10:2000:240:e643:4bff:fee8:a63c
    Connecting to free.nchc.org.tw (free.nchc.org.tw)|140.110.240.80|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 40420 (39K) [application/x-gzip]
    Saving to: ‘pg_bigm-1.2-20200228.tar.gz’
    
    pg_bigm-1.2-2020022 100%[===================>]  39.47K   229KB/s    in 0.2s    
    
    2022-03-26 08:55:50 (229 KB/s) - ‘pg_bigm-1.2-20200228.tar.gz’ saved [40420/40420]
    
    /content/pg_bigm-1.2-20200228
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -O2 -fPIC -I. -I./ -I/usr/local/pgsql/include/server -I/usr/local/pgsql/include/internal  -D_GNU_SOURCE   -c -o bigm_op.o bigm_op.c
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -O2 -fPIC -I. -I./ -I/usr/local/pgsql/include/server -I/usr/local/pgsql/include/internal  -D_GNU_SOURCE   -c -o bigm_gin.o bigm_gin.c
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -O2 -fPIC -shared -o pg_bigm.so bigm_op.o bigm_gin.o -L/usr/local/pgsql/lib    -Wl,--as-needed -Wl,-rpath,'/usr/local/pgsql/lib',--enable-new-dtags  
    /bin/mkdir -p '/usr/local/pgsql/lib'
    /bin/mkdir -p '/usr/local/pgsql/share/extension'
    /bin/mkdir -p '/usr/local/pgsql/share/extension'
    /usr/bin/install -c -m 755  pg_bigm.so '/usr/local/pgsql/lib/pg_bigm.so'
    /usr/bin/install -c -m 644 .//pg_bigm.control '/usr/local/pgsql/share/extension/'
    /usr/bin/install -c -m 644 .//pg_bigm--1.2.sql .//pg_bigm--1.1--1.2.sql .//pg_bigm--1.0--1.1.sql  '/usr/local/pgsql/share/extension/'


## PostgreSQLの立ち上げ

PostgreSQLを実行するユーザーを作成します。


```shell
!yes | adduser --disabled-password postgres
```

    Adding user `postgres' ...
    Adding new group `postgres' (1000) ...
    Adding new user `postgres' (1000) with group `postgres' ...
    Creating home directory `/home/postgres' ...
    Copying files from `/etc/skel' ...
    Changing the user information for postgres
    Enter the new value, or press ENTER for the default
    	Full Name []: 	Room Number []: 	Work Phone []: 	Home Phone []: 	Other []: Is the information correct? [Y/n] 

DBを構築する場所を初期化します。


```shell
!sudo -u postgres /usr/local/pgsql/bin/initdb -D /tmp/postgres --encoding=UTF8
```

    The files belonging to this database system will be owned by user "postgres".
    This user must also own the server process.
    
    The database cluster will be initialized with locale "en_US.UTF-8".
    The default text search configuration will be set to "english".
    
    Data page checksums are disabled.
    
    creating directory /tmp/postgres ... ok
    creating subdirectories ... ok
    selecting dynamic shared memory implementation ... posix
    selecting default max_connections ... 100
    selecting default shared_buffers ... 128MB
    selecting default time zone ... Etc/UTC
    creating configuration files ... ok
    running bootstrap script ... ok
    performing post-bootstrap initialization ... ok
    syncing data to disk ... ok
    
    initdb: warning: enabling "trust" authentication for local connections
    You can change this by editing pg_hba.conf or using the option -A, or
    --auth-local and --auth-host, the next time you run initdb.
    
    Success. You can now start the database server using:
    
        /usr/local/pgsql/bin/pg_ctl -D /tmp/postgres -l logfile start
    


pg_bigmをロードするための設定を書き込みます。


```shell
!echo shared_preload_libraries = 'pg_bigm' >> /tmp/postgres/postgresql.conf
```

PostgreSQLをバックグラウンドで走らせます。


```bash
%%bash --bg
sudo -u postgres /usr/local/pgsql/bin/pg_ctl -D /tmp/postgres start
```

    Starting job # 2 in a separate thread.


5秒間、起動を待ちます。


```python
import time
time.sleep(5)
```

ユーサーを確認します。


```shell
!echo "\\du" | sudo -u postgres /usr/local/pgsql/bin/psql
```

                                       List of roles
     Role name |                         Attributes                         | Member of 
    -----------+------------------------------------------------------------+-----------
     postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
    


プロセスを確認します。


```shell
!ps aux | grep postgres | grep -v grep
```

    postgres    9778  0.2  0.1 174156 17936 ?        Ss   08:55   0:00 /usr/local/pgsql/bin/postgres -D /tmp/postgres
    postgres    9780  0.0  0.0 174156  2576 ?        Ss   08:55   0:00 postgres: checkpointer 
    postgres    9781  0.0  0.0 174156  2576 ?        Ss   08:55   0:00 postgres: background writer 
    postgres    9782  0.0  0.0 174156  2576 ?        Ss   08:55   0:00 postgres: walwriter 
    postgres    9783  0.0  0.0 174724  5444 ?        Ss   08:55   0:00 postgres: autovacuum launcher 
    postgres    9784  0.0  0.0  28796  2072 ?        Ss   08:55   0:00 postgres: stats collector 
    postgres    9785  0.0  0.0 174588  3404 ?        Ss   08:55   0:00 postgres: logical replication launcher 


## DB作成


```shell
!echo "create database db" | sudo -u postgres /usr/local/pgsql/bin/psql
```

    CREATE DATABASE



```shell
!echo "CREATE EXTENSION pg_bigm" | sudo -u postgres /usr/local/pgsql/bin/psql db
```

    CREATE EXTENSION



```shell
!echo "\\dx" | sudo -u postgres /usr/local/pgsql/bin/psql db
```

                                       List of installed extensions
      Name   | Version |   Schema   |                           Description                            
    ---------+---------+------------+------------------------------------------------------------------
     pg_bigm | 1.2     | public     | text similarity measurement and index searching based on bigrams
     plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
    (2 rows)
    


## Pythonクライアントのインストール


```shell
!pip install psycopg2
```

    Requirement already satisfied: psycopg2 in /usr/local/lib/python3.7/dist-packages (2.7.6.1)


 ## データのインポート

テーブルを作成して、データを50万件登録します。10分ほど時間がかかります。


```python
import psycopg2
import json
import bz2
from tqdm.notebook import tqdm

db = psycopg2.connect(database="db", user="postgres", host="/tmp/")
cursor = db.cursor()

cursor.execute('drop table if exists wiki_jp')
cursor.execute('create table wiki_jp(title text, body text)')

limit = 500000
insert_wiki = 'insert into wiki_jp (title, body) values (%s, %s);'

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    n = 0
    for line in tqdm(fin, total=limit*1.5):
        data = json.loads(line)
        title = data['title'].strip()
        body = data['text'].replace('\n', '')
        if len(title) > 0 and len(body) > 0:
            cursor.execute(insert_wiki, (title, body))
            n += 1
        if n == limit:
            break
db.commit()
db.close()
```

    /usr/local/lib/python3.7/dist-packages/psycopg2/__init__.py:144: UserWarning: The psycopg2 wheel package will be renamed from release 2.8; in order to keep installing from binary please use "pip install psycopg2-binary" instead. For details see: <http://initd.org/psycopg/docs/install.html#binary-install-from-pypi>.
      """)



      0%|          | 0/750000.0 [00:00<?, ?it/s]


登録件数を確認します。


```shell
!echo "select count(*) from wiki_jp;" | sudo -u postgres /usr/local/pgsql/bin/psql db
```

     count  
    --------
     500000
    (1 row)
    


## インデックスを使わない検索

like検索でシーケンシャルに検索した場合を測定します。

検索プランを表示します。インデックスを使っていないことが確認できます。


```shell
!echo "set enable_bitmapscan=off; explain select * from wiki_jp where body like '%日本語%';" | sudo -u postgres /usr/local/pgsql/bin/psql db
```

    SET
                                     QUERY PLAN                                  
    -----------------------------------------------------------------------------
     Gather  (cost=1000.00..69084.02 rows=411 width=64)
       Workers Planned: 2
       ->  Parallel Seq Scan on wiki_jp  (cost=0.00..68042.92 rows=171 width=64)
             Filter: (body ~~ '%日本語%'::text)
    (4 rows)
    


\timingを設定すると、出力の最後に処理時間が出力されます。


```shell
%%writefile command.txt
set enable_bitmapscan=off;
\timing
select * from wiki_jp where body like '%日本語%';
```

    Writing command.txt


出力される件数と処理時間の部分だけをtailコマンドで切り出しています。


```shell
%%time
!sudo -u postgres /usr/local/pgsql/bin/psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 10038.010 ms (00:10.038)
    CPU times: user 154 ms, sys: 27.4 ms, total: 182 ms
    Wall time: 17.8 s


2回目


```shell
%%time
!sudo -u postgres /usr/local/pgsql/bin/psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 6964.317 ms (00:06.964)
    CPU times: user 131 ms, sys: 13.7 ms, total: 145 ms
    Wall time: 14.3 s


内部での処理時間と%%timeによるセルの実行時間に乖離があります。内部の処理時間は検索のみの時間かもしれません。

参考にtitleのみを抽出するクエリを測定します。


```shell
%%writefile command2.txt
set enable_bitmapscan=off;
\timing
select title from wiki_jp where body like '%日本語%';
```

    Overwriting command2.txt



```shell
%%time
!sudo -u postgres /usr/local/pgsql/bin/psql db < command2.txt | tail -3
```

    (17006 rows)
    
    Time: 6340.743 ms (00:06.341)
    CPU times: user 57.4 ms, sys: 16.1 ms, total: 73.6 ms
    Wall time: 6.45 s


内部の検索時間は変わりません。全体の処理時間が短くなったのは、転送する結果の量の違いによるものでしょう。

## 全文検索用インデックスの作成

インデックスの作成には15分ほどかかります。


```shell
%%time
!echo "CREATE INDEX wiki_jp_idx ON wiki_jp USING gin (body gin_bigm_ops);" | sudo -u postgres /usr/local/pgsql/bin/psql db
```

    CREATE INDEX
    CPU times: user 7.24 s, sys: 946 ms, total: 8.18 s
    Wall time: 14min 40s


## インデックスを使った検索

検索プランを表示します。インデックスを使っていることが確認できます。


```shell
!echo "set enable_bitmapscan=on; explain select * from wiki_jp where body like '%日本語%';" | sudo -u postgres /usr/local/pgsql/bin/psql db
```

    SET
                                     QUERY PLAN                                 
    ----------------------------------------------------------------------------
     Bitmap Heap Scan on wiki_jp  (cost=44.39..240.10 rows=50 width=64)
       Recheck Cond: (body ~~ '%日本語%'::text)
       ->  Bitmap Index Scan on wiki_jp_idx  (cost=0.00..44.38 rows=50 width=0)
             Index Cond: (body ~~ '%日本語%'::text)
    (4 rows)
    


インデックスを有効にした検索コマンドを作成します。


```shell
%%writefile command.txt
set enable_bitmapscan=on;
\timing
select * from wiki_jp where body like '%日本語%';
```

    Overwriting command.txt


1回目


```shell
%%time
!sudo -u postgres /usr/local/pgsql/bin/psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 1958.167 ms (00:01.958)
    CPU times: user 83 ms, sys: 18.8 ms, total: 102 ms
    Wall time: 9.26 s


2回目


```shell
%%time
!sudo -u postgres /usr/local/pgsql/bin/psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 1754.218 ms (00:01.754)
    CPU times: user 73.4 ms, sys: 17.2 ms, total: 90.6 ms
    Wall time: 9.26 s


インデックスを使うことによる効果は表れていますが、それほどでもありません。検索対象の数が増えるともっと顕著な差が現れると思われます。

参考にtitleのみを抽出するクエリを測定してみます。


```shell
%%writefile command２.txt
set enable_bitmapscan=on;
\timing
select title from wiki_jp where body like '%日本語%';
```

    Overwriting command２.txt



```shell
%%time
!sudo -u postgres /usr/local/pgsql/bin/psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 1804.455 ms (00:01.804)
    CPU times: user 80.3 ms, sys: 16.7 ms, total: 96.9 ms
    Wall time: 9.06 s


## DBの停止


```shell
!sudo -u postgres /usr/local/pgsql/bin/pg_ctl -D /tmp/postgres stop
```

    waiting for server to shut down.... done
    server stopped

