# AWS上にCloud Log Receiver を一から起動する方法

1. (初回構築時のみ行う) CloudTrailやCostExplorerを有効化する。詳細は「これだけでOK！ AWS 認定ソリューションアーキテクト – アソシエイト試験突破講座（初心者向け21時間完全コース）」の「セクション2: Day1対応の実施」を見る。

1. RegionをN.Virginiaに変更する。以降の作業は全てN.Virginiaで行う。時々勝手に別Regionに飛ばされることがあるので、エラーが出た時の原因の可能性の一つとして覚えておくこと。

1. (初回構築時のみ行う) key pairs (公開鍵と秘密鍵)をマネジメントコンソール上で作る。key pair nameは任意でOK。ただし作成と同時にダウンロードされるpemファイル(秘密鍵)は共有・公開せず、NASにもGitlabにも上げず厳重に保管すること。後のステップで利用する。

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/EC2KeyPairs.PNG" width=80%>

1. (初回構築時のみ行う) 作成したkey pairs のNameを各種Cloudformationのyamlファイルに上書きする。 

1. Cloudformationマネジメントコンソールに移動する

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/cloudFormationTop.PNG" width=80%>

1. aws_cloud_log_collection\AWSCloudFormation\CommonStack\RootStackGlobalCommonResources.yamlを実行する。開発環境構築時はdev、テスト環境構築時はtest、本番環境構築時はprodを必ず選択する。しないと構築失敗する。Stack Nameは適当でOK。Tagsはkey: name, value: stack name、で登録する。

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/runCloudformation1.PNG" width=50%>
<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/runCloudformation2.PNG" width=50%><img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/runCloudformation3.PNG" width=50%>

最後はチェックを入れてCreate Stack。全てのリソースがCREATE_COMPLETEになったら次に進む。
<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/runCloudformation4.PNG" width=50%><img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/runCloudformation5.PNG" width=50%>

1. RootStackGlobalCommonResources.yamlと同じ要領でaws_cloud_log_collection\AWSCloudFormation\CommonStack\RootStackS3ForCfnTemplates.yamlを実行する(EnvTypeには十分注意すること！)。

1. RootStackGlobalCommonResources.yamlと同じ要領でRootStackRegionalCommonResources.yamlを実行する(EnvTypeには十分注意すること！)。成功したら以下のようになる。

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/runCloudformationRegionalCommonStack.PNG" width=80%>

1. S3マネジメントコンソールに移動する
1. (初回構築時のみ行う) S3バケット「logs-cfn-templates*」の中で「LogsInquirer」フォルダと「LogReceiver」フォルダを作成する。AES-256で暗号化すること

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/s3FolderCreation.PNG" width=80%>

1. 「LogReceiver」フォルダにNestedStackAutoScaling.yamlとNestedStackEC2.yamlとNestedStackNATGateway.yamlとNestedStackNetworkLoadBalancer.yamlとNestedStackSubnet.yamlをアップロードする

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/s3cfnTempFiles.PNG" width=80%>

<参考> aws_cloud_log_collection\AWSCloudFormation\LogsLogReceiver\RootStackMultiEC2SecurityDoctorLogs.yamlの中で、LogReceiverにアップロードしたテンプレートファイル(＝Nestしているファイル)は、各resourceのTemplateURLプロパティで指定している。
<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/LogreceiverCfn2.PNG" width=50%><img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/LogreceiverCfn2.PNG" width=50%>

1. (初回構築時のみ行う) AWS Systems Manager Parameter Storeで、自分のGitlab username とGitlab passwordを登録する。Nameとidが以下の画像と一致するように登録すること(開発環境=devの場合)。本番環境の場合はdevをprodに読み替えること。AwsAccessKeyIdとAwsSecretAccessKeyの登録は不要。

<img src="https://github.com/kensaku-okada/aws_cloud_log_collection/blob/master/pictures/startup_manual/AWSSystemsManagerParameterStore1.PNG" width=80%>

1. Cloudformationマネジメントコンソールに移動する。

1. 同じ要領でRootStackMultiEC2SecurityDoctorLogs.yamlを実行する(EnvTypeには十分注意すること！)。このスタックの完了には2，3分時間がかかる。通知を受けるメールアドレスはParametersのOperatorEMailで変える。

1. 作成したEC2インスタンスにアクセスしたければ、「プライベートサブネットに作成されたEC2インスタンスにアクセスする方法」を参照する。

