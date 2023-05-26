---
title: "GolangでMySQL/MariaDB（その２）"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "sqlx", "日本語", "mariadb", "mysql"]
published: false
publication_name: "robon"
---

# はじめに

先日、Golang で MySQL と MariaDB の各種データ型を扱った場合の評価を行いました。

https://zenn.dev/robon/articles/9c53c6905f2ffb

今回は、日本語の識別子、データを扱った場合の評価を行います。

# MySQL と MariaDB
## データベース

前回と同様、Amazon RDS 上に構築した MySQL と MariaDB サーバを使用します。

## テスト用のテーブル
### 対象とするデータ型

前回は、全てのデータ型を対象としましたが、今回は文字データを保持するものを対象とします。

| 名称 | 別名 | 説明 |
|------|-----|------|
| char |  | 固定長文字列 |
| varchar |  | 可変長文字列 |
| binary |  | 固定長byte列 |
| varbinary |  | 可変長byte列 |
| tinytext |  | character large object 大きさ順 |
| text |  |  |
| mediumtext |  |  |
| longtext |  |  |
| enum |  | 列挙 |
| set |  | 集合 |
| json |  | JSON文字列 |

MySQL と MriaDB は、text のバリエーションが多いのと enum や set があります。

### テスト用のテーブル

ちょっとアレな感じの CREATE TABLE 文にします。

```sql
DROP TABLE 日ほンｺﾞ表;

CREATE TABLE 日ほンｺﾞ表 (
    列１ char(10) COMMENT '固定長文字列',
    れつ2 varchar(10) COMMENT '可変長文字列',
    レツ３ tinytext COMMENT 'tinytext',
    ﾚﾂ4 text COMMENT 'text',
    Ｒｅｔｓｕ５ mediumtext COMMENT 'mediumtext',
    列６ longtext COMMENT 'longtext',
    列７ enum('小', '中', '大') COMMENT '列挙',
    列８ set('あか', 'みどり', 'あお') COMMENT '集合',
    列９ json COMMENT 'JSON文字列',

    CONSTRAINT pk_日ほンｺﾞ表 PRIMARY KEY(列１)
) COMMENT='日本語テスト';
```

### RDS の MariaDB の characterset 

前回の status コマンドの結果にも表示されていましたが、デフォルトで RDS 上に作成すると latin1 だったので、変更しておきます。

```
MariaDB [agra]>  alter database agra character set utf8mb3;
Query OK, 1 row affected (0.01 sec)

MariaDB [agra]> status
--------------
mysql  Ver 15.1 Distrib 5.5.68-MariaDB, for Linux (x86_64) using readline 5.1

Connection id:          175
Current database:       agra
Current user:           agra@■.■.■.■
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server:                 MariaDB
Server version:         10.6.10-MariaDB-log managed by https://aws.amazon.com/rds/
Protocol version:       10
Connection:             maria.■■■■.ap-northeast-1.rds.amazonaws.com via TCP/IP
Server characterset:    latin1
Db     characterset:    utf8mb3
Client characterset:    utf8mb3
Conn.  characterset:    utf8mb3
TCP port:               3306
Uptime:                 1 hour 52 min 27 sec

Threads: 3  Questions: 4008  Slow queries: 0  Opens: 49  Open tables: 15  Queries per second avg: 0.594
--------------
```

# Golang
## 処理系とライブラリ

前回と同様、Golang は、1.18.6。github.com/go-sql-driver/mysql は、v1.6.0 を使用します。

## アプリケーション

前回と同様ですが、Golang 側の識別子も日本語にしてみます。ただし、Golang の識別子の先頭文字は、パッケージ外への export の可否の識別に使われるため、先頭文字は英大文字にしておきます。

