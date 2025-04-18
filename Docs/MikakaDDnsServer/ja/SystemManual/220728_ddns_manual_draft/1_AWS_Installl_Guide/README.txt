﻿開発中の Mikaka DDNS サーバーの暫定構築手順書

2022/8/05 登 大遊

本ドキュメントは、開発中の Mikaka DDNS サーバーの暫定構築手順書である。

ひとまず、本日時点のドラフトを、テキストファイルとして書いておいたものである。

本ドキュメントは、今後、PowerPoint 等を用いて、よりわかりやすく清書する予定である。


■ AWS でインスタンスを 1 つ作成する。
Ubuntu Server 20.04 LTS (HVM), SSD Volume Type
x86 (64-bit)
t2.medium

パブリック IPv4, IPv6 を割り当てる。
※ VPC は、あらかじめ IPv4, IPv6 の両方に対応させておく必要がある。VPC の設定については、AWS の標準的な機能であるので、説明は省略する。

セキュリティグループの設定
IPv4, IPv6 ともインターネットに対して
TCP 22 (ssh), http (80), https (443), MSSQL (1433) ※ 一時的な管理用
UDP 53
を開く。

ストレージ容量を 60GB くらいにする。 (ログとかも溜まるので)

Elastic IP アドレスで 1 つグローバル IPv4 アドレスを固定で割り当てて、上記のインスタンスに関連付ける。


■ インスタンスの Linux 環境をセットアップする
グローバル IPv4 アドレスに対して、SSH でログインする。

ip a

コマンドで、グローバル IPv6 アドレスが割り当てられていることを確認する。

以下のコマンドを実行し、タイムゾーンを JST (日本標準時) に変更する。

sudo timedatectl set-timezone Asia/Tokyo

まず、以下のコマンドを実行し、Swap が存在するかどうか確認する。

free

AWS では、Swap が存在しない可能性が高い。以下のように、「Swap: 0」と表示されていれば、Swap が存在しないことを意味する。

              total        used        free      shared  buff/cache   available
Mem:        4028136      822244     1555544         796     1650348     2968092
Swap:             0           0           0

Swap が存在しなければ、以下のコマンドを実行し、Swap 領域を作成する。(ディスクへの書き込みが発生するため、数分かかる場合がある。)

sudo dd if=/dev/zero of=/swapfile bs=16M count=250
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo cp -n /etc/fstab /etc/fstab.original

sudo dd of=/etc/fstab oflag=append conv=notrunc <<EOF
/swapfile swap swap defaults 0 0
EOF


上記のとおり実行した後、sudo reboot で再起動をし、free コマンドを再度実行して、Swap が作成され有効化されたことを確認する。


■ .NET SDK 6.0.106 のインストールを行なう。
以下の黒枠内のコマンド (2 行) を、そのままコピー & ペーストして、SSH のコンソールに貼り付ける。



sudo bash -c "bash -x <(curl --insecure --pinnedpubkey sha256//lvnOVgA0u06WySztudkn+urQda/zFBRd65A5wCmcBpQ= \
  --raw https://static.lts.dn.ipantt.net/d/210111_002_microsoft_dotnet_sdk_12255/220624_install_dotnet_6.0.106.sh)"


自動的にバイナリファイルがダウンロードされ、インストールが実行される。

最後に

“Install .NET OK !!”

と表示されれば、完了である。

.NET SDK のインストールが正常に完了していることを確認する。

以下のコマンドを実行する。

dotnet --info

以下のように表示されれば OK である。

.NET SDK (reflecting any global.json):
 Version:   6.0.106
 Commit:    ★★★
(略)
.NET SDKs installed:
  6.0.106 [/usr/share/dotnet/sdk]
.NET runtimes installed:
  Microsoft.AspNetCore.App ★.★.★ [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
  Microsoft.NETCore.App ★.★.★ [/usr/share/dotnet/shared/Microsoft.NETCore.App]

■ SQL Server 2019 Express のインストールと初期設定

以下のコマンドを実行する。SQL Server 2019 がインストールされる。(数分間時間がかかる)

wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -

sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-2019.list)"

sudo apt-get -y update && sudo apt-get -y install mssql-server

以下のコマンドを実行し、SQL Server Express 2019 を初期設定する。

sudo /opt/mssql/bin/mssql-conf setup


