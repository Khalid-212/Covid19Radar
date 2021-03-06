# 濃厚接触検知のデータモデルと API仕様

本プロジェクトで用いられるドメインモデルと周辺知識に関して解説します。

[Beacon](https://www.cresco.co.jp/iot/solution/keyaki/beacon/) とは　Bluetoothの信号の発信機で、信号を数秒に１回、半径数十メートルの範囲に発信します。この機能は、iPhone や Android のスマートフォンにも搭載されています。本プロジェクトでは、この機能を使って、コロナウイルスに感染した人と濃厚接触した人に通知が出来るようにすることを目指しています。ただし、誰がコロナウイルスに感染したかなどはきわめてセンシティブな情報であり、他者から本人が特定されることなく、アプリケーションのユーザが自分がコロナウイルスの感染者と濃厚接触があったかを知ることができるためのシステムです。

## Beacon のデータモデル

Beacon 端末（今回のプロジェクトではスマートフォン）は、BLE を用いて周囲の端末にデータを送信します。Beaconの端末が送信できるデータは下記の３つのプロパティになっています。本来は、UUIDがBeaconの識別子であり、Major, Minor の値は、識別子のクラスタリングするためのプロパティです。例えば、Beaconが個人を特定するUUIDが振られた場合、Majorが学校のID,Minorがクラスを割り振る事が可能でフィルタリング等に使われます。

今回のアプリケーションの場合、個人を特定することなく、コロナウイルスの陽性の人と濃厚接触があったことを知りたいため、次のような仕様になっています。

## アプリケーションの識別

`UUID` は今回は Beacon の識別子で、今回の`COVID-19Radar`のアプリケーションでユニークの値です。つまり、今回のアプリでは全員が同じUUIDを持ちます。これによって、Beaconのデータを受信した端末が、Beaconが、`COVID-19Radar`をインストールした端末からのデータであることを判別することができます。

## ユーザーの識別

ユーザの識別は、MajorとMinor の二つの値のペアでその人であることを識別します。ただし、個人的な情報は一切サーバーサイドに送信しないようにします。例えば、`Major: 0 , Minor: 1` の人が、`Major: 0, Minor : 2`の人と濃厚接触したよというデータだけをサーバー側に保持します。あえて特定の個人や、端末との紐づけは実施しません。これは先に述べた通り個人の特定をデータ上できないようにするためです。このユーザーの識別子は、端末をインストールした際に、サーバーのAPI経由で採番されるルールになっています。採番は、`Major: 0, Minior: 0`からスタートして、Minor 0 -> 65535 までインクリメントし、その次の値は `Major: 1, Minor 0` になります。尚データが int 16 なのは IoT の性質上、32bitですらない端末を想定しています。

| プロパティ | 概要 | サンプル | 
| ---------- | ------------------ | --------- |
| UUID | Beacon の識別子 Beacon からの通信がこのアプリからの通信であることを識別します。 | 550e8400-e29b-41d4-a716-446655440000 |
| Major | [Beacon の仕様](http://webmarketing.hatenadiary.com/entry/2016/06/03/%E3%80%90Beacon%E3%83%9E%E3%83%8B%E3%82%A2_-_EP3%E3%80%91Beacon%E3%81%8B%E3%82%89%E9%80%81%E3%82%89%E3%82%8C%E3%82%8B%E3%83%87%E3%83%BC%E3%82%BF%E3%81%A8%E3%81%AF%EF%BC%9F%EF%BC%88Major%E5%80%A4)にある Major。16 bit int の数字。Major と　Minor のセットがユーザの識別子になります。 | 0 - 65535 の値 |
| Minor | [Beacon の仕様](http://webmarketing.hatenadiary.com/entry/2016/06/03/%E3%80%90Beacon%E3%83%9E%E3%83%8B%E3%82%A2_-_EP3%E3%80%91Beacon%E3%81%8B%E3%82%89%E9%80%81%E3%82%89%E3%82%8C%E3%82%8B%E3%83%87%E3%83%BC%E3%82%BF%E3%81%A8%E3%81%AF%EF%BC%9F%EF%BC%88Major%E5%80%A4)にあるにある Minor。16 bit int の数字。Major と　Minor のセットがユーザの識別子になります。 | 0 - 65535 の値 |

## 濃厚接触の判定

濃厚接触の判定のためには、相手の端末との距離と、接触していた時間を用いて判断します。

相手と、自分の距離の判定の方法は、ビーコン端末から、`Distance`が送信されます。これは、iOS, Android の双方の端末のAPIで取得できるのですが、メーカーによって精度がまちまちであり、例えば間に障害物がある場合には、実際は１メートルなのに、４メートルと判定されてしまったりします。つまり、信頼性にかけます。そこで、相手との距離の判別のためのロジックをサーバー側でもカスタマイズ、チューニング可能なように、Rssi, TXPower を送信します。特定のアルゴリズムを使うと、これにより２端末間の距離を導出することができます。ただし、これを使っても誤差が生じるので、例えば１０回のデータの平均値等を用いて判定します。このロジックは試行錯誤が必要になるために、変更可能にしておく必要があります。

スマートフォン側では、例えば、１分に１回などの頻度でサンプリングをして、上記の手法により、サマリーをして、バッファリングしておき、例えば累積時間が３０分になったものから、サーバーに送信するといった方法を行います。

| プロパティ | 概要 | 
| ---------- | ------------------ | 
| Distance | 相手端末との距離 |
| Rssi | 電波強度。どのぐらいの強度で電波を受信したか？ |
| TXPower | 相手側の送出電波力　|
| ElapsedTime | 特定の相手との累積接触時間 |
| LastDetectTime | 特定の相手と最後に接触した時間 |

_BeaconDataModel.cs_

```csharp
    public sealed class BeaconDataModel

    {
        public string UUID { get; set; }

        public string Major { get; set; }

        public string Minor { get; set; }

        public double Distance { get; set; }

        public int Rssi { get; set; }
        public int TXPower { get; set; }

        public TimeSpan ElaspedTime { get; set; }
        public DateTime LastDetectTime { get; set; }
    }
```

# サーバー側 API 

サーバー側には次の API が必要になることが見込まれています。下記の機能は全てGitHub issue として定義される予定ですが、他のAPIも必要になる可能性があります。

## 濃厚接触 API (CloseContactAPI)

上記で解説した `BeaconDataModel` ある端末が受信した、相手側 Beacon からの情報累積、と、ある端末（ユーザ）の識別データである `UserData` をサーバー側に送信します。これにより、サーバー側で、どの人が、どの人と濃厚接触したかが記録されます。人の判別は Major, Minor のペアで行いますが、個人情報、端末のが特定できる情報は一切送信しません。この時点では、クライアントは相手がコロナウイルス陽性であるか否かは知ることができません。サーバー側で処理されます。

尚、濃厚接触の報告は、CosmosDBにストアされすぐにレスポンスをクライアントに返しますが、CosmosDB の ChangeFeedにより、バッチが起動し、CosmosDBに登録された情報を元に、あるユーザのMajor + Minor をキーとして検索すると、濃厚接触があったか、その期間はどれぐらいかの情報を取得できるような、検索に特化した別のテーブルに非同期で書き込みに行く必要があります。また、それ以外にも、下記で説明する [Network Navigator](#PowerBI-NetworkNavigator)のためのテーブルを作成する必要がある可能性があります。

```csharp
    class UserDataModel
    {

        public string Uuid { get; set; }

        public string Major { get; set; }

        public string Minor { get; set; }

        public string GetId()
        {
            return String.Format("{0}.{1}.{2}", Uuid, Major.PadLeft(5, '0'), Minor.PadLeft(5, '0'));
        }
    }
```

## ユーザＩＤの発行

サーバー側で、ユニークな Major, Minor の値を生成して、クライアントに返却します。ロジックは、`Major:0, Minor:0` から初めて `Major:0, Minor: 65535` に到達したら `Major:1, Minor: 0` が次の値になります。

## 自分が濃厚接触があったか確認するＡＰＩ 

クライアントからのポーリングにより、送信された、`UserDataModel`のデータ、つまり、Major, Minor のペアで検索すると、自分がコロナ陽性の人と濃厚接触したか、それはどの程度の時間かということを知ることが出来るようになります。

## PowerBI NetworkNavigator 

素敵なグラフによるビューを[
NetworkNavigator](https://github.com/microsoft/PowerBI-visuals-NetworkNavigator)により提供する予定です。ただし、このライブラリの元データのデータ構造は現在調査中ですので、確定しだいシェアされると思われます。[濃厚接触 API](#濃厚接触-API-(CloseContactAPI))の非同期処理として、そのテーブルが提供される予定です。

## 陽性登録・解除API

ある人が、病院等の施設で、陽性であると判明した場合、陽性であることをCOVID-19Radarに登録します。具体的には、端末に表示されたMajor,MinorのIDを医師が聞く、もしくはそれ以外の手段（例えば、QRコードの表示の読み取り）により、登録が可能です。ただし、この機能を使うことができるのは、医師など、COVID-19Radar の事務局に指定された人のみになります。これは、自分でそうではないのに、コロナ陽性にセットするような愉快犯を防止するためです。同じく、医師は患者が、陽性から陰性に変わったときに、陽性の解除をすることが可能です。具体的には、陽性患者の Major, Minor を格納したマッピングテーブルが作成されることになると思いますが、設計の詳細は実施していませんので、実施していきましょう。

# 非機能要件

個人のプロジェクトとしてスタートしており、特に予算も無いので、コスト面で負担にならないようなアーキテクチャ選定を実施したいと思います。

# 仕様のアップデート

本仕様が変更された場合、このドキュメントが Pull Request によって更新されていくことが望ましいです。