```go
package main

import (
	"database/sql"
	"log"
	"os"
	"reflect"

	_ "github.com/go-sql-driver/mysql"
	"github.com/google/go-cmp/cmp"
	"github.com/jmoiron/sqlx"
)

// S日ほンｺﾞ表 は、日本語テストテーブル
type S日ほンｺﾞ表 struct {
	F列１     sql.NullString `db:"列１"`     // char(10)
	Fれつ2    sql.NullString `db:"れつ2"`    // varchar(10)
	Fレツ３    sql.NullString `db:"レツ３"`    // tinytext
	Fﾚﾂ4    sql.NullString `db:"ﾚﾂ4"`    // text
	FＲｅｔｓｕ５ sql.NullString `db:"Ｒｅｔｓｕ５"` // mediumtext
	F列６     sql.NullString `db:"列６"`     // longtext
	F列７     sql.NullString `db:"列７"`     // enum('小', '中', '大')
	F列８     sql.NullString `db:"列８"`     // set('あか', 'みどり', 'あお')
	F列９     sql.NullString `db:"列９"`     // json
}

// Key は、日本語テストテーブルのキー
type Key struct {
	Col01 string `db:"col01"`
}

func main() {
	dsn := os.Getenv("DSN")
	db, err := sqlx.Open("mysql", dsn)
	if err != nil {
		log.Printf("sql.Open error %s", err)
	}

	key := Key{"ｺﾃｲ長文字列"}
	src := S日ほンｺﾞ表{
		F列１:     sql.NullString{String: "ｺﾃｲ長文字列", Valid: true},
		Fれつ2:    sql.NullString{String: "ｶﾍﾝ長文字列", Valid: true},
		Fレツ３:    sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
		Fﾚﾂ4:    sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
		FＲｅｔｓｕ５: sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
		F列６:     sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
		F列７:     sql.NullString{String: "小", Valid: true},
		F列８:     sql.NullString{String: "あか,みどり,あお", Valid: true},
		F列９:     sql.NullString{String: `{"日本語キー": "日本語文字列"}`, Valid: true},
	}

	_, err = db.NamedExec(`
		INSERT INTO 日ほンｺﾞ表 (
			列１, れつ2, レツ３, ﾚﾂ4, Ｒｅｔｓｕ５, 列６, 列７, 列８, 列９
		) VALUES (
			:列１, :れつ2, :レツ３, :ﾚﾂ4, :Ｒｅｔｓｕ５, :列６, :列７, :列８, :列９
		)`,
		src,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}

	dst := S日ほンｺﾞ表{}
	query, args, err := db.BindNamed(`
		SELECT 
		列１, れつ2, レツ３, ﾚﾂ4, Ｒｅｔｓｕ５, 列６, 列７, 列８, 列９
		FROM 日ほンｺﾞ表 
		WHERE 列１ = :col01`,
		key,
	)
	if err != nil {
		log.Printf("db.BindNamed error %s", err)
	}
	err = db.QueryRowx(query,
		args...,
	).StructScan(
		&dst,
	)
	if err != nil {
		log.Printf("db.QueryRow error %s", err)
	}
	if !reflect.DeepEqual(src, dst) {
		// log.Printf("\nsrc = %#v\ndst = %#v\n", src, dst)
		diff := cmp.Diff(src, dst)
		if len(diff) > 0 {
			log.Print(diff)
		}
	}

	_, err = db.NamedExec(`
		DELETE FROM 日ほンｺﾞ表
		WHERE 列１ = :col01`,
		key,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}
}
```

## sqlx の問題（再掲）

以前の PostgreSQL の評価時に明らかになったのですが、sqlx の [Named Query](http://jmoiron.github.io/sqlx/#namedParams) は、マルチバイト Unicode 対応できていません。

このため、このプログラムを動かすには、sqlx を修正する必要があります。今回の修正は、sqlx には [プルリク](https://github.com/jmoiron/sqlx/pull/865) 済みです（が、休眠しているようなので、取り込まれないかもしれません）。

この修正を適用して動かすためには、以下のリポジトリを clone して、go.mod で replace する必要があります。

https://github.com/roboninc/sqlx

https://github.com/take0a/go-sqlx-sample/blob/master/go.mod

# おわりに

今回の評価では、MySQL と MariaDB に特有の日本語の問題は発見できませんでした。
