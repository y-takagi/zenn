---
title: "iOSでデータを永続化する方法"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios"]
published: true
---


作成日時: 2016-07-01
iOS version: 9


この投稿では、iOSのファイルシステムについて理解し、データを永続化(iCloud含む)する方法を紹介する。尚、サンプルコードは動かない可能性もあるので参考程度にして下さい。

## iOS File System
アプリがファイルシステムとやり取り出来る場所は、ほぼアプリのサンドボックス内のディレクトリに制限されている。新しいアプリがインストールされる際、インストーラーはサンドボックス内に複数のコンテナを作成し、図1に示す構成をとる。各コンテナには役割があり、Bundle Containerはアプリのバンドルを保持し、Data Containerはアプリとユーザ両方のデータを保持する。Data Containerは用途毎に、さらに複数のディレクトリに分けられる。アプリは、例えばiCloud Containerのように、実行時に追加のコンテナへのアクセスをリクエストすることもある。

![](https://storage.googleapis.com/zenn-user-upload/cb9bz60vbhds2frzwihsl5sba4yx)

図1. An iOS app operating within its own sandbox

表1は、各コンテナ内の重要なサブディレクトリについて、その用途やアクセス制限についてまとめたものである。ファイルの種類とそれに対応する保存場所はガイドラインに明記してあり、それに反した場合はリジェクトの対象となるので注意が必要。

表1. Commonly used directories of an iOS app

| ディレクトリ | 説明 | iTunes, iCloud バックアップ対象 |
|:-----------|:----|:-----------------------------|
| `AppName`.app| メインバンドルと呼ばれている。アプリとアプリのリソースファイルを含んでいるディレクトリ。読み取り専用。 | NO |
| Documents/ | ユーザが作成したコンテンツ等を保存しておく場所。 | YES |
| Documents/Inbox/ | 他のアプリからファイルを受け取るときに使用するディレクトリ。読み取り専用。 | YES |
| Library/ | アプリで利用するデータを保存しておく場所。よく使われるサブディレクトリにApplication SupportとCachesがあるが、カスタムディレクトリを作ることも可能。| YES |
| Library/Application Support/ | アプリで利用するデータを保存しておく場所。 | YES |
| Library/Caches/ | 一時的なデータや、後から再度ダウンロードして復旧可能なデータを保存する場所。アプリ側でデータの追加、削除を管理する必要あり。ただし、ディスク・スペースが足りない等の理由でシステムが自動削除する可能性がある。 | NO |
| Library/Preferences/| NSUserDefaultsのデータが保存される場所。| YES |
| tmp/ | 一時的なデータを保存する場所。使い終わったらアプリ側で削除する必要あり。ただし、アプリが動作してないときにシステムが自動削除する可能性がある。| NO |

### Path to Directory
各種ディレクトリパスの取得方法

```swift
let manager = FileManager.default()

// Documents/
let documentDir = manager.urlsForDirectory(.documentDirectory, inDomains: .userDomainMask)[0]

// Library/
let libraryDir = manager.urlsForDirectory(.libraryDirectory, inDomains: .userDomainMask)[0]

// Library/Application Support/
let applicationSupportDir = manager.urlsForDirectory(.applicationSupportDirectory, inDomains: .userDomainMask)[0]

// Library/Caches/
let cachesDir = manager.urlsForDirectory(.cachesDirectory, inDomains: .userDomainMask)[0]

// tmp/
let tmpDir = manager.temporaryDirectoy
```


### Write/Read Files
iOSではオブジェクトをファイルに保存する際に

- アーカイブ(シリアライズ)
- プロパティリスト(*.plist)

を使う2通りがある。アーカイブは単純に、オブジェクトをNSData型に変換（これをアーカイブという）してからファイルに保存する方法。またNSArrayとNSDictionaryクラスは、それぞれプロパティリストとして保存する仕組みが提供されている。

アーカイブして保存する例

```swift
// UIImageの保存
let imagePath = documentDir.appendingPathComponent("sample.png")
let image = UIImage(named: "bundle_image.png")
// 書き込み
UIImagePNGRepresentation(image)?.write(to: imagePath, atomically: true)
// 読み込み
UIImage(data: NSData(contentsOf: imagePath))

// Stringの保存
let strPath = documentDir.appendingPathComponent("sample.txt")
let str = "Sample text"
let data = str.data(using: .utf8)
// 書き込み
data?.write(to: strPath)
// 読み込み
String(data: Data(contentsOf: strPath), encoding: .utf8)
```

プロパティリストとして保存する例

```swift
let path = documentDir.appendingPathComponent("sample.plist")
var user = NSDictionary(dictionary: ["Name":"Tim Cook"])

// 書き込み
user.write(to: path, atomically: true)

// 読み込み
var user = NSDictionary(contentsOf: path)
```

## Data Managing Services
SDKにはデフォルトでデータを管理する仕組みが用意されていて、用途に応じてNSUserDefaultsやCore Dataを使って管理が可能である。なお、Core Dataの代わりにRealmというモバイル向けデータベースを使うケースも増えてきているが、ここでは割愛する。

表2. Commonly used services for managing data in iOS app

| タイプ | 説明 |
|:------|:----|
| NSUserDefaults | Key/Value方式で任意のオブジェクトの保存・読み込みが可能。plistとして保存される。 |
| Core Data | SQLite用のORM。DBファイルはDocuments/以下に作られる。 |
| Keychain | 暗号化されたストレージ。パスワードや秘密鍵、証明書などを安全に保存する用途として使われる。 |

## iCloud Storage
iCloud上にデータを保存しておくことで、複数端末から同じデータへのアクセスが可能となる。iCloudでは表3に示す3つのストレージタイプがあり、用途に応じて使い分ける。各サービスを利用するには、Xcode projectのCapabilitiesタブでiCloudをONにして、設定しておく必要がある。

表3. Types of storage service in iCloud

|タイプ|説明|
|:----|:--|
| Key-value storage | アプリの設定や状態などを保存するためのKVS。最大容量は1アプリ1MB。 |
| iCloud Documents | ファイルベースのストレージ(Core Data含む)。ドキュメントやドローファイル、または複雑なアプリの状態などを格納するために利用可能。最大保存容量はユーザーのiCloudアカウントに依存。|
| CloudKit storage | Appleが提供するBaaSの内のストレージ機能。構造化されたデータを管理したり、ユーザー同士で共有したいファイルやデータを保存したい際に利用する。データの管理にはデータベースが使用され、各レコードはKVSとして扱う。|

### Key-value storage
iCloud Key-value storageには、[NSUbiquitousKeyValueStore](https://developer.apple.com/reference/foundation/nsubiquitouskeyvaluestore)クラスを使ってアクセスする。書き込みするデータはまずメモリで保持され、その後システムによって適切なタイミングでディスクに書き込まれる。もしiCloudアカウントにサインインしていない状態で書き込みを行った場合、データは次の同期までローカルに保存されておく。ユーザがiCloudアカウントにサインインしたら、システムが自動的にローカルのKVSとiCloudサーバとの帳尻を合わせてくれる。

コード例

```swift
// AppDelegate.swift

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    let kvs = NSUbiquitousKeyValueStore.default()

    // register to observe notifications from the store
    NSNotificationCenter.defaultCenter()
                        .addObserver(self,
                                     selector: #selector(storeDidChange:),
                                     name: NSUbiquitousKeyValueStoreDidChangeExternallyNotification,
                                     object: kvs)

    // get changes that might have happened while this
    // instance of your app wasn't running
    kvs.synchronize()

    // Get/Set value
    kvs.set("Value", forKey: "Key")
    kvs.string(forKey: "Key")

    return true
}
```

### iCloud Documents
アプリで扱うドキュメント（[UIDocument](https://developer.apple.com/reference/uikit/uidocument)を継承したクラスのオブジェクト）をiCloud上に保存しておくサービス。ここで言うドキュメントは、単一のファイルやファイルパッケージとしてディスクに書き込むことができるデータの集合体である。ローカルではiCloud Containerにファイルが置かれ、Documents/以下のファイルはユーザに公開される。それ以外に置かれているファイル、ディレクトリはユーザに非公開である。

ドキュメントはデフォルトでiCloud apiに対応しており、iCloud上のファイル変更とローカルでのファイル変更を上手く調整してくれる。これは[NSFileCoordinator](https://developer.apple.com/reference/foundation/nsfilecoordinator)クラスのオブジェクトと[NSFilePresenter](https://developer.apple.com/reference/foundation/nsfilepresenter)プロトコルによって実現されている。ファイルの開く、保存、名前変更をするためのUIはアプリ上で実装する必要がある。

コード例

```swift
let manager = FileManager.default()
let containerPath = manager.urlForUbiquityContainerIdentifier(nil)
let documentPath = containerPath.appendingPathComponent("Documents")
let filePath = documentPath.appendingPathComponent("document.key")

// Create document file to iCloud Container
let document = UIDocument(fileURL: filePath)
document.save(to: document.fileURL, for: .forCreating)

// Download is automatically done by the OS
```

またCore Dataも、[UIManagedDocument]()クラス(UIDocumentのサブクラス)を利用することで、同じ仕組でiCloudとの同期を取ることができる。

### CloudKit
基本的には他のユーザと共有するデータを保存しておくサービス。各レコードはKVSとして扱われ、単純なタイプ（文字列、数字、日付など）から複雑なタイプ（位置情報、参照、アセットなど）のオブジェクトまで対応している。

他のiCloudサービスと同じで、データはローカルのコンテナで作成されシステムによって同期が取られる。[CKContainer]()オブジェクトでコンテナ内にあるデータベース[CKDatabase]()オブジェクトにアクセスできる。

コード例

```swift
let container = CKContainer.default()
let database = container.publicCloudDatabase

// Create record
let record = CKRecord(recordType: "employee")
record.setObject("Jacob", forKey: "firstName")
record.setObject("Nielsen", forKey: "lastName")
database.save(record, completionHandler: { (record: CKRecord?, error: NSError?) -> Void in })

// Fetch record
let recordID = record.recordID
database.fetch(withRecordID: recordID, completionHandler: { (record: CKRecord?, error: NSError?) -> Void in
})
```

## Reference
- [File System Programming Guide](https://developer.apple.com/library/ios/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html)
- [iCloud Design Guide](https://developer.apple.com/library/prerelease/content/documentation/General/Conceptual/iCloudDesignGuide/Chapters/Introduction.html)
