---
title: "GolangでDb2（その１）"
emoji: "✌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "sqlx", "sql", "database", "db2"]
published: false
publication_name: "robon"
---
# はじめに

前回は、Db2 for Docker を動かしてみました。

https://zenn.dev/robon/articles/155b2d716bfc63

そもそも、目的だった Golang からのアクセスを検証します。

これまで、各種データベースの評価をしてきましたので、Db2以外のデータベースが気になる方は、以下を参照ください。

https://zenn.dev/robon/articles/9a005b45213167
https://zenn.dev/robon/articles/9ab6cb9e1b9d1c
https://zenn.dev/robon/articles/9c53c6905f2ffb
https://zenn.dev/robon/articles/9d6ba760a74b89

# Db2
## データベース

前回動かしたもの Db2 for Docker を使用します。

| 設定 | 値 |
|---|---|
| Database server | DB2/LINUXX8664 11.5.9.0 |
| Edition | Community |

## クライアント

これまでは Amazon EC2 の Amazon Linux 2 でテストしてきましたが、今回からは、Amazon Linux 2023 を使用します。

### ツールとドライバー

これまでは、Pure Go でしたが、今回のライブラリは、ODBC ドライバーの共有ライブラリを必要とするため、IBM Data Server Driver for ODBC and CLI を使用します。インストール方法は[前回の記事](https://zenn.dev/robon/articles/155b2d716bfc63)を参照ください。

## テスト用のテーブル
### データ型の一覧

[マニュアル](https://www.ibm.com/docs/en/db2/11.5?topic=elements-data-types) から読み取った結果です。

| 型名 | 範囲 | 備考 |
|---|---|---|
| TIME | 00:00:00 ～ 23:59:59 | |
| TIMESTAMP | 0001/01/01 00:00:00.000000000001 ～ 9999/12/31 23:59:59.999999999999 | デフォルト小数秒は6桁 |
| DATE | 0001/01/01 ～ 9999/12/31 ||
| CHAR | 固定長文字列（1～255byte or 1～63CODEUNIT32） ||
| VARCHAR | 可変長文字列（1～32,672byte or 1～8,168CODEUNIT32） | |
| CLOB | 可変長文字列（1～2,147,483,647byte or 1～536,870,911CODEUNIT32）||
| GRAPHIC | 固定長グラフィック（2byte文字）文字列（1～127グラフィック文字 or 1～63CODEUNIT32）||
| VARGRAPHIC | 可変長グラフィック文字列（1～16,336グラフィック文字 or 1～8,168CODEUNI32）||
| DBCLOB | 可変長グラフィック文字列（1～1,073,741,823グラフィック文字 or 1～536,870,911CODEUNIT32）||
| BINARY | 固定長バイナリ（1～255byte）||
| VARBINARY | 可変長バイナリ（1～32,672byte）||
| BLOB | 可変長バイナリ（1～2,147,483,647byte）||
| BOOLEAN | 真理値 | TRUE or FALSE |
| SMALLINT | -32,768～32,767 | 16bit |
| INTEGER | -2,147,483,648～2,147,483,647 | 32bit |
| BIGINT | -9,223,372,036,854,775,808 ～ 9,223,372,036,854,775,807 | 64bit |
| DECIMAL | -10^31+1～10^31-1 | 10進数（最大精度31桁） |
| DECFLOAT | 10進浮動小数点数（最大精度34桁）と無限大と非数 ||
| REAL | 単精度浮動小数点数 | 実数の32bit近似 |
| DOUBLE | 倍精度浮動小数点数 | 実数の64bit近似 |
| XML | 整形式のXMLドキュメント ||

### テスト用のテーブル

マニュアルより先に golang 用のドライバーのテストケースから起こした関係で、上記とは順番もブロックも異なりますが、以下の CREATE TABLE 文を使用します。

```sql
create table datatypes (
    col01 int NOT NULL, 
    col02 SMALLINT, 
    col03 BIGINT, 
    col04 INTEGER, 
    col05 DECIMAL(4,2), 
    col06 NUMERIC, 
    col07 float, 
    col08 double, 
    col09 decfloat, 
    col10 char(10), 
    col11 varchar(10), 
    col12 char for bit data, 
    col13 clob(10),
    col14 dbclob(100), 
    col15 date, 
    col16 time, 
    col17 timestamp, 
    col18 blob(10), 
    col19 boolean,
    col20 graphic(10),
    col21 vargraphic(10),
    col22 binary(10),
    col23 varbinary(10),
    col24 xml,
    CONSTRAINT pk_datatypes PRIMARY KEY(col01)
) ccsid unicode;
```

# Golang
## 処理系
```bash
$ go version
go version go1.20.7 linux/amd64
```

## ライブラリ

github.com/ibmdb/go_ibm_db を使用します。

データ型の仕様は一か所にはまとまっていないようなので、入力パラメータなのか出力なのか、カラム情報を作るところなのか、カラム情報からgolangの値を作るところなのか、いろいろ見る必要がありそうです。

オブジェクト指向型なのか、機能分解型なのかというのは、個別の処理の追い易さを優先するのか、網羅的な一覧性の高さを優先するのかという問題でもあるようです。

## アプリケーション

テスト用のテーブルに対して入出力を行うアプリケーションを実装します。歯抜けの状態から徐々に埋めていけるように、Nullable 型で入出力します。

```go: main.go
package main

import (
	"database/sql"
	"log"
	"os"
	"reflect"
	"time"

	"github.com/google/go-cmp/cmp"
	_ "github.com/ibmdb/go_ibm_db"
	"github.com/roboninc/sqlx"
)

// Datatypes は、データ型テストテーブル
type Datatypes struct {
	Col01 sql.NullInt32   `db:"COL01"` // int NOT NULL,
	Col02 sql.NullInt32   `db:"COL02"` // SMALLINT,
	Col03 sql.NullInt64   `db:"COL03"` // BIGINT,
	Col04 sql.NullInt64   `db:"COL04"` // INTEGER,
	Col05 sql.NullString  `db:"COL05"` // DECIMAL(4,2),
	Col06 sql.NullString  `db:"COL06"` // NUMERIC, デフォルトは 5,0
	Col07 sql.NullFloat64 `db:"COL07"` // float,
	Col08 sql.NullFloat64 `db:"COL08"` // double,
	Col09 sql.NullString  `db:"COL09"` // decfloat,
	Col10 sql.NullString  `db:"COL10"` // char(10),
	Col11 sql.NullString  `db:"COL11"` // varchar(10),
	Col12 []byte          `db:"COL12"` // char for bit data,
	Col13 sql.NullString  `db:"COL13"` // clob(10),
	Col14 []byte          `db:"COL14"` // dbclob(100),
	Col15 sql.NullTime    `db:"COL15"` // date,
	Col16 sql.NullTime    `db:"COL16"` // time,
	Col17 sql.NullTime    `db:"COL17"` // timestamp,
	Col18 []byte          `db:"COL18"` // blob(10),
	Col19 sql.NullBool    `db:"COL19"` // boolean,
	Col20 sql.NullString  `db:"COL20"` // graphic(10),
	Col21 sql.NullString  `db:"COL21"` // vargraphic(10),
	Col22 []byte          `db:"COL22"` // binary(10),
	Col23 []byte          `db:"COL23"` // varbinary(10),
	Col24 sql.NullString  `db:"COL24"` // xml,
}

// Key は、データ型テストテーブルのキー
type Key struct {
	Col01 int32 `db:"COL01"`
}

func main() {
	sqlx.BindDriver("go_ibm_db", sqlx.QUESTION)

	dsn := os.Getenv("DSN")
	db, err := sqlx.Open("go_ibm_db", dsn)
	if err != nil {
		log.Printf("sql.Open error %s", err)
	}

	key := Key{1}
	src := Datatypes{
		Col01: sql.NullInt32{Int32: 1, Valid: true},
		Col02: sql.NullInt32{Int32: 2, Valid: true},
		Col03: sql.NullInt64{Int64: 3, Valid: true},
		Col04: sql.NullInt64{Int64: 4, Valid: true},
		Col05: sql.NullString{String: "12.34", Valid: true},
		Col06: sql.NullString{String: "12345", Valid: true},
		Col07: sql.NullFloat64{Float64: 1.5, Valid: true},
		Col08: sql.NullFloat64{Float64: 2.125, Valid: true},
		Col09: sql.NullString{String: "1234567890.12345", Valid: true},
		Col10: sql.NullString{String: "char(10)  ", Valid: true},
		Col11: sql.NullString{String: "varchar", Valid: true},
		// Col12: sql.NullString{String: "31", Valid: true},        input != output
		Col12: []byte{0x31},
		Col13: sql.NullString{String: "clob(10)", Valid: true},
		// Col14: sql.NullString{String: "ａ", Valid: true},		input != output
		Col14: []byte{0x82, 0xA0}, // あ
		Col15: sql.NullTime{Time: time.Date(2001, 2, 3, 0, 0, 0, 0, time.UTC), Valid: true},
		Col16: sql.NullTime{Time: time.Date(0001, 1, 1, 4, 5, 6, 0, time.UTC), Valid: true}, // 0001-01-01固定（SQLServerと同じ）
		Col17: sql.NullTime{Time: time.Date(2001, 2, 3, 4, 5, 6, 0, time.UTC), Valid: true},
		Col18: []byte{1, 2, 3},
		Col19: sql.NullBool{Bool: true, Valid: true},
		Col20: sql.NullString{String: "ぐらふぃっく　　　　", Valid: true},
		Col21: sql.NullString{String: "グラフィック", Valid: true},
		Col22: []byte{1, 2, 3, 0, 0, 0, 0, 0, 0, 0},
		Col23: []byte{1, 2, 3},
		// Col24: sql.NullString{String: "<xml /><root></root>", Valid: true},  SQL16110N  XML syntax error.
		// Col24: []byte("<xml /><root></root>"),　　　　　　　　　　　　　　　　　SQL16110N  XML syntax error.
	}

	_, err = db.NamedExec(`
		INSERT INTO datatypes (
			COL01, COL02, COL03, COL04, COL05, COL06, COL07, COL08, COL09, COL10,
			COL11, COL12, COL13, COL14, COL15, COL16, COL17, COL18, COL19, COL20,
			COL21, COL22, COL23, COL24
		) VALUES (
			:COL01, :COL02, :COL03, :COL04, :COL05, :COL06, :COL07, :COL08, :COL09, :COL10,
			:COL11, :COL12, :COL13, :COL14, :COL15, :COL16, :COL17, :COL18, :COL19, :COL20,
			:COL21, :COL22, :COL23, :COL24
		)`,
		src,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}

	dst := Datatypes{}
	query, args, err := db.BindNamed(`
		SELECT 
			COL01, COL02, COL03, COL04, COL05, COL06, COL07, COL08, COL09, COL10,
			COL11, COL12, COL13, COL14, COL15, COL16, COL17, COL18, COL19, COL20,
			COL21, COL22, COL23, COL24
		FROM datatypes 
		WHERE COL01 = :COL01`,
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
		DELETE FROM datatypes
		WHERE COL01 = :COL01`,
		key,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}
}
```

# おわりに

Db2 の `char for bit data` や `dbclob` は、「初めまして」な感じで苦労しましたが、golang 側からは直接、文字や文字列として扱うのは難しそうなので、一旦、[]byte で受ける必要がありそうです。
また、`xml` については、他のデータベースのように単なる文字列ではなく、独自データ型のようなので、使いたい場合は、別途、調査が必要のようです。

日本語については、別途、テストしたいと思います。
