# nginx でバーチャルホストごとにクライアント証明書を分ける

nginx でクライアント認証が必須な複数のバーチャルホストで、サイトごとに異なる証明書するようにしてみたメモ。

クライアント証明書の発行元のルートCAを複数用意する方法と、クライアント証明書のCNを条件にアクセス制御する方法を試しました。

Docker で nginx のコンテナを作成してその中で作業します。

```sh
# nginx のコンテナを作成して中に入ります。
docker run --rm -it -p 8443:443 -v $PWD/conf:/conf -v $PWD/work:/work -w /work nginx:alpine sh

# openssl をインストールします。
apk add --no-cache openssl

# オレオレルートサーバ証明書を作成します。
openssl req -batch -new -x509 -newkey rsa:2048 -nodes -sha256 \
  -subj "/CN=*.lvh.me" -days 3650 -keyout ./server.key -out ./server.crt
```

## バーチャルホストごとにクライアント証明書のルートCAを指定

クライアント証明書を発行するルートCAを複数作成し、バーチャルホストごとに異なる発行元の証明書でクライアント認証を行います。

まず、ルートCA **aaa** を作成します。デフォルト設定ファイル `/etc/ssl/openssl.cnf` で `./demoCA` に作成されるようになっているので、`./aaa/demoCA` を作ってそこに作成します。

```sh
mkdir -p ./aaa/demoCA/
cd ./aaa/

mkdir -p ./demoCA/private
mkdir -p ./demoCA/newcerts
touch ./demoCA/index.txt
echo '00' | tee ./demoCA/serial
echo '00' | tee ./demoCA/crlnumber

# オレオレルート証明書
openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -x509 -days 3650 \
  -subj "/CN=root" -keyout ./demoCA/private/cakey.pem -out ./demoCA/cacert.pem

# 失効リスト
openssl ca -gencrl -crldays 730 -out ./demoCA/crl.pem

# クライアント証明書
openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/CN=client" -keyout client.key -out client.csr
openssl ca -batch -policy policy_anything -days 730 -in client.csr -out client.crt

cd ..
```

別のルートCA **bbb** を作成します。これは `./bbb/demoCA/` に作成します。

```sh
mkdir -p ./bbb/demoCA/
cd ./bbb/

mkdir -p ./demoCA/private
mkdir -p ./demoCA/newcerts
touch ./demoCA/index.txt
echo '00' | tee ./demoCA/serial
echo '00' | tee ./demoCA/crlnumber

# オレオレルート証明書
openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -x509 -days 3650 \
  -subj "/CN=root" -keyout ./demoCA/private/cakey.pem -out ./demoCA/cacert.pem

# 失効リスト
openssl ca -gencrl -crldays 730 -out ./demoCA/crl.pem

# クライアント証明書
openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/CN=client" -keyout client.key -out client.csr
openssl ca -batch -policy policy_anything -days 730 -in client.csr -out client.crt

cd ..
```

設定ファイルを指定して nginx を開始します。

```sh
nginx -c /conf/nginx-multi-ca.conf
```

設定ファイルの内容は次のとおりです。

- https://github.com/ngyuki-sandbox/nginx-vhost-ssl-client/blob/master/conf/nginx-multi-ca.conf

コンテナの外から nginx にアクセスして動作確認します。

```sh
# 証明書や秘密鍵のオーナーを変更。
cd work/
sudo chown -R "$USER": .

# デフォルトサイトはクライアント認証なし → this is default
curl --cacert server.crt https://xxx.lvh.me:8443/

# サイト aaa と bbb はクライアント認証必須 → 400 No required SSL certificate was sent
curl --cacert server.crt https://aaa.lvh.me:8443/
curl --cacert server.crt https://bbb.lvh.me:8443/

# サイト aaa は 証明書 aaa があれば OK → this is aaa
curl --cacert server.crt --cert aaa/client.crt --key aaa/client.key https://aaa.lvh.me:8443/

# サイト bbb は 証明書 bbb があれば OK → this is bbb
curl --cacert server.crt --cert bbb/client.crt --key bbb/client.key https://bbb.lvh.me:8443/

# サイトと証明書が異なるならダメ → 400 The SSL certificate error
curl --cacert server.crt --cert bbb/client.crt --key bbb/client.key https://aaa.lvh.me:8443/
curl --cacert server.crt --cert aaa/client.crt --key aaa/client.key https://bbb.lvh.me:8443/
```