--- 設定例 ---
  1) Evaluation (free, no production use rights, 180-day limit)
  2) Developer (free, no production use rights)
  3) Express (free)
  4) Web (PAID)
  5) Standard (PAID)
  6) Enterprise (PAID) - CPU Core utilization restricted to 20 physical/40 hyperthreaded
  7) Enterprise Core (PAID) - CPU Core utilization up to Operating System Maximum
  8) I bought a license through a retail sales channel and have a product key to enter.
Enter your edition(1-8):
3 と入力する。

Do you accept the license terms? [Yes/No]:
yes と入力する。

Enter the SQL Server system administrator password:
SqlNeko123
SqlNeko123  と 2 回仮のパスワードを入力する。(別のパスワードでもよい。)
Confirm the SQL Server system administrator password: 
Configuring SQL Server...

The licensing PID was successfully processed. The new edition is [Express Edition].
ForceFlush is enabled for this instance. 
ForceFlush feature is enabled for log durability.
Created symlink /etc/systemd/system/multi-user.target.wants/mssql-server.service → /lib/systemd/system/mssql-server.service.
Setup has completed successfully. SQL Server is now starting.
--- 設定例ここまで ---

これで、SQL Server 2019 Express のインストールは完了した。



■ SQL Server 2019 Express への作業用端末からの接続と初期データベース構築
作業用端末 (手元の Windows 端末) 上で、Google 等の検索エンジンを開き、
“SQL Server Management Studio”
を検索し、Microsoft の Web サイトから最新版をダウンロードして作業用端末にインストールする。

SQL Server Management Studio を起動し、サーバー名として先ほど割当てられたグローバル IPv4 アドレスを指定する。

「認証」は「SQL Server 認証」とし、「ログイン」は「sa」、「パスワード」は「SqlNeko123」とする。
これにより、AWS 上のコントローラサーバー上で動作している SQL Server 2019 Express に管理モードで接続できることを確認する。


以下の方法で、初期データベースを構築する。

まず、手元の作業用端末で

https://github.com/IPA-CyberLab/IPA-DN-Cores/archive/refs/heads/master.zip

をダウンロードし、空のディレクトリに展開する。

この ZIP の中の

IPA-DN-Cores-master/Cores.NET/Cores.Basic/Database/Lib/Misc/HadbSqlScripts/

というディレクトリに、以下の 3 つのファイルがある。

1_Create_Database.sql
2_Create_Users.sql
3_Create_Tables.sql

これらのファイルを 1 つずつテキストエディタ (メモ帳等でも OK) で開き、先に接続し起動状態になっている「SQL Server Management Studio」の「新しいクエリ」をクリックして開くクエリ入力ボックスにペーストして、「実行」ボタンまたは「F5」キーでクエリを実行する。

1、2、3 の順番に実行すること。

※ 3 で、「Warning! The maximum key length for a nonclustered index is 1700 bytes.」という警告が表示されるが、問題無い。


データベース作成スクリプトを順に実行し終わったら、「SQL Server Management Studio」の「オブジェクストエクスプローラ」に新しいデータベース「HADB001」が出現しているはずである。(作成直後は画面を更新するまでオブジェクトが表示されない場合がある。「データベース」フォルダをクリックしてから、「F5」で更新する必要がある場合がある。)



■ DDNS サーバーのインストール
Linux サーバー上の SSH で、以下のコマンドを実行する。
1 行ずつ実行し、エラーが出ていないかどうかよく確認してから、次を実行すること。

sudo mkdir -p /data1/MikakaDDnsServerDaemon/

sudo chmod 777 /data1/MikakaDDnsServerDaemon/

cd /data1/MikakaDDnsServerDaemon/

git clone --recursive https://github.com/IPA-CyberLab/IPA-DN-Cores.git

cd /data1/MikakaDDnsServerDaemon/IPA-DN-Cores/
git checkout e051664075c3f805be40f03ad9b894893352bc50

#### 初期設定ファイルを書き込み
mkdir -p /data1/MikakaDDnsServerDaemon/IPA-DN-Cores/Cores.NET/Dev.Test/Local/App_TestDev/Config/
cat <<\EOF > /data1/MikakaDDnsServerDaemon/IPA-DN-Cores/Cores.NET/Dev.Test/Local/App_TestDev/Config/MikakaDDnsService.json

