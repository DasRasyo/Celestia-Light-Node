# Celestia Light Node Kurulum Rehberi

## Sistem Gereksinimleri

Memory: 2 GB RAM <br/>
CPU: Single Core <br/>
Disk: 25 GB SSD Storage <br/>
Bandwidth: 56 Kbps for Download/56 Kbps for Upload <br/>


## Sistem güncellemerini yaparak başlıyoruz.

```
sudo apt update && sudo apt upgrade -y

sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential \
git make ncdu -y
```

## Go kurulumu ile devam ediyoruz.

```
ver="1.18.2"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
go versiyon komutunun çıktısı şu şekilde olmalıdır:

```
go version go1.18.2 linux/amd64
```

## Celestia Node Kurulum

```
cd $HOME
rm -rf celestia-node
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node
git checkout tags/v0.3.0-rc2
make install
```

Celestia versiyon kontrolü:

```
celestia version
```

Çıktı şu şekilde olmalı:

```
Semantic version: v0.3.0-rc2
Commit: 89892d8b96660e334741987d84546c36f0996fbe
```

## Node'u başlatalım

```
celestia light init
```

Şuna benzer bir çıktı almalısınız:

```
2022-09-15T11:42:11.103+0200    INFO    node    node/init.go:26 Initializing Light Node Store over '/root/.celestia-light-arabica'
2022-09-15T11:42:11.159+0200    INFO    node    node/init.go:62 Saving config   {"path": "/root/.celestia-light-arabica/config.toml"}
2022-09-15T11:42:11.159+0200    INFO    node    node/init.go:67 Node Store initialized
```

## Cüzdan oluşturma

```
make cel-key
./cel-key add <cüzdan-adınız> --keyring-backend test --node.type light
```

<cüzdan-adınız> kısmını, cüzdanınıza vermek istediğiniz ad ile değiştiriyorsunuz.
Bu komut cüzdanınızı oluşturacak, mnemonic ve cüzdan adresinizi çıktı olarak verecektir. Bunları kaydetmeyi unutmayın.

## Cüzdan adresinizi ve adınızı şu kod ile görüntüleyebilirsiniz:

```
./cel-key list --node.type light --keyring-backend test
```

## Cüzdan adresinize Celestia Discord sunucusundaki #faucet kanalından test tokenı almayı unutmayın.

## Servis oluşturmayla devam ediyoruz.

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-lightd.service
[Unit]
Description=celestia-lightd Light Node
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/celestia light start --core.grpc https://rpc-mamaki.pops.one:9090 --keyring.accname <cüzdan-adınız>
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```

NOT: ExecStart satırının sonundaki alanı kendi cüzdan adınız ile değiştirmeyi unutmayın!

Aşağıdaki kodlarla node'umuzu başlatıyoruz.

```
systemctl enable celestia-lightd
systemctl start celestia-lightd
```

Servis Kontrolü

```
systemctl status celestia-lightd
```

Log kontrolü

```
journalctl -u celestia-lightd.service -f
```

NOT: Log kontrol kodunu girdiğinizde `No journal files were found` hatası alırsanız aşağıdaki kodu girmelisiniz:

```
sudo systemctl restart systemd-journald
```

Restart:

```
systemctl start celestia-lightd
```

Durdurma:

```
systemctl stop celestia-lightd
```

Celestia Light Node'unuz, headerleri senkronize etmeye başladı. Senkronizasyon tamamlandıktan sonra, Light Node, Bridge Node'dan Data Availability Sampling (DAS) yapacaktır.

## Cüzdan bakiyesi sorgulama:

```
curl -X GET http://127.0.0.1:26658/balance
```

## Blok header bilgisi alma:

```
curl -X GET http://127.0.0.1:26658/header/1
```

NOT: Hangi bloğun bilgisini almak istiyorsanız kodun sonundaki 1 yerine onu yazmanız gerekiyor.

## PayForData tx gönderme:

Aşağıda verilen değerler hazır değerlerdir. Sizler şu siteden istediğiniz değeri vererek kendinize göre sorgulama yapabilirsiniz.
(https://go.dev/play/p/7ltvaj8lhRl)

```
curl -X POST -d '{"namespace_id": "0c204d39600fddd3",
  "data": "f1f20ca8007e910a3bf8b2e61da0f26bca07ef78717a6ea54165f5",
  "gas_limit": 60000}' http://localhost:26658/submit_pfd
```

Bu kodu girdğinizde `"rpc error: code = NotFound desc = account XXXXXXXXXXXXXXX not found"` hatası alırsanız cüzdanınızda yeterli miktarda coin yok demektir. Explorerdan kontrol edin. Cüzdanınızda yeterli coin olduğundan emin olun.

Pay For Data işleminizi gönderdikten sonra, başarılı olduğunda node, Pay For Data işleminin dahil edildiği blok yüksekliğini çıktı olarak verir. 
Ardından, mesaj paylaşımlarınızın size geri dönmesini sağlamak için bu blok yüksekliğini ve Pay For Data işleminizi gönderdiğiniz namespace_id'yi kullanabilirsiniz. Aşağıdaki örnekte çıktıda verilen blok yükseliğinin 214568 olduğunu varsayalım. 

```
curl -X GET \
  http://localhost:26658/namespaced_shares/0c204d39600fddd3/height/214568
```

NOT: Yukarıdaki kodun sonundaki `214568` blok yüksekliğini Pay For Data için gönderdiğiniz kodun çıktısındaki blok yüksekliği ile değiştirmelisiniz.

Şuna benzer bir çıktı gelecektir(çıktı vermesi uzun sürebilir):

```
{
   "shares":[
      "DCBNOWAP3dMb8fIMqAB+kQo7+LLmHaDya8oH73hxem6lQWX1AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=="
   ],
   "height":214568
}
```
NOT: Cüzdan oluşturma ve Pay For Data tx gönderme işlemleri opsiyoneldir. Cüzdan oluşturma adımını uygulamadan geçerseniz, node başlatıdığında otomatik olarak bir cüzdan oluşturacaktır. Bu cüzdan adresine ve adına `./cel-key list --node.type light --keyring-backend test` ile ulaşabilirsiniz.
