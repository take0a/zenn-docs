---
title: "GolangでOracle（その２）"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "sqlx", "sql", "日本語", "oracle"]
published: false
publication_name: "robon"
---

# はじめに

先日、Golang で Oracle の各種データ型を扱った場合の評価を行いました。

https://zenn.dev/robon/articles/9ab6cb9e1b9d1c

今回は、日本語の識別子、データを扱った場合の評価を行います。

# Oracle
## データベース

前回と同様、Amazon RDS 上に構築した Oracle サーバを使用します。

## テスト用のテーブル
### 対象とするデータ型

前回は、全てのデータ型を対象としましたが、今回は文字データを保持するものを対象とします。

| コード | データ型 | 説明（抜粋） |
|---|---|---|
| 1 | VARCHAR2(size [BYTE \| CHAR]) | 最大長がsizeバイトまたは文字の可変長文字列。 | 
| 1 | NVARCHAR2(size) | 最大長がsize文字の可変長Unicode文字列。 | 
| 8 | LONG | 最大2GB(231から1を引いたバイト数)の可変長文字データ。 |
| 96 | CHAR [(size [BYTE \| CHAR])] | 長さsizeバイトまたは文字の固定長文字データ。 |
| 96 | NCHAR[(size)] | 長さsize文字の固定長文字データ。 |
| 112 | CLOB | シングルバイト文字またはマルチバイト・キャラクタを含むキャラクタ・ラージ・オブジェクト。 |
| 112 | NCLOB | Unicodeキャラクタを含むキャラクタ・ラージ・オブジェクト。 |

非推奨の LONG 以外は、N なしと N ありの 可変長文字列、固定長文字列、文字型ラージオブジェクト になります。

### テスト用のテーブル

ちょっとアレな感じの CREATE TABLE 文にします。

```sql
CREATE TABLE 日ほンｺﾞ表 (
    列１ VARCHAR2(10 CHAR),
    れつ2 NVARCHAR2(10),
    レツ３ LONG,
    ﾚﾂ4 CHAR(10 CHAR),
    Ｒｅｔｓｕ５ NCHAR(10),
    列６ CLOB,
    列７ NCLOB,
    CONSTRAINT pk_日ほンｺﾞ表 PRIMARY KEY(列１)
);
```

この CREATE TABLE 文の場合は、sqlplus で実行する前に NLS_LANG の設定が必要になります。Amazon Linux 2 上で動かす場合は、以下のようにします。

```bash
$ export NLS_LANG=Japanese_Japan.UTF8
$ sqlplus agra@oracle.■■■■.ap-northeast-1.rds.amazonaws.com/orcl

SQL*Plus: Release 19.0.0.0.0 - Production on 木 5月 25 10:18:47 2023
Version 19.19.0.0.0

Copyright (c) 1982, 2022, Oracle.  All rights reserved.

パスワードを入力してください: 
最終正常ログイン時間: 木 5月  25 2023 10:09:56 +00:00


Oracle Database 19c Standard Edition 2 Release 19.0.0.0.0 - Production
Version 19.18.0.0.0
に接続されました。
SQL> @testdata/create_table.sql
```

# Golang
## 処理系とライブラリ

前回と同様、Golang は、1.18.6。github.com/sijms/go-ora/v2 は、v2.7.6 を使用します。

今回はタイムスタンプは、検証の対象ではありませんので、ドライバ付属の独自型の問題はありません（気になる方は前回の記事を参照ください）。

## アプリケーション

前回と同様ですが、Golang 側の識別子も日本語にしてみます。ただし、Golang の識別子の先頭文字は、パッケージ外への export の可否の識別に使われるため、先頭文字は英大文字にしておきます。

```go
package main

import (
	"database/sql"
	"log"
	"os"
	"reflect"

	"github.com/google/go-cmp/cmp"
	"github.com/jmoiron/sqlx"
	_ "github.com/sijms/go-ora/v2"
)

// S日ほンｺﾞ表 は、日本語テストテーブル
type S日ほンｺﾞ表 struct {
	F列１     sql.NullString `db:"列１"`     // varchar2
	Fれつ2    sql.NullString `db:"れつ2"`    // nvarchar2
	Fレツ３    sql.NullString `db:"レツ３"`    // long
	Fﾚﾂ4    sql.NullString `db:"ﾚﾂ4"`    // char(10)
	FＲｅｔｓｕ５ sql.NullString `db:"ＲＥＴＳＵ５"` // nchar(10) 全角でも大文字
	F列６     sql.NullString `db:"列６"`     // clob
	F列７     sql.NullString `db:"列７"`     // nclob
}

// Key は、日本語テストテーブルのキー
type Key struct {
	Col01 string `db:"COL01"`
}

func main() {
	sqlx.BindDriver("oracle", sqlx.NAMED)

	dsn := os.Getenv("DSN")
	db, err := sqlx.Open("oracle", dsn)
	if err != nil {
		log.Printf("sql.Open error %s", err)
	}

	key := Key{"ｶﾍﾝ長文字列"}
	src := S日ほンｺﾞ表{
		F列１:     sql.NullString{String: "ｶﾍﾝ長文字列", Valid: true},
		Fれつ2:    sql.NullString{String: "ｶﾍﾝ長文字列", Valid: true},
		Fレツ３:    sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
		Fﾚﾂ4:    sql.NullString{String: "ｺﾃｲ長文字列   ", Valid: true},
		FＲｅｔｓｕ５: sql.NullString{String: "ｺﾃｲ長文字列   ", Valid: true},
		F列６:     sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
		F列７:     sql.NullString{String: "あぁアｱｶﾞＡａ漢〇€㈱ー～―‐－", Valid: true},
	}

	_, err = db.NamedExec(`
		INSERT INTO 日ほンｺﾞ表 (
			列１, れつ2, レツ３, ﾚﾂ4, ＲＥＴＳＵ５, 列６, 列７
		) VALUES (
			:列１, :れつ2, :レツ３, :ﾚﾂ4, :ＲＥＴＳＵ５, :列６, :列７
		)`,
		src,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}

	dst := S日ほンｺﾞ表{}
	query, args, err := db.BindNamed(`
		SELECT 
		列１, れつ2, レツ３, ﾚﾂ4, ＲＥＴＳＵ５, 列６, 列７
		FROM 日ほンｺﾞ表 
		WHERE 列１ = :COL01`,
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
		WHERE 列１ = :COL01`,
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

今回の評価では、Oracle 特有の日本語の問題は発見できませんでした。