{
  "DDns_UdpListenPort": 53,
  "HadbSystemName": "MIKAKA_DDNS",
  "HadbOptionFlags": "None",
  "HadbSqlServerHostname": "127.0.0.1",
  "HadbSqlServerPort": 1433,
  "HadbSqlServerDisableConnectionPooling": false,
  "HadbSqlDatabaseName": "HADB001",
  "HadbSqlDatabaseReaderUsername": "sql_hadb001_reader",
  "HadbSqlDatabaseReaderPassword": "sql_hadb_reader_default_password",
  "HadbSqlDatabaseWriterUsername": "sql_hadb001_writer",
  "HadbSqlDatabaseWriterPassword": "sql_hadb_writer_default_password",
  "HadbBackupFilePathOverride": "",
  "HadbBackupDynamicConfigFilePathOverride": "",
  "LazyUpdateParallelQueueCount": 32,
  "DnsResolverServerIpAddressList": [
    "8.8.8.8",
    "1.1.1.1"
  ]
}

EOF


### Daemon としてインストール

cd /data1/MikakaDDnsServerDaemon/IPA-DN-Cores/Cores.NET/Dev.Test/ && sudo dotnet run -c Release MikakaDDnsServerDaemon install

※ 初回は、1 分くらいかかることがある。

### デーモン有効化
sudo systemctl enable MikakaDDnsServer

### デーモン開始
sudo systemctl start MikakaDDnsServer

### DDNS サーバー設定画面を開いて動作を確認する
Web ブラウザで

http://サーバーの IPv4 アドレス/

を開く。JSON-RPC とかいう画面が表示されれば、ひとまず、正常動作している。

同様に、IPv6 アドレスでもアクセスしてみよう。

http://[サーバーの IPv6 アドレス]/

を開く。JSON-RPC とかいう画面が表示されれば、ひとまず、正常動作している。



■ ドメイン取ってくる
小遣いで VALUE DOMAIN とかお名前 .com とかで安価な .com とか .net ドメインを 1 つ取ってきて試す。

以下、取ってきたドメイン名を

「abc_ddns.com」

と仮定する。

ドメイン業者のコントロールパネルなどで、abc_ddns.com について以下のように設定をする。

・ ドメイン業者の提供する DNS ゾーンサービスは、利用しない。
・ ns01 と ns02 という名前の 2 つのネームサーバー (NS) をドメインに対して設定する。
・ つまり、ns01.abc_ddns.com と ns02.abc_ddns.com という名前の NS を設定するということである。
・ ここで、ns01.abc_ddns.com と ns02.abc_ddns.com という NS ホスト名をドメインに Glue レコードとして設定するが、この際に、IP アドレスを指定する。
・ ns01.abc_ddns.com と ns02.abc_ddns.com ともに、現在立ち上げているこの Linux サーバーのグローバル IPv4 アドレスを指定する。
・ これにより、現在立ち上げているこの Linux サーバーが、abc_ddns.com の権威 DNS ゾーンサーバーとして機能することになる。
・ 2 つ作成しないといけない理由は、ほとんどの親ドメインの DNS サーバーでは、子ドメインの NS として 2 つ存在しなければ、ゾーンが有効にならないようになっているためである。




■ DDNS サーバーのセットアップ

http://サーバーの IPv4 アドレス/admin_config/

ID: USERNAME_HERE
PW: PASSWORD_HERE

いずれも初期 ID と初期 PW である。

組み込みエディタのようなものが表示される。
以下のように設定を置換する。

# DDNS ドメイン名
# 初期状態で以下のようになっているが
DDns_DomainName                                      ddns_example.net
DDns_DomainName                                      ddns_example.org
DDns_DomainName                                      ddns_example.com
DDns_DomainNamePrimary                               ddns_example.net
#  ↓
# これを以下のようにする。DDns_DomainName は 1 つで良い。
DDns_DomainName                                      abc_ddns.com
DDns_DomainNamePrimary                               abc_ddns.com

# 初期状態で以下のようになっているが
DDns_Protocol_SOA_MasterNsServerFqdn                 ns01.ddns_example.net
#  ↓
# これを以下のようにする。
DDns_Protocol_SOA_MasterNsServerFqdn                 ns01.abc_ddns.com



# 初期状態で以下のようになっているが
DDns_StaticRecord                                    A ns01 1.2.3.4
DDns_StaticRecord                                    A ns02 1.2.3.4
DDns_StaticRecord                                    A @ 1.2.3.4 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord                                    A v4 1.2.3.4 ! health_check_url=https://<THIS_IP>/health_check/
#  この「1.2.3.4」という部分を、現在立ち上げているこの Linux サーバーのグローバル IPv4 アドレスに置換する。

