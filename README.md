# DIとDIコンテナ

## DIとは

はじめに、DIというプログラミング上の設計手法があります。  
DIコンテナは、そのDIを実現する手段のひとつです。  

下記リンクはleague/containerのドキュメントです。CakePHP4にも採用された、人気のあるDIコンテナライブラリのひとつです。  
[https://container.thephpleague.com/4.x/](https://container.thephpleague.com/4.x/)
> 訳：コンテナは依存性注入コンテナです。これにより、依存性注入設計パターンを実装できます。つまり、クラスの依存性を分離し、必要な場所にコンテナーに注入させることができます。

あるクラスAが動作するために、別のクラスBを必要とするような場合、  
クラスAはクラスBに依存した状態にあり、クラスBはクラスAにとっての依存性ということになります。  

このとき、クラスAの内部でクラスBをインスタンス化して用いるのではなく、  
外部で生成したクラスBのインスタンスを、メソッドの引数を通じてクラスAに与えることで、  
クラス同士の関係を疎結合にするべきである、という考え方がDIの基本です。  

クラス同士の関係を疎結合にするべきである、という考え方がDIの基本です。  

単純な例として、KnsPDOはコンストラクタの引数として、PDOのインスタンスを要求します。  

```PHP
class KnsPDO implements KnsPDOInterface
{
    // コンストラクタの引数にPDOのインスタンスを要求している。
    public function __construct(private \PDO $pdo)
    {
    }
}

// PDOをインスタンス化し、KnsPDOのコンストラクタに渡して使う。
$pdo = \PDO('mysql:host=localhost;dbname=dbname;user=user;password=password;charset=utf8mb4');
$knspdo = new KnsPDO($pdo);
```

メソッドを通じて依存性を注入することをメソッドインジェクションといいます。  
特にコンストラクタでのメソッドインジェクションは、コンストラクタインジェクションと別称されます。  
DIコンテナを用いる場合、基本はコンストラクタインジェクションでDIを実現します。

## DIコンテナの役割

KnsPDOの場合はシンプルですが、以下のような場合はどうでしょうか。

- IndexActionというクラスがあり、IndexDomainクラスとIndexResponderクラスを要求している。  
  - IndexDomainクラスはKnsPDOクラスを要求している。  
    - KnsPDOクラスはPDOクラスを要求している。  
  - IndexResponderクラスはTypewriterクラス（テンプレートエンジン）を要求している。  

多くの場合、DIの考えを実装に取り入れようとすれば、クラス同士の依存関係のネストが深くなるので、  
開発者はインスタンス化処理を順々に記述し、正しい順番で注入していく必要があります。  

- それをどこに記述するのか？  
- 分かりやすく管理できないか？  

そうした問題を解決するための手段がDIコンテナです。  

```PHP

// DIコンテナをインスタンス化
$container = new \Knp\DiContainer\DiContainer;

// KnsPDOクラスのインスタンス化処理を登録
$container->attach(KnsPDO::class, function (): KnsPDO {
    return KnsPDO::connect('mysql:host=localhost;dbname=test;user=admin;password=admin;charset=utf8mb4');
});

$knspdo = $container->get(KnsPDO::class);
```

## Auto Wiring

規模が小さい、あまり複雑すぎないアプリケーションにおいては、  
DIコンテナライブラリの多くが備えているAuto Wiringという機能が有用です。  

あるクラスが要求する依存性を判別し、それがインスタンス化できるクラスであれば自動で注入してくれます。  

先述したIndexActionの例で説明します。  

```PHP
class IndexAction {
    // IndexDomain、IndexResponder、ResponseInterfaceを引数として要求する。
    public function __construct(
        protected readonly ResponseInterface $response,
        protected readonly IndexDomain $domain,
        protected readonly IndexResponder $responder
    )
    {
    }
}

class IndexDomain {
    // KnsPDOを引数として要求する。
    public function __construct(private readonly KnsPDO $pdo)
    {
    }
}

class IndexResponder {
    // Typewriterを引数として要求する。
    public function __construct(private readonly Typewriter $view)
    {
    }
}

// DIコンテナをインスタンス化
$container = new \Knp\DiContainer\DiContainer;

// \Laminas\Diactoros\Responseのインスタンス化処理を登録
$container->attach(ResponseInterface::class, function (): ResponseInterface {
    return new \Laminas\Diactoros\Response;
});

// Typewriterクラスのインスタンス化処理を登録
$container->attach(Typewriter::class, function (): Typewriter {
    return new Typewriter;
});

// KnsPDOクラスのインスタンス化処理を登録
$container->attach(KnsPDO::class, function (): KnsPDO {
    return KnsPDO::connect('mysql:host=localhost;dbname=test;user=admin;password=admin;charset=utf8mb4');
});

// ネストした依存関係を自動で解決して、IndexActionをインスタンス化
$container->get(IndexAction::class);
```

$containerからgetメソッドでIndexResponderクラスのインスタンスを取り出そうとすると、  
まずIndexResponderクラスが要求するTypewriterクラスをインスタンス化し、  
それをIndexResponderクラスのコンストラクタに与えてインスタンス化し返却する、という処理をおこないます。  

TypewriterやKnsPDOのインスタンス化処理については、コンテナに事前にセットされているのですが、  
「IndexResponderクラスにTypewriterクラスを注入する」という処理はどこにも記述されていません。  
DIコンテナのAutoWringが自動的に判別してくれる仕組みになっています。  

## Attributes Injection

Auto Wiringが機能するには、DIコンテナに登録（attach）済みの型か、具象クラスが指定されていなくてはなりません。  
未知のInterfaceや抽象クラスが指定されている場合、DIコンテナは何をインスタンス化してよいか分からないからです。  

以下の例は、knp/adrの基底Actionクラスと、そのコンストラクタの記述です。  

```PHP
abstract class Action　{
    abstract public function __construct(
        ResponseInterface $response,
        ?DomainInterface $domain,
        ?ResponderInterface $responder
    );
}
```

第一引数の`ResponseInterface`はattach済みなので解決できますが、  
第二、第三引数の`DomainInterface`と`ResponderInterface`はInterfaceであり、  
事前にattachされていない型なので、Auto Wiringでの依存解決に失敗します。  

これをどのように解決するかは、各DIコンテナライブラリの哲学によるところなのですが、  
`knp/di-container`では`Attributes`を用いる手段を採用しました。PHP-DIでも採用されているものです。  
基底Actionクラスを継承するクラスで、以下のように記述します。

```PHP
class IndexAction extends Action {
    public function __construct(
        protected readonly ResponseInterface $response,
        #[Inject(IndexDomain::class)] protected readonly DomainInterface $domain,
        #[Inject(IndexResponder::class)] protected readonly ResponderInterface $responder
    ) {
    }
}

$container->get(IndexAction::class);
// Response、IndexDomain、IndexResponderのインスタンスを備えたIndexActionのインスタンスを取得できる。
// IndexDomainはKnsPDOを持っており、KnsPDOはPDOを持っている。
// IndexResponderはTypewriterを持っている。
```

第二、第三引数の先頭に、`#[Inject()]`という記述を追加し、その中に具象クラス名を記述しました。  
`knp/di-container`はこのAttributesを参照して、依存解決を図ります。  
（protected readonlyという記述も追加されていますが、これはDIとは関係がありません）。
