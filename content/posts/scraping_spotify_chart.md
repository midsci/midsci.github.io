---
title: "Spotify Chart のスクレイピング"
date: 2022-09-24T23:42:53+09:00
draft: false
toc: false
images:
tags:
  - method
  - spotify
  - python
  - selenium
---

## はじめに

本記事では、Google Chrome を操作するための `Python` のライブラリ `selenium` を用いて、`Spotify Chart` から約過去5年分の流行曲の情報をスクレイピングします。

## Spotify Charts
`Spotify Chart` をご存じの方は読み飛ばしていただいて構いません。
`Spotify Chart` とは、音楽ストリーミングサービス `Spotify` が提供する、同サービス内で再生された回数や、他のSNS等でシェアされた回数を基に作った、独自のランキングデータです。
集計する時間単位、また集計方法で、それぞれ２種類あるので紹介します。

**時間単位**
ランキングを集計するにあたり、`Spotify` では、日付ベースと週ベースで集約を行っているようです。
ここで、日付ベースのことを `daily`、週ベースのことを `weekly` と呼ぶことにします。

---
注意点として、本記事では、ブラウザ上でページに存在するダウンロードボタンを押す動作を `Python` で実装します。
ただ、`weekly` データに関してはページ上にダウンロードボタンが存在しないため、この記事の内容では `weekly` データをスクレイピングすることはできません！

---

**集計方法**
`Spotify` では、ランキングを作成する手法として、再生数ベースのものと、SNS等でシェアされた回数を基に決定する手法の２通りのランキングを公開しています。
再生数ベースのものは、単純にその当時流行している音楽や、今現在流行している音楽についての情報をえるのに有用だと考えられます。
一方で、SNS等のシェアベースのランキングではこれから流行るであろう音楽の探索に有用だと考えられます。

再生数ベースのランキングを `regional` (再生数のカウントが地域ないし、各国コードで行われているため)、またSNS等でのシェアベースのランキングを `viral` と呼びます。

これら２つを組み合わせて、例えば、「過去の `viral`→`regional`に出現する音楽の傾向」をうまくモデリングすることで、これから流行する音楽の予測モデルができたりするのではないでしょうか。

### エリアについて
`Spotify` がサービスを提供しているのはもちろん日本だけではありません。
`Spotify Chart` 内で日本の国コードが `jp` であるように、他の国に関するデータも取得することができます。
また、 `global` という国コードを指定すると（実際には国コードではないが）全世界でのランキングを取得することができます。

本記事では、簡単のため、日本のデータに限ってスクレイピングを行います。

## 手法
方法としては至ってシンプルです。
`Google Chrome` で 「`Spotify Chart` のサイトを開き、ログインをし、自動でランキングデータを取得する。」という流れを愚直に実装します。

以下に必要なパッケージを示します。
不足している場合は適宜インストールしましょう。

```python
# パッケージのインポート
import time
import datetime
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
import chromedriver_binary
```

まずは、`Google Chrome` を開いて、`Spotify Chart` へアクセスします。

```python
# option の設定
options = webdriver.chrome.options.Options()
driver = webdriver.Chrome(options=options)

# spotify にログインするためのメールアドレス（アカウント名）とパスワード
LOGIN_ID = 'MAIL-ADDRESS'
LOGIN_PASSWORD = 'PASSWORD'

# spotify-chart に直接遷移しようとすると
# Log in ページに飛ばされる

driver.get('https://charts.spotify.com/home')

time.sleep(1.0)
```

次に、ブラウザ上でログインボタンをクリックし、ログイン用のメールアドレス、パスワードを入力し、ログインボタンを押します。

```python
# Log in ボタンをクリック
driver.find_element(By.PARTIAL_LINK_TEXT, "Log in").click()
time.sleep(1.0)

# username を入力
driver.find_element(By.ID, 'login-username').send_keys(LOGIN_ID)
time.sleep(1.0)

# password を入力
driver.find_element(By.ID, 'login-password').send_keys(LOGIN_PASSWORD)
time.sleep(1.0)

# Log in ボタンをクリック
driver.find_element(By.ID, 'login-button').click()
```

ここで、`Spotify` にログインできた状態になります。
ここからアドレス操作で、自動的にページ遷移→ランキングデータをダウンロードする、という処理を繰り返します。
日付を設定しておきます。
日付は `YYY-MM-DD` の形式で措定することができます。

```python
# スクレイピングする期間を設定
# 2017年１月１日〜現在の3日前の日付までがスクレイピング可能

today = datetime.date.today()
end = today - datetime.timedelta(days=2)
end = str(end.year) + '-' + str(end.month).zfill(2) + '-' + str(end.day).zfill(2)
```

さらに、`viral`か`regional`を格納したリスト、国コードを指定するためのリストを定義しておきます。

```python
# 種別（region が再生数ベースのランキング、viralがSNSのシェア等ベースのランキング）
# 国名をc_namesに格納

r_names = ['viral', 'regional']
c_names = ['global', 'jp']

# 以下、指定可能な国コード
# 'ar', 'au', 'at', 'by', 'be', 'bo', 'br', 'bg', 'ca', 'cl', 'co', 'cr', 
# 'cy', 'cz', 'dk', 'do', 'ec', 'eg', 'sv', 'ee', 'fi', 'fr', 'de', 'gr',
# 'gt', 'hn', 'hk', 'hu', 'is', 'in', 'id', 'ie', 'il', 'it', 'jp', 'kz',
# 'lv', 'lt', 'lu', 'my', 'mx', 'ma', 'nl', 'nz', 'ni', 'ng', 'no', 'pk',
# 'pa', 'py', 'pe', 'ph', 'pl', 'pt', 'ro', 'sa', 'sg', 'sk', 'za', 'kr',
# 'es', 'se', 'ch', 'tw', 'th', 'tr', 'ae', 'ua', 'gb', 'uy', 'us', 've', 'vn'
```

### 環境について

Python を使ってスクレイピングを行います。
筆者の手元では `Python==3.8` です。
また、ログイン認証は `selenium` という、`Python` でブラウザを操作するライブラリが必須となります。
`selenium` については、手元の `Chrome` とのバージョンの整合性がとれないとエラーを吐いちゃったりするので、気をつけてインストールしてください。
これについては例えば[こちらの記事]()が参考になります。

### コード

### コード解説

### スクレイピングの様子

## まとめ



これまでは、`Spotify` にログインせずとも、`csv` ファイルとして、過去五年間分の `Spotify Chart` データ（週単位、もしくは日単位での`Spotify`での人気ランキング上位）を取得することが可能でした。
最近になって、