# 初期状態で以下のようになっているが (IPv6 を利用する場合)
DDns_StaticRecord                                    AAAA @ 1111:2222:3333::4444  ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord                                    AAAA v6 1111:2222:3333::4444 ! health_check_url=https://<THIS_IP>/health_check/
#  この「1111:2222:3333::4444」という部分を、現在立ち上げているこの Linux サーバーのグローバル IPv6 アドレスに置換する。


# 初期状態で以下のようになっているが
Service_FriendlyName                                 Dev.Test
#  ↓
# これを以下のようにする。
Service_FriendlyName                                 俺様のＤＤＮＳサービス


この設定を「Apply Now」をクリックして保存する。

上記の設定が完了したら、その時点で、abc_ddns.com の権威サーバーとして、この立ち上げ中の DDNS サーバーが機能しているはずである。


手元の端末などで

dig a abc_ddns.com

dig aaaa abc_ddns.com

などと入れて、この DDNS サーバーの IPv4, IPv6 アドレスが返れば成功である。

そして、試しに

http://abc_ddns.com/

に HTTP でアクセスしてみる。これでうまくいけば、次に、

https://abc_ddns.com/

に HTTPS でアクセスしてみる。この HTTPS のアクセスは、最初は証明書エラーになるが、バックグラウンドで Let's Encrypt で証明書の自動取得が行なわれ、正常に完了すれば、約 1 分後にアクセスすると、Let's Encrypt の証明書が適用され、SSL 警告は表示されなくなっていることが分かる。 ※ Chrome などの Web ブラウザの場合、SSL セッションが期限切れになるまでは、SSL 証明書エラーが表示され続ける可能性がある。この場合は、Web ブラウザのすべてのプロセスを一度終了して再起動するか、「シークレットモード」のようなもので別のプロセスとして立ち上げるかすると、すぐに正常性を確認できる。

この状態で

http://abc_ddns.com/control/DDNS_Host/

からいくつかポンポンと DDNS ホストを作成してみる。
たとえば superman というホストを作成する。
するとなんと、作成直後から

superman.abc_ddns.com

に対して名前解決ができるようになることが分かる。これは、とても快調な気分になる。試しに何個かやってみるとよい。