## クライアント証明書のCNを条件にアクセス制御

ルートCAを作成。

```sh
mkdir -p common/demoCA/
cd ./common/

mkdir -p ./demoCA/private
mkdir -p ./demoCA/newcerts
touch ./demoCA/index.txt
echo '00' | tee ./demoCA/serial
echo '00' | tee ./demoCA/crlnumber

# オレオレルート証明書
openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -x509 -days 3650 \
  -subj "/CN=root" -keyout ./demoCA/private/cakey.pem -out ./demoCA/cacert.pem

# 失効リスト
openssl ca -gencrl -crldays 730 -out ./demoCA/crl.pem

# クライアント証明書
openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/CN=aaa" -keyout a0.key -out a0.csr
openssl ca -batch -policy policy_anything -days 730 -in a0.csr -out a0.crt

openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/CN=aaa/O=ore" -keyout a1.key -out a1.csr
openssl ca -batch -policy policy_anything -days 730 -in a1.csr -out a1.crt

openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/OU=ore/CN=aaa" -keyout a2.key -out a2.csr
openssl ca -batch -policy policy_anything -days 730 -in a2.csr -out a2.crt

openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/CN=aaaa" -keyout a9.key -out a9.csr
openssl ca -batch -policy policy_anything -days 730 -in a9.csr -out a9.crt

openssl req -batch -new -newkey rsa:2048 -nodes -sha256 -subj "/CN=bbb" -keyout bbb.key -out bbb.csr
openssl ca -batch -policy policy_anything -days 730 -in bbb.csr -out bbb.crt

cd ..
```

設定ファイルを指定して nginx を開始します。

```sh
nginx -c /conf/nginx-cert-dn.conf
```

設定ファイルの内容は次のとおりです。

- https://github.com/ngyuki-sandbox/nginx-vhost-ssl-client/blob/master/conf/nginx-multi-ca.conf

コンテナの外から nginx にアクセスして動作確認します。

```sh
cd work/
sudo chown -R "$USER": .

# デフォルトサイトはクライアント認証なし → this is default
curl --cacert server.crt https://xxx.lvh.me:8443/

# サイト aaa と bbb はクライアント認証必須 → 400 No required SSL certificate was sent
curl --cacert server.crt https://aaa.lvh.me:8443/
curl --cacert server.crt https://bbb.lvh.me:8443/

# サイト aaa に CN=aaa の証明書でアクセス → this is aaa
curl --cacert server.crt --cert common/a0.crt --key common/a0.key https://aaa.lvh.me:8443/
curl --cacert server.crt --cert common/a1.crt --key common/a1.key https://aaa.lvh.me:8443/
curl --cacert server.crt --cert common/a2.crt --key common/a2.key https://aaa.lvh.me:8443/

# CN=aaaa なのでダメ → 403 Forbidden
curl --cacert server.crt --cert common/a9.crt --key common/a9.key https://aaa.lvh.me:8443/

# サイト bbb に CN=bbb の証明書でアクセス → this is bbb
curl --cacert server.crt --cert common/bbb.crt --key common/bbb.key https://bbb.lvh.me:8443/

# サイトと証明書が対応していない → 403 Forbidden
curl --cacert server.crt --cert common/bbb.crt --key common/bbb.key https://aaa.lvh.me:8443/
curl --cacert server.crt --cert common/a0.crt --key common/a0.key https://bbb.lvh.me:8443/
```
