---
title: "GolangでOracle（その１）"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "sqlx", "sql", "database", "oracle"]
published: true
publication_name: "psc"
---
# はじめに

前回は、Golang で MySQL/MariaDB の各種データ型の入出力を検証しました。

https://zenn.dev/psc/articles/9c53c6905f2ffb

今回は、Oracle を評価します。

# Oracle
## データベース

Amazon RDS 上に構築した Oracle サーバーを使用します。

| 設定 | 値 |
|---|---|
| エンジンバージョン | 19.0.0.0.ru-2023-01.rur-2023-01.r1 |
| 文字セット | AL32UTF8 |
| インスタンスクラス | db.m5.large |
| vCPU | 2 |
| RAM | 8GB |
| ストレージ | 20GiB |

## クライアント

Amazon EC2 の Amazon Linux 2 から接続します。

### ツール

sqlplus を使用しますので、[Oracle社のサイト](https://www.oracle.com/jp/database/technologies/instant-client/linux-x86-64-downloads.html) からダウンロードします。（Oracleアカウントが必要になります）

Amazon Linux 2 の場合は、以下のようにインストールします。

```bash
$ sudo yum install oracle-instantclient19.19-basic-19.19.0.0.0-1.x86_64.rpm 
$ sudo yum install oracle-instantclient19.19-sqlplus-19.19.0.0.0-1.x86_64.rpm 
```

## テスト用のテーブル
### データ型の一覧

[マニュアル](https://docs.oracle.com/cd/E57425_01/121/SQLRF/sql_elements001.htm#i54330) から抜粋します。

| コード | データ型 | 説明（抜粋） |
|---|---|---|
| 1 | VARCHAR2(size [BYTE \| CHAR]) | 最大長がsizeバイトまたは文字の可変長文字列。 | 
| 1 | NVARCHAR2(size) | 最大長がsize文字の可変長Unicode文字列。 | 
| 2 | NUMBER [ (p [, s]) ] | 精度p、位取りsを持つ数。 | 
| 2 | FLOAT [(p)] | 精度pを持つNUMBERデータ型のサブタイプ。 | 
| 8 | LONG | 最大2GB(231から1を引いたバイト数)の可変長文字データ。 |
| 12 | DATE | 紀元前4712年1月1日から紀元9999年12月31日までの日付を指定します。 | 
| 100 | BINARY_FLOAT | 32ビットの浮動小数点数。 |
| 101 | BINARY_DOUBLE | 64ビットの浮動小数点数。 |
| 180 | TIMESTAMP [(fractional_seconds_precision)] | 日付の年、月、日および時刻の時、分、秒の値。 |
| 181 | TIMESTAMP [(fractional_seconds_precision)] WITH TIME ZONE | タイムゾーンによる時差などのすべてのTIMESTAMPの値。 |
| 231 | TIMESTAMP [(fractional_seconds_precision)] WITH LOCAL TIME ZONE | TIMESTAMP WITH TIME ZONEのすべての値。 |
| 182 | INTERVAL YEAR [(year_precision)] TO MONTH | 年および月で期間を格納。 |
| 183 | INTERVAL DAY [(day_precision)] TO SECOND [(fractional_seconds_precision)] | 日、時、分および秒で期間を格納。 |
| 23 | RAW(size) | 長さsizeバイトのバイナリ・データ。 | 
| 24 | LONG RAW | 最大2GBの可変長バイナリ・データ。 |
| 69 | ROWID | 表の行のアドレスを一意に表すBASE64文字列。 |
| 208 | UROWID [(size)] | 索引構成表の行の論理アドレスを表すBASE64文字列。 |
| 96 | CHAR [(size [BYTE \| CHAR])] | 長さsizeバイトまたは文字の固定長文字データ。 |
| 96 | NCHAR[(size)] | 長さsize文字の固定長文字データ。 |
| 112 | CLOB | シングルバイト文字またはマルチバイト・キャラクタを含むキャラクタ・ラージ・オブジェクト。 |
| 112 | NCLOB | Unicodeキャラクタを含むキャラクタ・ラージ・オブジェクト。 |
| 113 | BLOB | バイナリ・ラージ・オブジェクト。 | 
| 114 | BFILE | データベース外に保存された大きなバイナリ・ファイルへロケータを格納。 |

### テスト用のテーブル

そのまま CREATE TABLE文にします。

```sql
CREATE TABLE DATATYPES (
    COL01 VARCHAR2(10 CHAR),
    COL02 NVARCHAR2(10),
    COL03 NUMBER(10),
    COL04 NUMBER(10,4),
    COL05 FLOAT(10),
    COL06 LONG,
    COL07 DATE,
    COL08 BINARY_FLOAT,
    COL09 BINARY_DOUBLE,
    COL10 TIMESTAMP,
    COL11 TIMESTAMP WITH TIME ZONE,
    COL12 TIMESTAMP WITH LOCAL TIME ZONE,
    COL13 INTERVAL YEAR TO MONTH,
    COL14 INTERVAL DAY TO SECOND,
    COL15 RAW(10),
    -- COL16 LONG RAW, ORA-01754: a table may contain only one column of type LONG
    COL17 ROWID,
    COL18 UROWID,
    COL19 CHAR(10 CHAR),
    COL20 NCHAR(10),
    COL21 CLOB,
    COL22 NCLOB,
    COL23 BLOB,
    COL24 BFILE,
    CONSTRAINT PK_CUSTOMER PRIMARY KEY(COL01)
);
```

# Golang
## 処理系
```bash
$ go version
go version go1.18.6 linux/amd64
```

## ライブラリ

github.com/sijms/go-ora/v2 の v2.7.6 を使用します。データ型については、以下の仕様でつくられているようです。

https://github.com/sijms/go-ora/blob/v2.7.6/v2/data_set.go#L378-L442

## アプリケーション

テスト用のテーブルに対して入出力を行うアプリケーションを実装します。歯抜けの状態から徐々に埋めていけるように、Nullable 型で入出力します。

```go: main.go
package main

import (
	"database/sql"
	"database/sql/driver"
	"log"
	"os"
	"reflect"
	"time"

	"github.com/google/go-cmp/cmp"
	"github.com/jmoiron/sqlx"
	ora "github.com/sijms/go-ora/v2"
)

// Datatypes は、データ型テストテーブル
type Datatypes struct {
	Col01 sql.NullString  `db:"COL01"` // varchar2
	Col02 sql.NullString  `db:"COL02"` // nvarchar2
	Col03 sql.NullInt64   `db:"COL03"` // number(10)
	Col04 sql.NullString  `db:"COL04"` // number(10,4)
	Col05 sql.NullFloat64 `db:"COL05"` // float(10)
	Col06 sql.NullString  `db:"COL06"` // long
	Col07 sql.NullTime    `db:"COL07"` // date
	Col08 sql.NullFloat64 `db:"COL08"` // binary_float
	Col09 sql.NullFloat64 `db:"COL09"` // binary_double
	Col10 OraTimeStamp    `db:"COL10"` // timestamp
	Col11 OraTimeStamp    `db:"COL11"` // timestamp with tz
	Col12 OraTimeStamp    `db:"COL12"` // timestamp with local tz
	Col13 sql.NullString  `db:"COL13"` // interval year to month
	Col14 sql.NullString  `db:"COL14"` // interval day to second
	Col15 []byte          `db:"COL15"` // raw
	// Col16 []byte          `db:"COL16"` // long raw
	Col17 sql.NullString `db:"COL17"` // rowid
	Col18 sql.NullString `db:"COL18"` // urowid
	Col19 sql.NullString `db:"COL19"` // char(10)
	Col20 sql.NullString `db:"COL20"` // nchar(10)
	Col21 sql.NullString `db:"COL21"` // clob
	Col22 sql.NullString `db:"COL22"` // nclob
	Col23 []byte         `db:"COL23"` // blob
	// Col24 []byte          `db:"COL24"` // bfile
}

// Key は、データ型テストテーブルのキー
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

	JST, _ := time.LoadLocation("Asia/Tokyo")
	key := Key{"1"}
	src := Datatypes{
		Col01: sql.NullString{String: "1", Valid: true},
		Col02: sql.NullString{String: "123456789", Valid: true},
		Col03: sql.NullInt64{Int64: 123, Valid: true},
		Col04: sql.NullString{String: "123456.1234", Valid: true},
		Col05: sql.NullFloat64{Float64: 1.25, Valid: true},
		Col06: sql.NullString{String: "abc", Valid: true},
		Col07: sql.NullTime{Time: time.Date(2001, 2, 3, 4, 5, 6, 0, time.UTC), Valid: true}, // 時刻も持つ（仕様通り）
		Col08: sql.NullFloat64{Float64: 2.5, Valid: true},
		Col09: sql.NullFloat64{Float64: 3.125, Valid: true},
		Col10: OraTimeStamp{sql.NullTime{Time: time.Date(2001, 2, 3, 4, 5, 6, 7000, time.UTC), Valid: true}}, // マイクロ秒持てない
		Col11: OraTimeStamp{sql.NullTime{Time: time.Date(2001, 2, 3, 4, 5, 6, 7000, JST), Valid: true}},      // マイクロ秒もTZも持てない
		Col12: OraTimeStamp{sql.NullTime{Time: time.Date(2001, 2, 3, 4, 5, 6, 7000, JST), Valid: true}},      // マイクロ秒もTZも持てない
		Col13: sql.NullString{String: "+01-11", Valid: true},
		Col14: sql.NullString{String: "+09 23:59:59.000001", Valid: true},
		Col15: []byte{1, 2, 3},

		Col17: sql.NullString{String: "AAARXwAAEAAAAKbAAE", Valid: true},
		Col18: sql.NullString{String: "AAARXwAAEAAAAKbAAE", Valid: true},
		Col19: sql.NullString{String: "12345678  ", Valid: true}, // 末尾はスペース
		Col20: sql.NullString{String: "12345678  ", Valid: true}, // 末尾はスペース
		Col21: sql.NullString{String: "CLOB", Valid: true},
		Col22: sql.NullString{String: "NCLOB", Valid: true},
		Col23: []byte{1, 2, 3, 4},
	}

	_, err = db.NamedExec(`
		INSERT INTO DATATYPES (
			col01, col02, col03, col04, col05, col06, col07, col08, col09, col10,
			col11, col12, col13, col14, col15,        col17, col18, col19, col20,
			col21, col22, col23
		) VALUES (
			:COL01, :COL02, :COL03, :COL04, :COL05, :COL06, :COL07, :COL08, :COL09, :COL10,
			:COL11, :COL12, :COL13, :COL14, :COL15,         :COL17, :COL18, :COL19, :COL20,
			:COL21, :COL22, :COL23
		)`,
		src,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}

	dst := Datatypes{}
	query, args, err := db.BindNamed(`
		SELECT 
			col01, col02, col03, col04, col05, col06, col07, col08, col09, col10,
			col11, col12, col13, col14, col15,        col17, col18, col19, col20,
			col21, col22, col23
		FROM DATATYPES 
		WHERE col01 = :COL01`,
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
		WHERE col01 = :COL01`,
		key,
	)
	if err != nil {
		log.Printf("db.Exec error %s", err)
	}
}