■ その他 拡張1
『IPA_DN_WildcardCertServerUtil (Python で書かれた Let's Encrypt ワイルドカード DNS 用証明書自動更新・証明書ファイル提供サーバー)』
https://github.com/IPA-CyberLab/IPA_DN_WildcardCertServerUtil/

を、この DDNS ドメイン名に対して稼働させたい場合は、次のようにする。

1. https://github.com/IPA-CyberLab/IPA_DN_WildcardCertServerUtil/ を AWS などの別の新しい Ubuntu Linux サーバーにセットアップする。この途中で、グローバル IPv4 アドレスが定まるはずである。

2. DDNS サーバーの設定エディタで、

DDns_StaticRecord                                    A ssl-cert-server 4.3.2.2
DDns_StaticRecord                                    A ssl-cert-server-v4 4.3.2.1
DDns_StaticRecord                                    AAAA ssl-cert-server 2001:af80::4321
DDns_StaticRecord                                    AAAA ssl-cert-server-v6 2001:af80::4321


という部分がある。ここの
「4.3.2.2」というデフォルト設定のダミーの IPv4 アドレスを IPA_DN_WildcardCertServerUtil のサーバーの IPv4 アドレスに設定する。

同様に、ここの
「2001:af80::4321」というデフォルト設定のダミーの IPv4 アドレスを abc_ddns.com のサーバーの IPv6 アドレスに設定する。

3. これにより、IPA_DN_WildcardCertServerUtil で Let's Encrypt による証明書自動取得を実施することができるようになり、*.abc_ddns.com というホスト名のワイルドカード SSL 証明書の取得を行ない、その証明書を Web サーバーで提供し続けることができるようになる。


■ その他 拡張 2
DDNS の電子メールによるシークレットキー回復サービスを利用するためには、DDNS の設定エディタで

---
Service_SendMail_SmtpServer_Hostname                 your-smtp-server-address.your_company.org

Service_SendMail_SmtpServer_Port                     25

Service_SendMail_SmtpServer_EnableSsl                false

Service_SendMail_SmtpServer_Username                 

Service_SendMail_SmtpServer_Password                 

Service_SendMail_MailFromAddress                     noreply@your_company.org
---

とある部分を、適切に設定する。


■ その他 拡張 3
DDNS サーバーは、2 ～ 4 台くらいに増加させることができる。
そして、親 DNS サーバーに複数の NS を登録しておき、DDNS サーバーの設定ファイルの DDns_StaticRecord にもこれに応じた登録を適切に行なっていれば、DDNS サーバーの冗長が実現できる。
この場合、データベースだけは共通の 1 台にする必要がある。

上の説明で、

/data1/MikakaDDnsServerDaemon/IPA-DN-Cores/Cores.NET/Dev.Test/Local/App_TestDev/Config/MikakaDDnsService.json

の HadbSqlServerHostname として記載するデータベースホスト名をリモートのデータベースサーバーに設定する。

制約事項は、次のとおりである。

(1) DDNS サーバーの権威 DNS サーバーとしての機能 (ゾーンサーバーとしての機能) を複数台に分散することについて考える。NS レコードを複数登録し、クラウド上などで動作させる各インスタンスの IP アドレスを指定していれば、DNS の仕組みにより、1 つの NS への問い合わせが失敗したら別の NS への問い合わせが行なわれるので、一部の DDNS サーバーが停止しても、名前解決は停止しない。したがって、複数のインスタンスのうち一部のインスタンスが停止しても、名前解決は継続できる。

(2) ただし、DDNS サーバーのユーザー (DDNS ホストの登録者) に対して案内している DDNS ホストの登録・更新用の URL のホスト名部分が示す IP アドレスを、ある 1 台のインスタンスだけを指すようにしていると、まさにそのインスタンスが停止してしまったとき、DDNS ホストの新規登録・更新・削除ができなくなる。(繰り返しになるが、(1) により、名前解決は継続できる。)
そこで、DDNS サーバーのユーザー (DDNS ホストの登録者) に対して案内している DDNS ホストの登録・更新用の URL のホスト名部分が示す IP アドレスは、複数設定しておくことを推奨する。
具体的には、DDns_StaticRecord で A レコード (必要な場合は AAAA レコード) を複数定義し、その定義の末尾に 「! health_check_url=https://<THIS_IP>/health_check/」という制御文字列を付けることにより、その A レコード (または AAAA レコード) に記載されている IP アドレス上の DDNS サーバー機能が正常に動作しているかどうかを相互監視し合い、DNS レスポンスにおいて、動作が異常な DDNS サーバーに関する A レコード (または AAAA レコード) を応答しないようにすることができるのである。

(3) データベースサーバーは、1 台しか存在できない。したがって、データベースサーバーがダウンしている間は、DDNS ホストの登録・更新・削除ができなくなる。これは、本 DDNS サービスの回避できない制約事項である。ただし、データベースサーバーがダウンしている間も、最後にデータベースサーバーが正常であったときの DDNS ホストレコードの一覧は各 DDNS サーバーのローカルバックアップファイルとしてキャッシュされているので、名前解決は継続することが可能である。したがって、ほとんどのケースにおいて、データベースサーバーの一時的な障害による影響は少ない。



■ DDNS のトラブルシューティング
DDNS サーバーに DNS クエリが正しく飛んできているかなどの基礎的な確認は、DDNS サーバーをホストして動作している Linux サーバー上で tcpdump 等の標準的ツールを用いて確認することができる。

DDNS サーバーのログは、DDNS サービスのコントロールパネルの「Admin Log Browser」から参照できる。

DDNS サーバーに登録されているすべてのオブジェクトの検索・列挙は、DDNS サービスのコントロールパネルの
「Admin Full Text Search」
から可能である。

「Admin Object Editor」というものがある。「Admin Full Text Search」の検索結果として UID を得た場合、「Admin Object Editor」でその UID を指定することにより、DDNS サーバーに登録されているオブジェクトの手動での編集・削除が可能である。これは、ほとんどの場合は実行するべきではない。通常は正規のコントロールパネルの画面上で、DDNS ホストの登録者自らが画面または API を用いて操作を行なうべきである。しかし、何らかの理由で強制的にオブジェクトの書き換えをする必要が生じた場合は、DDNS サーバーの管理者だけは、「Admin Object Editor」を利用して、強制的に操作をすることができる。


■ セキュリティ対策
仮に構築する場合は、上記のようにデフォルトでよいが、長期的に本格運用する場合は、セキュリティ対策を施す必要がある。


1. DDNS サーバーの管理者パスワードは、変更するべきである。設定エディタから、

Service_AdminBasicAuthUsername                       USERNAME_HERE

Service_AdminBasicAuthPassword                       PASSWORD_HERE

の部分を変更すれば、直ちに適用される。


2. いかに 1 の方法で管理者パスワードを変更して、いかに厳重に管理したとしても、その管理パスワードが漏洩してしまった場合にセキュリティ侵害が発生するリスクが残る。そこで、管理画面に対して、特定のホワイトリストの IP アドレス (自社の管理用端末など) 以外からアクセス不能にすることを推奨する。

デフォルトでは、

Service_AdminPageAcl                                 0.0.0.0/0; ::/0

が設定されていて、接続元 IP アドレスは制限がない。これを、たとえば、1.2.3.0/24 とか 1.2.3.4/32 といった具合に制限するとよい。制限をしたら、それらの IP アドレスからは管理画面にアクセスでき、一方で、他の IP アドレスからはアクセスできなくなったことを、必ず試行して確認すること。



3. データベースサーバーの sa のパスワード、および、データベースの読み書き用パスワードは変更するべきである。

sa のパスワードの設定方法は、上記に述べたとおりである。変更する場合は、SQL Server の標準機能を用いて変更すること。

データベースの読み書き用パスワードは、SQL Server 側では、SQL Server の標準機能を用いて変更すること。一方で、DDNS サービスのデーモン側では、上述の設定ファイルのうち

  "HadbSqlDatabaseReaderUsername": "sql_hadb001_reader",
  "HadbSqlDatabaseReaderPassword": "sql_hadb_reader_default_password",
  "HadbSqlDatabaseWriterUsername": "sql_hadb001_writer",
  "HadbSqlDatabaseWriterPassword": "sql_hadb_writer_default_password",

の部分を変更すること。この設定ファイルの変更は、MikakaDDnsServer デーモンを再起動することで適用される。


4. データベースサーバーに脆弱性が発生した場合に備えて (Microsoft SQL Server はしばしば脆弱性が発生する)、データベースサーバーに対してインターネットからアクセスできないようにしたほうがよい。これはとても簡単で、クラウドサービス上の VM の場合は、TCP ポート 1433 に対するファイアウォールを設定すればよい。オンプレミスの物理マシンや VM の場合でファイアウォールが境界上にない場合は、iptables 等で設定をすればよい。

複数の DDNS インスタンスを稼働させる場合、必然的に、データベースサーバーと DDNS サーバーとは、別のホストになる。したがって、データベースサーバーに対しては、それらの DDNS インスタンスからのリモート接続を許容する必要がある。クラウドの場合は VPC のような標準機能を、オンプレミスの場合は LAN の通信を利用すれば、これらは容易に実現可能である。


5. ユーザーからの画面操作、または API 操作で、エラーが発生したときに、ソースコードの行番号や関数名、コールスタック等が表示されてしまい、ソースコードの機密性が侵害される (したがって、セキュリティ的に問題がある) と考える人が世の中には一定数存在する。これは、業務システム等の、プロプライエタリなシステムにおいては正しいが、、本 DDNS ソフトウェアはオープンソースであるから、ソースコードはすべて公開されており、機密性が侵害されることはない。そこで、本システムに関しては、これは誤った考え方である。ただし、誤った考え方をする人に説明をして、正しく直すには、大変コストがかかるから、それよりも簡単な表面的解決方法として、一応、エラー表示からこれらを消しておこうと考えることは、合理的なことである。そのような場合は、設定のうち

Service_HideJsonRpcErrorDetails                      false

を true に変更すればよい。そうすれば、ユーザーからの画面操作、または API 操作で、エラーが発生したときに、ソースコードの行番号や関数名、コールスタック等が表示されないようになる。



■ DDNS サーバーを 2 台以上に冗長化する方法 (2022/8/4 追記 登)
DDNS サーバーを 2 台以上にする手順は以下のとおりです。
なお、上記の「その他 拡張 3」も参照してください。


1. 1 台の場合と同様に、DDNS サーバーを 2 台以上の VM に入れて稼働させます。ただし、データベースサーバーは同じホストには入れません。この状態では DB がないので動作エラーになりますが、正常です。
2. SQL Server のデータベースサーバーは、独立した 1 台の VM に入れて稼働させます。
3. 2 台の DDNS サーバーを一度停止します。
sudo systemctl stop MikakaDDnsServer
各 DDNS サーバーの
/data1/MikakaDDnsServerDaemon/IPA-DN-Cores/Cores.NET/Dev.Test/Local/App_TestDev/Config/MikakaDDnsService.json
をテキストエディタで開きます。

  "HadbSqlServerHostname": "127.0.0.1",

の IP アドレスを、SQL Server を稼働させている IP アドレスに書き換えます。通常、VM 同の通信が VPC で行なわれると思われますので、これは、プライベート IP アドレスになると思います。

4. 2 台の DDNS サーバーを再開します。
sudo systemctl start MikakaDDnsServer
この状態で管理画面に入れることを確認します。
(DB との通信に失敗しているときは、管理画面に入れません。この場合、DDNS サーバーの Debug ログを参照して、DB との通信に失敗している原因を確認して回復します。)

5. Admin Config Editor の DDns_StaticRecord で、ns01, ns02 の 2 台のホストの定義がありますが、これに、2 台の DDNS サーバーのグローバル IP アドレスを指定します。(標準の設定手順書では同じ IP アドレスを指定していました。)
また、親ドメインに関する上位 DNS サーバーの NS レコード (Delegation) について、Glue レコードとして ns01, ns02 の IP アドレスを指定するケースでは、同じく、2 台の DDNS サーバーのグローバル IP アドレスを指定します。

6. 一方の DDNS サーバーの API を用いて適当な DDNS ホストを作成してみて、もう一方の DDNS サーバーの API を用いてその DDNS ホストの定義が見えることを確認します。
また、DNS クライアント (nslookup または dig) を用いて、2 台の DDNS サーバーの両方に対して名前解決をしてみて、結果が同じであることを確認します。

さらに 3 台目、4 台目の DDNS サーバーを稼働させる場合は、ns03, ns04, ... というように追加していくことができます。

クラウド (AWS 等) を用いる場合、複数の DDNS サーバーは、同じリージョンの、異なるアベイビリティゾーンに分散すると良いでしょう。



■ DDNS サーバー冗長化の際の、ユーザー向け DDNS サーバーのアクセス用の A レコードおよび AAAA レコードを複数記載する場合におけるヘルスチェック機能 (2022/8/5)

2022/8/4 の本ドキュメントの案内 (変更差分は Git の履歴を参照のこと) では、DDNS サーバー冗長化の際の、ユーザー向け DDNS サーバーのアクセス用の A レコードおよび AAAA レコードを複数記載する場合におけるヘルスチェック機能は AWS Route 53 等の外部のクラウド型 DNS サーバーホスティングサービスのヘルスチェック機能を用いることを推奨していた。しかしながら、この方法では、外部のクラウド型 DNS ホスティングサービスに依存することになってしまう。

そこで、2022/8/5 のソースコードのバージョンから、DDNS サーバーそのものに、ヘルスチェック機能を内蔵した。

これについて、以下のとおり解説する。

2 台以上の DDNS サーバーを立てる場合、典型的な例では、DDns_StaticRecord の定義は、以下のような記載を行なうことになるであろう:

DDns_StaticRecord   A @ 1.2.3.4
DDns_StaticRecord   A @ 5.6.7.8
DDns_StaticRecord   AAAA @ 11:22:33::44
DDns_StaticRecord   AAAA @ 55:66:77::88
DDns_StaticRecord   A v4 1.2.3.4
DDns_StaticRecord   A v4 5.6.7.8
DDns_StaticRecord   AAAA v6 11:22:33::44
DDns_StaticRecord   AAAA v6 55:66:77::88

ところが、上記の記載だけでは、ユーザーからの本ドメインの DDNS サーバーの Web サーバーそのものに対するアクセスが、2 つの DDNS サーバーに均等分散されてしまう。正常時はそれで良いが、2 台の DDNS サーバーのうち 1 台がダウンした場合などの異常時は、2 分の 1 の確率でしかアクセスに成功しなくなってしまう。

2 台以上の DDNS サーバーが存在するときは、理想的には、相互に監視し合い、ダウンしている DDNS サーバーがあるときは、DDns_StaticRecord のリストから仮想的に一時削除することができれば有益である。そうすれば、ダウンしている DDNS サーバーの A レコードまたは AAAA レコードを応答しなくて済むためである。

これを実現するために、DDNS サーバーそのものに内蔵しているヘルスチェック機能を利用する場合は、次のように記載する。

DDns_StaticRecord   A @ 1.2.3.4 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   A @ 5.6.7.8 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   AAAA @ 11:22:33::44 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   AAAA @ 55:66:77::88 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   A v4 1.2.3.4 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   A v4 5.6.7.8 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   AAAA v6 11:22:33::44 ! health_check_url=https://<THIS_IP>/health_check/
DDns_StaticRecord   AAAA v6 55:66:77::88 ! health_check_url=https://<THIS_IP>/health_check/

つまり、従来の DDns_StaticRecord の定義文の末尾に、

---
! health_check_url=https://<THIS_IP>/health_check/
---

という文字列を追記するだけである。このように、設定方法はとても簡易であるが、この設定は、DDNS サーバー内部において、以下のように有益な動作を実現する。


まず、プログラムは、health_check_url= に記載されている URL にアクセスしようとする。この際は、DDns_HealthCheck_TimeoutMsecs で指定されるタイムアウト秒数 (5 秒間) を用いて、DDns_HealthCheck_NumTry で指定される試行回数 (3 回)、成功するまで試行する。(成功の定義は、HTTP または HTTPS でのアクセスに成功し、HTTP のコード 200 が応答することである。対象 DDNS サーバーの HTTP サーバーには /health_check/ という仮想ディレクトリが存在し、ここにアクセスがあると、正常であれば「OK」という HTTP コード 200 のコンテンツが応答され、異常であれば 500 Internal Server Error 等が戻される。また、そもそも対象 DDNS サーバーがネットワーク的に到達不能であれば、タイムアウトが発生する。)

成功したならば、その DDns_StaticRecord は存在しているものとみなされる。

一方、失敗したならば、その DDns_StaticRecord は存在していないものとみなされる。

なお、URL のうち <THIS_IP> の部分は、マジック文字列であり、これは、A または AAAA レコードの本文で記載されている IPv4 / IPv6 アドレスに置換される。このようなマジック文字列の表記を実現している理由は、表現上、アドレス表示の重複を避けるためである。もちろん、<THIS_IP> の代わりに IPv4 / IPv6 アドレスを直書きしても良いが、たいていの場合、それは不必要な作業である。

[注釈]

1. ヘルスチェックは DDns_HealthCheck_IntervalSecs で指定された秒数間隔 (デフォルトで 60 秒) ごとに実行される。この設定変数を増加または減少させると、ヘルスチェック間隔を変更することが可能である。

2. ヘルスチェックの対象となるべき URL の IP アドレス部が、ヘルスチェックを実行する側の DDNS サーバーそのものであると検出された場合、ヘルスチェックにおける実際の通信は、省略される。代わりに、自らの DDNS サーバーの内部状態に基づいて生成するヘルスチェック結果 (これは /health_check/ という HTTP URL で応答されるものと同じ結果) が利用される。

3. DDNS ドメインそのものの NS レコードの定義や、その NS レコードに対応する A / AAAA レコードを DDns_StaticRecord 設定構文により定義している場合 (ns01 とか ns02 というような名前であろう) は、決して、「! health_check_url」構文を用いてヘルスチェックを実施してはならない。NS レコードや、これに対応する A レコード / AAAA レコードの定義は、一時的な各 NS サーバーの停止によらず、常に安定して定義が存在していることが求められるためである。(1 つ上位の DNS サーバーにおける NS 定義および Glue レコードの存在と、DDns_StaticRecord 設定構文により定義している NS レコードおよびこれに対応する A / AAAA レコードの存在は、完全一致している必要がある。もちろん、完全一致していなくても一応動作はすると考えられるが、DNS クライアントが混乱する原因となる可能性があるので、できるだけ避けるべきである。)


[ソースコードのバージョンについて]

「!」以降が制御文としてみなされるのは、本 DDNS サーバーのソースコードの Git コミット ID が「e051664075c3f805be40f03ad9b894893352bc50」以降のものに限定される。それ以前のコミットのソースコードを利用している場合は、「!」以降が制御文としてみなされる設定を指定してはならない (構文エラーとみなされてしまう)。「!」以降を制御文として記述するためには、まず本 DDNS サーバーをアップデートする必要がある。具体的には、

cd /data1/MikakaDDnsServerDaemon/IPA-DN-Cores/
git fetch
git checkout e051664075c3f805be40f03ad9b894893352bc50

としてコミット ID e051664075c3f805be40f03ad9b894893352bc50 のコードをチェックアウトした後、

sudo systemctl stop MikakaDDnsServer

sudo systemctl start MikakaDDnsServer

と実行して、デーモンを再起動すること。デーモンを再起動すると、自動的に新しいソースコードに基づくビルドが行なわれ、新しいバージョンが起動する。


