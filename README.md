# Laughter Collector

音声データから笑い声を抽出してデータセットを作成するためのスクリプト群です。また非言語音声や感嘆詞の抽出も可能です（が誤りが多いかもしれません）。

## 原理

- 音声ファイルを読み込み、-40dBを無音とみなしスライス
- スライスした音声データに対して、Whisperで書き起こしを行う
- 書き起こしデータに対して、正規表現を用いて笑い声かどうか・非言語音声や感嘆詞かどうかを判定

## 使い方

Python>=3.10が必要です。

### インストール

```bash
git clone https://github.com/litagin02/laughter-collector
cd laughter-collector
python -m venv venv
venv\Scripts\activate
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
pip install -r requirements.txt
```

### 元データの準備

- 音声ファイルたちをディレクトリ（以下`path/to/original_data`とする）に格納
- 各ファイルの拡張子は".wav", ".mp3", ".flac", ".ogg", ".opus"のいずれかである必要があります（必要に応じて`utils.py`を書き換えれば他もいけます）。
- `path/to/original_data`では好きなようにサブディレクトリたちの階層を作ってそこに音声ファイルを格納してください。**デフォルトでは直下の音声ファイルは反映されず、1つ下の階層からのみ反映されます**。結果は相対パスを保持したまま指定した`path/to/output`に保存されます。

### データセットの作成

```bash
python collect_laughter.py -i path/to/original_data -o path/to/output
```

裏では`split.py`が呼び出され、マルチプロセスで音声を切り出して`path/to/output/temp`にスライスされた音声が保存されて行き、それを本体のスクリプトが順次読み込んで書き起こしをバッチ処理で行います。

### 結果

元々の音声ファイルが以下のような構造だったとします。
```
path/to/original_data
├── subdir1
│   ├── foo.wav
│   ├── bar.wav
│   └── baz.wav
└── subdir2
    ├── subdir3
    │   └── qux.wav
    └── quux.wav
```

結果は以下のような構造になります。
```
path/to/output
├── laugh
│   ├── subdir1
│   │   ├── foo_0.wav
│   │   ├── foo_1.wav
│   │   ├── bar_0.wav
│   │   └── baz_0.wav
│   └── subdir2
│       ├── subdir3
│       │   └── qux_0.wav
│       └── quux_0.wav
├── nv
│   ├── subdir1
│   │   ├── foo_2.wav
| ...
└── trans
    ├── subdir1
    │   └── trans.csv
    └── subdir2
        ├── subdir3
        │   └── trans.csv
        └── trans.csv
```

ここで`foo_0.wav`は`foo.wav`の0番目のスライスを示しています。`trans.csv`は書き起こしデータです。

### 注意

- 笑い声の判定と非言語音声の判定では笑い声が優先されます。
- 非言語音声の判定は誤りが多く、特定のひらがなから単語ができてしまう場合それが非言語音声として判定されることがあります。結果を見ながら、そのような単語を`exclude_words.txt`に追加してください（毎回このファイルが参照されるので、スクリプト実行中でも変更が反映されます）。
- スライスの細かいパラメータや、笑い声正規表現等は、それぞれ`split.py`、`pattern.py`内を参照しつつ必要ならば変更してください。
- デフォルトではHugging FaceのWhisperのmediumモデルが使われます（笑い声や非言語音声かどうかさえ判定できればよく書き起こし精度はそこまで必要がない）、が必要に応じて`collect_laughter.py`の引数`--model`でモデルを指定できます。細かい他の引数等はコードを参照してください。

## その他のスクリプト

結果の笑い声ファイルたちに対するFaster Whisper (large-v2) での書き起こし：
```bash
python transcribe.py -i path/to/output/laugh -o transcriptions.csv
```
