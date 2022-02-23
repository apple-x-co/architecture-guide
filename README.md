# My Architecture Guide

## Development flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '0.9em'}}}%%
flowchart TD
    START("START") -- No1 --> uml_usecase("ユースケース図作成")
    uml_usecase -- No2 --> uml_model("モデル図作成\n（集約の明示化、ドメイン知識）")
    uml_model -- No3 --> test_domain("ドメイン層のテスト作成\n（Model, DomainService）")
    test_domain -- No4 --> impl_domain("ドメイン層の実装作成")
    impl_domain -- No5 --> test_domain

    uml_usecase -- No6 --> test_application("アプリケーション層のテスト作成\n（UseCase）")
    test_application -- No7 --> impl_application("アプリケーション層の実装作成")

    impl_application --> END("END")
```

## Diagram

### Use Case

「○○を△△する」形式で記載する  

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '0.9em'}}}%%
graph LR
    worker("fa:fa-user Worker")
    worker --> create_user("ユーザーを登録する")
    worker --> upate_user("ユーザーを更新する")
    worker --> delete_user("ユーザーを削除する")
    worker --> get_user("ユーザーを取得する")
    worker --> list_user("ユーザー一覧を取得する")

    worker -.- comment1("comment1 comment1")
    
    worker -.- comment2("comment2 comment2")

    subgraph "User's Usecase"
        create_user
        upate_user
        delete_user
        get_user
        list_user
    end
```

### Model

集約を明確にする。集約内は同一トランザクションで処理を行うので    
階層を深くしたり、集約内のモデルを増やさない  

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '0.9em'}}}%%
graph BT
    usermeta("User Meta<hr>+metaName<br>+metaValue")
    user("User<hr>+userId<br>+userName<br>+email<br>+userMetas")

    subgraph "User's Aggregate"
        usermeta --compositon--- user
    end

    order("Order<hr>+userId")

    subgraph "Anothor Aggregate"
        order --id reference --> user
    end

    order -.- order_note("注文はXXの状態で作成される")
```

## Layer

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '0.9em'}}}%%
graph
    http
    cli
    memo("* What は Use Case 層に、How は、Model層に記述されているのが理<br>* Model のコンストラクタは新規専用。再構築用メソッドは別にする")

    presentation("Presentation layer<hr>Endpoint<br>Validation(JSON Schema, Form)<br>Mapping input data")
    infra("Infrastructure layer<hr>Repository<br>Query Service (CQRS Read)<br>External System<br>Authentication")
    application("Application layer<br>(Use case)<hr>Query Service Interface (CQRS Read) & DTO<br>Authentication Interface & Session Object ??")
    domain("Domain layer<hr>Value Object(ID, Name ...)<br>Model(Aggregate)<br>Domain Service(Exist check ...)<br>Factory(Policy ...)<br>Repository Interface<br>External System Interface")

    http -.-> presentation
    cli -.-> presentation


    subgraph a
        presentation --> application
        
        infra --implements--> application
        infra --interface--> domain

        application --> domain
    end

    style presentation fill:#FFF9ED,stroke:#333
    style infra fill:#FEEEEB,stroke:#333
    style application fill:#D1EEF9,stroke:#333
    style domain fill:#CDF3EA,stroke:#333
```

### 値オブジェクト

```text
以下に該当しない場合は、プリミティブ型（int,string,bool）にする
・ルールが存在しているか  
・その値を単体で扱いたいか

なお、識別子は値オブジェクトにした方が良い。  
```

例：  
* `AppCore\Domain\User\UserId`  
* `AppCore\Domain\User\UserName`

### ドメインモデル

例：  
* `AppCore\Domain\User\User`

### リポジトリ

```text
ドメインモデルの永続化・検索を行う。集約内は全て処理する。
```

例：  
* `AppCore\Domain\User\UserRepositoryInterface`  
* `AppCore\Infrastructure\Persistence\RDB\UserRepository`

### ドメインサービス

```text
モデルや値オブジェクトに定義した場合に、不自然さがある場合にドメインサービスを利用する。
主に、集合に対するルールや制約を定義する。例えば、重複チェック。
```

例：  
* `AppCore\Domain\User\UserDomainService`  

### クエリーサービス（クエリモデル）

```text
集約を跨いだモデルを取得したい場合に利用する。戻り値は専用の参照系モデル（DTO等）。
参照系モデルは、値オブジェクトを含まないプリミティブな型で構成する。
```

例：  
* `AppCore\Application\User\UserQueryServiceInterface`  
* `AppCore\Application\User\UserXxxDto`  
* `AppCore\Infrastructure\Service\UserQueryService`

### システム外部との入出力（例：メール送信）

```text
内部の層(ドメイン層、ユースケース層)にインターフェイスを定義し、インフラ層に実装クラスを置く。
```

例：  
* `AppCore\Application\Shared\AuthenticationInterface`  
* `AppCore\Infrastructure\Shared\Authentication`  
* `AppCore\Domain\PushNotification\PushNotificationSenderInterface`  
* `AppCore\Domain\PushNotification\PushMessage`  
* `AppCore\Infrastructure\Shared\PushNotificationSender`  
* `AppCore\Domain\Email\EmailSenderInterface`  
* `AppCore\Domain\Email\Email`  
* `AppCore\Infrastructure\Shared\EmailSender`  

### ユースケース

```text
インプットデータに基づいた条件で、実行しアウトプットデータの形式で返却する。
実行には、リポジトリ・ドメインサービス・クエリーサービスを利用する。
```

例：  
* `AppCore\Application\User\Create\UserCreateInputData`  
* `AppCore\Application\User\Create\UserCreateOutputData`  
* `AppCore\Application\User\UserCreateUseCase`

### ビューモデル

```text
表示系モデル。値オブジェクトを含まないプリミティブな型で構成する。
```

例：  
* `AppCore\InterfaceAdapter\Presenter\User\UserGetViewModel`

### コントローラ

```text
クライアントからのリクエストを元に、ユースケースを実行する。
取得したデータは、表示系モデルに入れてビューに渡す
```

## Reference

* Onion Architecture  
https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/  

* ITDDD  
https://github.com/nrslib/itddd

* CQRS実践入門  
https://little-hands.hatenablog.com/entry/2019/12/02/cqrs

* DDDのモデリングとは何なのか、 そしてどうコードに落とすのか  
https://www.slideshare.net/koichiromatsuoka/domain-modeling-andcoding

* PlantUML  
https://plantuml.com/ja/class-diagram