// OraTimeStamp は、Null を許容する Time
type OraTimeStamp struct {
	sql.NullTime
}

// Scan implements the Scanner interface.
func (ot *OraTimeStamp) Scan(value any) error {
	if value == nil {
		ot.Time, ot.Valid = time.Time{}, false
		return nil
	}
	ot.Valid = true
	switch temp := value.(type) {
	case ora.TimeStamp:
		ot.NullTime.Time = time.Time(temp)
	case *ora.TimeStamp:
		ot.NullTime.Time = time.Time(*temp)
	case ora.TimeStampTZ:
		ot.NullTime.Time = time.Time(temp)
	case *ora.TimeStampTZ:
		ot.NullTime.Time = time.Time(*temp)
	default:
		return ot.NullTime.Scan(value)
	}
	return nil
}

// Value implements the driver Valuer interface.
func (ot OraTimeStamp) Value() (driver.Value, error) {
	if !ot.Valid {
		return nil, nil
	}
	return ora.TimeStampTZ(ot.Time), nil
}
```

# おわりに

Oracle の各種 TimeStamp は、time.Time や sql.NullTime では１秒未満の小数部やタイムゾーンが連携できませんでしたので、sql.Scanner で driver.Valuer な OraTimeStamp を用意して、ドライバで TimeStamp 用に定義している独自型での入出力に対応しました。

日本語については、別途、テストしたいと思います。