1. Route53マネジメントコンソールに移動する
1. 利用したいドメインをHosted Zoneとして登録する
1. 登録したドメインのDNSサーバー(ネームサーバー)名を登録する
1. 登録したドメインのRecord Setを作成して、NLBのDNS名と紐付ける。Routing Policy は"Simple"にする


# EC2を再起動しないで、インスタンスタイプやUserDataを更新する方法(EC2インスタンスの差し替え。)

# Autoscalingのパラメータの変更の仕方(マネジメントサービス)

# ドメインの登録とNLBとの紐付けの方法(マネジメントサービス)

# アクセスキーの取得の仕方。(マネジメントサービス)->S3からローカルにログを落とす方法を知るため。

# プライベートサブネットに作成されたEC2インスタンスにアクセスする方法
1. システム監視のためにAWS CloudtrailまたはGuardDutyをを実装する。(現在、VPCFlowLogしか有効化されてない)
1. 「AWS上にSecurity Doctor Cloud Log Receiver を一から起動する方法」を参考に、RootStackGlobalCommonResources.yamlを実行する
1. 「AWS上にSecurity Doctor Cloud Log Receiver を一から起動する方法」を参考に、RootStackRegionalCommonResources.yamlを実行する
1. aws_cloud_log_collection\AWSCloudFormation\SecurityDoctorLogsLogSender.yamlを実行する
1. 作成されたインスタンスにSSHログインする。パブリックIPアドレスはEC2のマネジメントコンソールより確認する。
1. 以下のコマンドを順番に実行していく
    - 1. sudo -i
    - 1. nano \<your private key file name\>.pem
        - \# プライベートキーファイルの中身をコピペする(-----BEGIN RSA PRIVATE KEY-----と-----END RSA PRIVATE KEY-----もコピーする。ファイルの中身全部コピーする)
    - 1. chmod 400 \<your private key file name\>.pem
    - 1. ssh ubuntu@<アクセスしたいインスタンスのprivate ip address> -i \<your private key file name\>.pem
        - \# 例：ssh ubuntu@10.1.2.104 -i kensaku_root_kensaku_admin.pem

Windows上でSSHログインするツールはTeratermがおススメ。Puttyより楽。

# リリースまでに絶対やること(これらだけやればとりあえずリリースしても良いかも。最終的な決定は「やってない事」について協議して決定したい)
1. DockerfileのUSERの権限をrootから、必要最低限の権限を持ったユーザーに制限する。(参考：https://www.qoosky.io/techs/f38c112ca9 https://qiita.com/tanan/items/e79a5dc1b54ca830ac21)
1. CloudformationのAutoscalingグループで作成するインスタンスをスポットからオンデマンドにする。万が一、スケーリング中すぐに取り上げられたくないので。ー＞対応完了
1. 以下のリンクにあるTrivyを使って、コンテナの脆弱性テストをする。http://knqyf263.hatenablog.com/entry/2019/08/20/120713
1. NLBからとれるログをS3に溜めるようにする。
1. EC2インスタンスを更新する方法のマニュアルを作る
    - 有料サポートより回答は貰っているので、それに対応するようにコードを修正してマニュアルを作る。ー＞対応完了(私用によりUserDataの更新はできない)


# やってない事
1. 本番用のドメインの登録およびHTTPS証明書の作成とデプロイ　ー＞リリースと同時に対応完了
1. CRUDレベルでのIAMポリシーの制御(マネッジドポリシー単位での制御は行った)
1. NestedStackEC2.yamlを利用するときに、LogReceiverGitlabUserName LogReceiverGitlabPassword AwsAccessKeyId AwsSecretAccessKeyを入力しなくてもいいように修正する　ー＞対応済み
1. 本番アカウントでのデプロイ確認　ー＞リリースと同時に対応完了
1. SDLS01サーバーを廃止し、SDLS00サーバーがS3からプルするコンテナを作る
1. 監査ログ収集インフラの構築
1. UserDataの代わりにcfn-initでコマンドを記述する
1. メモリが8GB以上のインスタンスで、Logstashの定義ファイルの設定を変えてHeap領域を全体の半分にすることでメモリ稼働率がどう変わるかWatchする
    - 各インスタンスタイプのメモリスペックを取得して、それをUserDataに組み込む方法を調べる。
1. IAMの中で、PolicyとRoleは自由に作れるがGroupsとUsersは閲覧できないようにして、MFAも強制する、そんなGroupを作る
