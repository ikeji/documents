# rpipe

- 複数のマシン間でパイプ通信をしたい。  
  ex. ```src-cmd | ssh remote-host dst-cmd```
- TCPが切断されても、そこから再開できるようにしたい。

## protocol

### ハンドシェイク

双方

- バナー
- プロトコルバージョン

受信側から送信側へ

- 開始位置(64bit)

### 送信側から受信側へ

生データをそのまま。

### 受信側から送信側へ

ACK

- Type: ack (1byte)
- 受信完了位置(64bit)

NACK (これ以降は受信できない事を示す)

- Type: nack (1byte)

TODO: 切断と終了ってリモート側で区別できたっけ？

## command

```
rpipe remote-host dst-cmd
rpipe-client --token [uuid] --command 'dst-cmd'
rpipe-server --token [uuid] --command 'dst-cmd'
```

## connect

1. コマンドが実行される。 ```src-cmd | rpipe remote-host dst-cmd```
2. rpipeはトークンを乱数で生成する。
3. rpipeコマンドは、SSHを実行する。  
   ```ssh remote-host 'rpipe-client --token [token] --command "dst-cmd"'```
4. リモートマシンで実行されたrpipe-clientコマンドはrpipe-serverを実行する。  
   ```rpipe-server --token [token] --command 'dst-cmd'```
5. rpipe-serverは、tokenを元にunixドメインソケットを作り、それ以降、rpipe-clientとの間はそれを使って通信する。
6. rpipe-serverは、dst-cmdを実行し通信する。

## send

1. src-cmdの出力は、rpipeの入力にはいる。
2. rpipeはsshごしに、rpipe-clientにデータを送る。この時、送ったデータは再送にそなえて取っておく。
3. rpipe-clientはunixドメインソケットごしにrpipe-serverに送る。
4. rpipe-serverは、dst-cmdにデータを送る。書き込みバッファにデータを置けたら、ackをrpipeに返す。
5. rpipeはackが来たらそのデータを捨てる。

## re-connect

1. rpipeコマンドは、sshコマンドの終了で切断を知る。
2. rpipeコマンドは、再度SSHを実行する。再接続時は、commandは渡さない(渡しても無視される)。
   ```ssh remote-host 'rpipe-client --token [token]'```
3. rpipe-clientは、残ってるはずのunixドメインソケットに接続する。unixドメインソケットが消失していたら、パイプが破壊されたものとして扱う。
4. 再接続を受けたrpipe-serverは、ハンドシェイクの後、自分がどこまでdst-cmdに渡したかをrpipeコマンドに伝える。
5. rpipeは指定されたバイト数以降を送りなおす。

