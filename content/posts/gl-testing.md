---
date: '2025-04-01T10:04:01+09:00'
title: 'Greenlightアプリケーションのテスト'
---

https://github.com/Blockstream/greenlight/blob/main/docs/src/tutorials/testing.md の翻訳。  

- [このチュートリアルの学習内容](#このチュートリアルの学習内容)
- [テストの必要性と対象範囲](#テストの必要性と対象範囲)
- [`gl-testing`テストフレームワークの概要](#gl-testingテストフレームワークの概要)
  - [`gl-testing`とプロダクション環境の挙動の差異](#gl-testingとプロダクション環境の挙動の差異)
- [前提条件](#前提条件)
    - [Git](#git)
    - [protoc](#protoc)
    - [Docker](#docker)
  - [Dockerイメージの使用](#dockerイメージの使用)
- [環境構築](#環境構築)
  - [内部テスト](#内部テスト)
- [最初のテスト実装](#最初のテスト実装)
  - [IDEのオートコンプリーション活用](#ideのオートコンプリーション活用)
- [モックネットワークに対する手動テスト](#モックネットワークに対する手動テスト)
  - [ランダムなポートとディレクトリを使用する理由](#ランダムなポートとディレクトリを使用する理由)
- [The `gl-testserver`](#the-gl-testserver)
- [How to use `gl-testserver`](#how-to-use-gl-testserver)
  - [Running multiple tests in parallel](#running-multiple-tests-in-parallel)

---

## このチュートリアルの学習内容

* テスト用Dockerイメージの構築と活用方法
* Core Lightningノードによるテストネットワークの構築技術
* Greenlightテストフレームワーク`gl-testing`の実践的活用法
* Greenlightノードを用いた効率的なテスト実装
* `gl-testing`環境でのPython REPL活用技術

## テストの必要性と対象範囲

Greenlightを基盤とするアプリケーションを開発した場合、以下の検証が必須である：
1. 現時点での動作確認
2. 将来的な動作の継続性確保

プロダクション環境でのテストは可能だが、次の制約がある：
- ローカル環境と比較して実行速度が遅い
- Greenlightサービスのリソースを消費し、コストが発生する

ローカル環境で再現性の高いテストを実行できれば、開発効率は大幅に向上する。Greenlightはこの目的に特化したテストフレームワークを提供している。

## `gl-testing`テストフレームワークの概要

Greenlight [リポジトリ][repo]には、完全な機能を備えた（「バッテリー同梱」の）テストフレームワークが含まれている。このフレームワークは、Core Lightning開発にも使用されている[pyln-testing][pyln-testing]をベースとしている。

`gl-testing`は以下の機能を提供する：
- 任意の複雑さを持つLightningノードネットワークの構築
- Greenlightサービスのローカルモック環境の構築
- pyln-fixturesを活用した高度なテスト環境の実現
- 再現性の高いアプリケーションテスト手法
- 開発依存関係の隔離

### `gl-testing`とプロダクション環境の挙動の差異

`gl-testing`とプロダクションシステムの挙動における相違点は[`gl-testing` readme][gl-testing-diff]に記録されている。

このチュートリアルでは、`gl-testing`を使用して新規Greenlightクライアントの登録からノードでの請求書発行までのテスト実装方法を解説する。また、テストフレームワークやアプリケーションに対して手動コマンドを実行できるREPLの起動方法も説明する。

## 前提条件

#### Git
`gl-testing`テストフレームワークはGreenlight Githubリポジトリに含まれている。作業コピーを取得するには、システムに`git`のインストールが必要である。インストール方法の詳細は[git-guides][git-install]を参照のこと。

#### protoc
「モックネットワークに対する手動テスト」セクションではホストシステム上で`gl-client`をビルドする必要がある。Greenlightはクライアント・ノード間通信に[grpc][grpc]を使用するため、Protobufコンパイラ`protoc`が必要となる。[protoc][protoc]からインストール手順を確認すること。

#### Docker
Greenlightアプリケーションのテストには多数の依存関係（複数のCore Lightningバージョン、多数のPythonパッケージ、rustとcargo、protoファイルコンパイラなど）が必要である。開発環境の整理のため、`gl-testing`テストフレームワークには必要な依存関係をすべて含む__Dockerfile__が同梱されており、ビルドされたDockerイメージ内でテストを実行できる。

Dockerイメージのビルドと使用には、動作するDockerのインストールが必要である。オペレーティングシステムへのDockerセットアップ方法は[Docker manual][docker-install]を参照のこと。

### Dockerイメージの使用

Linuxホストでは依存関係の数が多いためDockerイメージの使用を強く推奨する。WindowsおよびMacOSでは、Core LightningがLinux向け`x86_64`の事前コンパイル版のみを提供しているため、現時点ではDockerイメージによるテストのみがサポートされている。

## 環境構築

`gl-testing`フレームワークを使ったテストを開始するには、リポジトリのローカルコピーを取得する必要がある。

```bash
git clone git@github.com:Blockstream/greenlight.git gl-testing-tutorial
```

以降のチュートリアルでは、クローンしたリポジトリ内で作業を進める。

```bash
cd gl-testing-tutorial
```

Greenlightリポジトリには便利なターゲットが定義された`Makefile`が含まれている。これを使用して、テストフレームワーク`gl-testing`の実行に必要なすべての依存関係を含むDockerイメージ`gltesting`をビルドする。

```bash
make docker-image
```

これでテスト環境が整備されたので、`gl-testing`で作業するために必要なすべての依存関係を含むシェルに入る。

```bash
make docker-shell
```

`exit`コマンドまたは`Ctrl-D`キーを押すことで、いつでもdocker-shellを終了できる。

### 内部テスト

Greenlight開発チームは`gl-testing`を使用して`gl-client`バインディング自体のテストも実施している。`gl-client`やその他のコンポーネントのプルリクエストを作成する場合、`gl-testing`を使用して変更内容を提出前にローカルでテストできる。

## 最初のテスト実装

gl-testingフレームワークは、クライアントをプログラム的に操作する方法が実装されている場合に最適に機能する。このチュートリアルでは、以下の基本的なテストを実装する：

1. テストネットワークの構築
2. Greenlightノードの起動
3. ネットワークからGreenlightノードへのチャネル開設
4. Greenlightノードでの請求書生成
5. ネットワーク内のノードからの支払い実行

```mermaid
flowchart LR
	A((CLN 1)) === B((CLN 2));
	B === C((GL 1));
	A -. payment .-> C;
```

まず、gl-testing-tutorialディレクトリのルートに`my-first-test.py`ファイルを作成する。gl-testingフレームワークは内部で[`pytest`][pytest]を使用しているため、Python開発者はテスト作成方法を理解しやすい。

```python
from gltesting.fixtures import *

def test_invoice_payment(node_factory, clients, bitcoind):
    print("Hello World!")
```

このコードはテストフィクスチャをインポートし、標準出力に"Hello World!"と表示する単純なテストを定義している。`test_`で始まる関数名はテストランナーによって検出され実行される。引数はテストに渡されるフィクスチャを表す（詳細は[pytest][pytest]を参照）。

テスト内で`gl-testing`のフィクスチャが正しくインポートできるかを確認する。テストを実行するには、前述のDockerイメージのシェルに入る必要がある。

```bash
make docker-shell
```

gl-client、gl-plugin、またはその他のバインディングなどGreenlightの一部を必要とするテストを実行する前に、これらのコンポーネントをDockerシェル内でビルドする必要がある。

```bash
make build-self
```

シェル内で`pytest`コマンドを実行してテストを開始する。詳細な出力を得るために`-v`オプションと、出力をキャプチャせずにコンソールに表示する`-s`オプションを追加する。

```bash
pytest -v -s my-first-test.py
```

実行結果の最後の数行は次のようになる：

```bash
Hello World!
PASSED
BitcoinRpcProxy shut down after processing 0 requests
Calling stop with arguments ()
Result for stop call: Bitcoin Core stopping

============================================================ 1 passed in 4.10s =============================================================
```

テスト実行に成功したので、次により実用的なテストを実装する。

最初のステップとして、後にGreenlightノードを接続するためのCore Lightningノードのネットワークを構築する。`pyln-testing`は一般的なタスクを処理するフィクスチャを提供しており、これらを使用してGreenlightノード以外のノードを起動・制御する。

```python
from gltesting.fixtures import *

def test_invoice_payment(node_factory, clients, bitcoind):
    # Create 2-node-network (l1)----(l2)
    l1, l2 = node_factory.line_graph(2)
    l2.fundwallet(sats=2*10**6)
```

このコードでは`node_factory`フィクスチャを使用して、既にチャネルが確立されている2つのCore Lightningノード`l1`と`l2`からなる`line-graph`ネットワークを構築している。これらのノードには統合されたRPCクライアントがあり、`l2`ノードに200万satoshiの資金を提供する。

### IDEのオートコンプリーション活用

IDEの自動補完機能を活用したい場合は、poetryで設定された環境のPythonインタープリタを`libs/gl-testing`から選択する。その後、フィクスチャからクラスをインポートし、型アノテーションを付与することができる。
例：
```python
from gltesting.fixtures import *
from gltesting.fixtures import Clients
from pyln.testing.fixtures import NodeFactory, LightningNode
...
def test_xyz(node_factory: NodeFactory, clients: Clients):
    nodes: list[LightningNode] = node_factory.line_graph(2)
    l1, l2 = nodes[0], nodes[1]
...
```

次にGreenlightノードの設定を行う。`clients`フィクスチャを使用して、固有のディレクトリ、サイナーシークレット、証明書を持つ新しいクライアントを作成する。`Client.register()`メソッドを呼び出してクライアントをGreenlightに登録し、`Client.node()`を使用して、登録されたクライアントに属するGreenlightノードをスケジュールして返す。`configure=True`引数はクライアント証明書を保存するようクライアントに指示する。

```python
from gltesting.fixtures import *

def test_invoice_payment(node_factory, clients, bitcoind):
    # Create 2-node-network (l1)----(l2)
    l1, l2 = node_factory.line_graph(2)
    l2.fundwallet(sats=2*10**6)

    # Register a new Greenlight client.
    c = clients.new()
    c.register(configure=True)
    gl1 = c.node()
```

これでテストフレームワーク上に最初のGreenlightノードが稼働した。

次に、Greenlightノードをネットワークに接続する。そのために、Greenlightノードとネットワーク間でチャネルを確立する必要がある。`l2`ノードをGreenlightノードのエントリーポイントとして選択し、`l2`と`gl`間のチャネルに資金を提供する。接続のハンドシェイク、チャネルへの資金提供、請求書の作成にはノードに対するサイナーが必要となる。Greenlightのサイナーはユーザー側の管理を維持するためクライアント側で実行される。まず、ノードがサイナーから署名を要求できるように、クライアントサイナーを起動する。

```python
from gltesting.fixtures import *

def test_invoice_payment(node_factory, clients, bitcoind):
    # Create 2-node-network (l1)----(l2)
    l1, l2 = node_factory.line_graph(2)
    l2.fundwallet(sats=2*10**6)

    # Register a new Greenlight client.
    c = clients.new()
    c.register(configure=True)
    gl1 = c.node()

    # Start signer and connect to (l2)
    s = c.signer().run_in_thread()
    gl1.connect_peer(l2.info['id'], f'127.0.0.1:{l2.daemon.port}')
```

次に`l2`と`gl`の間にチャネルを開設する。
チャネルが確立されるのを待つための補助関数`wait_for`をインポートする。
この関数は`gl1`のチャネル状態をポーリングし、チャネルが確認され完全に機能するようになった時点で返る。

```python
from gltesting.fixtures import *
from pyln.testing.utils import wait_for

def test_invoice_payment(node_factory, clients, bitcoind):
    # Create 2-node-network (l1)----(l2)
    l1, l2 = node_factory.line_graph(2)
    l2.fundwallet(sats=2*10**6)

    # Register a new Greenlight client.
    c = clients.new()
    c.register(configure=True)
    gl1 = c.node()

    # Start signer and connect to (l2)
    s = c.signer().run_in_thread()
    gl1.connect_peer(l2.info['id'], f'127.0.0.1:{l2.daemon.port}')

    # Fund a channel (l2)----(gl).
    # This results in the following network
    # (l1)----(l2)----(gl)
    l2.rpc.fundchannel(c.node_id.hex(), 'all')
    # Generate a block to synchronize and proceed
    # with the channel funding.
    bitcoind.generate_block(1, wait_for_mempool=1)
    # Wait for the channel to confirm.
    wait_for(lambda:
        gl1.list_peers().peers[0].channels[0].state == 'CHANNELD_NORMAL'
    )
```

請求書を作成して支払う前に、ゴシップ情報がすべてのノードに伝播するのを待つ必要がある。そうしなければ、請求書はルートヒントが不足した状態となり、唯一のチャネルが行き止まりだと見なされる。再び`wait_for`関数を使用して、両方のチャネルが双方向であるためネットワーク上で4つのチャネルエントリが見えることを確認する。

```python
from gltesting.fixtures import *
from pyln.testing.utils import wait_for

def test_invoice_payment(node_factory, clients, bitcoind):
    # Create 2-node-network (l1)----(l2)
    l1, l2 = node_factory.line_graph(2)
    l2.fundwallet(sats=2*10**6)

    # Register a new Greenlight client.
    c = clients.new()
    c.register(configure=True)
    gl1 = c.node()

    # Start signer and connect to (l2)
    s = c.signer().run_in_thread()
    gl1.connect_peer(l2.info['id'], f'127.0.0.1:{l2.daemon.port}')

    # Fund a channel (l2)----(gl).
    # This results in the following network
    # (l1)----(l2)----(gl)
    l2.rpc.fundchannel(c.node_id.hex(), 'all')
    # Generate a block to synchronize and proceed
    # with the channel funding.
    bitcoind.generate_block(1, wait_for_mempool=1)
    # Wait for the channel to confirm.
    wait_for(lambda:
        gl1.list_peers().peers[0].channels[0].state == 'CHANNELD_NORMAL'
    )

    # Wait for all channels to appear in our view of the network. We 
    # don't even have to wait for our channel to appear in l1s 
    # gossip: We can give a hint as soon as we know that our channel 
    # is not a dead end. We wait for 4 entries in our gossmap as both 
    # channels are bidirectional.
    bitcoind.generate_block(5)
    wait_for(
        lambda: len([c for c in gl1.list_channels().channels]) == 4 
    )
```

最後に請求書を作成して支払いを実行する。Greenlightノードで請求書を作成し、`l2`経由で`l1`ノードから支払いを行う。請求書を作成するために、Greenlightクライアント`glclient`からCore Lightningのプロトスタブ`clnpb`をインポートする。

```python
from gltesting.fixtures import *
from pyln.testing.utils import wait_for
from glclient import clnpb

def test_invoice_payment(node_factory, clients, bitcoind):
    # Create 2-node-network (l1)----(l2)
    l1, l2 = node_factory.line_graph(2)
    l2.fundwallet(sats=2*10**6)

    # Register a new Greenlight client.
    c = clients.new()
    c.register(configure=True)
    gl1 = c.node()

    # Start signer and connect to (l2)
    s = c.signer().run_in_thread()
    gl1.connect_peer(l2.info['id'], f'127.0.0.1:{l2.daemon.port}')

    # Fund a channel (l2)----(gl).
    # This results in the following network
    # (l1)----(l2)----(gl)
    l2.rpc.fundchannel(c.node_id.hex(), 'all')
    # Generate a block to synchronize and proceed
    # with the channel funding.
    bitcoind.generate_block(1, wait_for_mempool=1)
    # Wait for the channel to confirm.
    wait_for(lambda:
        gl1.list_peers().peers[0].channels[0].state == 'CHANNELD_NORMAL'
    )

    # Wait for all channels to appear in our view of the network. We 
    # don't even have to wait for our channel to appear at the gossmap
    # of l1: We can give a hint as soon as we know that our channel is 
    # not a dead end. We wait for 4 entries in our gossmap as both 
    # channels are bidirectional.
    bitcoind.generate_block(5)
    wait_for(
        lambda: len([c for c in gl1.list_channels().channels]) == 4 
    )
   
    # Create an invoice.
    bolt11 = gl1.invoice(
        amount_msat=clnpb.AmountOrAny(amount=clnpb.Amount(msat=100000)),
        description="test-desc",
        label ="test-label",
    ).bolt11

    # Pay invoice.
    l1.rpc.pay(bolt11)
```

これで`gl-testing`テストフレームワークとGreenlightノードを使用した最初のテストが完成した。このテストでは、Greenlightノードを含む3つのLightningノードからなる小規模なネットワークを構築し、Greenlightノードで請求書を作成して支払いを実行した。

テストの実行結果を確認するために、Dockerシェルから以下のコマンドを実行する：

```bash 
make docker-shell
```

```bash
pytest -vs my-first-test.py
```

テスト成功時には以下のような出力が表示される：

```bash
========================================== 1 passed in 30.83s ===========================================
```

## モックネットワークに対する手動テスト

既存のテストをステップ実行したり、ネットワークトポロジーを構築した後で対話的にシェルで操作したりする場合がある。これらの目的には`breakpoint()`が有効である。また、ホストから`gltesting`環境にアクセスするための設定にも`breakpoint()`を活用できる。

以下のテストは小規模なLightningネットワークを構築し、[REPL][repl]に移行してセットアップの確認や各種操作を実行できるようにする。REPLからアクセスできるようにするため、`scheduler`および`directory`フィクスチャも含めている：

```python
from gltesting.fixtures import *

def test_setup(clients, node_factory, scheduler, directory, bitcoind):
    """Sets up a gltesting backend and a small lightning network.

    This is meant to be run from inside the docker shell. See the 
    gltesting tutorial for further info.
    """
    l1, l2, l3 = node_factory.line_graph(3)
    
    # Assuming we want interact with l3 we'll want to print
    # its contact details:
    print(f"l3 details: {l3.info['id']} @ 127.0.0.1:{l3.daemon.port}")
    print()
    print(f"export GL_CA_CRT={directory}/certs/ca.pem")
    print(f"export GL_NOBODY_CRT={directory}/certs/users/nobody.crt")
    print(f"export GL_NOBODY_KEY={directory}/certs/users/nobody-key.pem")
    print(f"export GL_SCHEDULER_GRPC_URI=https://localhost:{scheduler.grpc_port}")
    
    breakpoint()
```

このコードの説明：
1. この時点で、3個のノードが直線状に配置されたネットワークが構築されている
2. Pythonコードを受け付けるREPLを起動する
3. モックスケジューラがリッスンしているポート番号を表示する
4. モックスケジューラと通信するための鍵ペアと証明書の場所を表示する

このテストを実行するには、まずDockerシェルに入る：

```bash
make docker-shell
```

次に、docker-shell内からREPLを起動する：

```bash
pytest -s examples/setup_repl.py
```

以下のような出力が表示される：

```bash
$ pytest -s testy.py
========== テストセッションを開始 ==========
platform linux -- Python 3.8.10, pytest-7.2.1, pluggy-1.0.0
rootdir: /repo
plugins: cov-3.0.0, xdist-2.5.0, forked-1.6.0, timeout-2.1.0
collected 1 item

testy.py /tmp/ltests-syfsnw83 でテストを実行中
[... ネットワーク設定に関する多くの行が表示される ...]

scheduler: https://localhost:44165
l3 details: **node_id** @ 127.0.0.1:40261
export GL_CA_CRT=/tmp/gltesting/**tmpdir**/certs/ca.pem
export GL_NOBODY_CRT=/tmp/gltesting/**tmpdir**/certs/users/nobody.crt
export GL_NOBODY_KEY=/tmp/gltesting/**tmpdir**/certs/users/nobody-key.pem
export GL_SCHEDULER_GRPC_URI=https://localhost:**scheduler_port**

>>>>>>>>>> PDB set_trace >>>>>>>>>>
--Return--
> /repo/testy.py(20)test_my_network()->None
-> breakpoint()
(pdb)
```

この時点でPythonコードを対話的に実行できるREPLが利用可能となる。

次に、ホストからモックスケジューラにクライアントアプリケーションを接続するための環境変数を設定する。docker-shellから以下の行をコピーしてホスト側のシェルで設定する：

```bash
export GL_CA_CRT=/tmp/gltesting/**tmpdir**/certs/ca.pem
export GL_NOBODY_CRT=/tmp/gltesting/**tmpdir**/certs/users/nobody.crt
export GL_NOBODY_KEY=/tmp/gltesting/**tmpdir**/certs/users/nobody-key.pem
export GL_SCHEDULER_GRPC_URI=https://localhost:**scheduler_port**
```

最初の3行はクライアントライブラリが読み込むアイデンティティと接続時のスケジューラのアイデンティティ検証方法を指定し、最後の行は本番環境ではなくモックスケジューラに接続するよう指示する。

### ランダムなポートとディレクトリを使用する理由

テストは通常並列実行されるため、相互の分離が必要である。ポートとディレクトリをランダム化しなければ、テスト同士が干渉し、デバッグが困難になり、テストの安定性が損なわれる。

これでホスト上に自由に操作できるクライアントを作成できる。

以下はサンプルアプリケーション（リポジトリ内の`examples/app_test.py`に収録）である：

```python
import os
import pytest
from glclient import Scheduler, Signer, TlsConfig,Node, nodepb

class GetInfoApp:
    """An example application for gltesting.
    
    This example application shows the process on how to register,
    scheduler and call against a gltesting environment greenlight 
    node.

    To execute this example set up the docker gltesting environment,
    drop into a REPL as explained in the gltesting tutorial.

    Then run the test below outside the gltesting docker container
    (run it from the host).
    `pytest -s -v app_test.py::test_getinfoapp`.
    """
    def __init__(self, secret: bytes, network: str, tls: TlsConfig):
        self.secret: bytes = secret
        self.network = network
        self.tls: TlsConfig = tls
        self.signer: Signer = Signer(secret, network, tls) # signer needs to keep running
        self.node_id: bytes = self.signer.node_id()

    def scheduler(self) -> Scheduler:
        """Returns a glclient Scheduler

        The scheduler is created from the attributes stored in this
        class.
        """
        return Scheduler(self.node_id, self.network, self.tls)

    def register_or_recover(self):
        """Registers or recovers a node on gltesting
        
        Also sets the new identity after register/recover.
        """
        res = None
        try:
            res = self.scheduler().register(self.signer)
        except:
            res = self.scheduler().recover(self.signer)
        
        self.tls = self.tls.identity(res.device_cert, res.device_key)

    def get_info(self) -> nodepb.GetInfoResponse:
        """Requests getinfo on the gltesting greenlight node"""
        res = self.scheduler().schedule()
        node = Node(self.node_id, self.network, self.tls, res.grpc_uri)
        return node.get_info()


def test_getinfoapp():
    # These are normally persisted on disk and need to be loaded and
    # passed to the glclient library by the application. In this 
    # example we store them directly in the "app".
    secret = b'\x00'*32
    network='regtest'
    tls = TlsConfig()

    # Register a node
    giap = GetInfoApp(secret, network, tls)
    giap.register_or_recover()

    # GetInfo
    res = giap.get_info()
    print(f"res={res}")
```

このサンプルアプリケーションはノードを登録し、Greenlightノードに対して`getinfo`を要求する。REPL上のgl-testing設定でこれを実行してみる。

ホスト上（docker-shell内ではなく）でサンプル用のPython環境を有効化する：

```bash
poetry shell
```

```bash
poetry install
```

最初のコマンドで[`poetry`][poetry]-shellを起動し、2番目のコマンドで`pyproject.toml`ファイルから必要な依存関係をインストールする。

docker-shell上のREPL設定とホスト上のpoetry-shellを使用して、テストアプリケーションを実行する：

```bash
pytest -s app_test.py::test_getinfoapp
```

以下のような出力が表示されれば成功である：

```bash
================== テストセッション開始 ==================
...                                                        
app_test.py::test_getinfoapp res=id: "\002\005\216\213l*\323c\354Y\252\023d)%mtQd\302\275\310\177\230\360\246\206\220\354,\\\233\013"
alias: "VIOLENTSPAWN-v23.05gl1"
color: "\002\005\216"
version: "v23.05gl1"
lightning_dir: "/tmp/gltesting/tmp/tmpdz6neih7/node-0/regtest"
our_features {
  init: "\010\240\210\n\"i\242"
  node: "\210\240\210\n\"i\242"
  invoice: "\002\000\000\002\002A\000"
}
blockheight: 103
network: "regtest"
fees_collected_msat {
}

PASSED

================== 1件のテストに合格 (0.10秒) ==================
```

この仕組みはホストが`/tmp/gltesting`ディレクトリをマウントすることでDockerコンテナとホスト間でファイルを共有し、docker-shellがホストのネットワークを再利用することで実現されている。そのため、ホスト上で実行されるクライアントやアプリケーションが、スケジューラやDockerコンテナ内のノードと直接通信できる。

テスト完了後は、REPLで`continue`または`Ctrl-D`を使用してシャットダウンする。

[docker-install]: https://docs.docker.com/engine/install/
[git-install]: https://github.com/git-guides/install-git
[repo]: https://github.com/Blockstream/greenlight
[pyln-testing]: https://github.com/ElementsProject/lightning/tree/master/contrib/pyln-testing
[pytest]: https://docs.pytest.org/en/7.2.x/
[gl-testing-diff]: https://github.com/Blockstream/greenlight/tree/main/libs/gl-testing#differences-between-greenlight-and-gl-testing
[stickers]: https://store.blockstream.com/product/sticker-bundle/
[repl]: https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop
[poetry]: https://python-poetry.org
[grpc]: https://grpc.io/
[protoc]: https://grpc.io/docs/protoc-installation/

## The `gl-testserver`

The `gl-testserver` は、プログラミング言語や開発環境に依存することなく、模擬の Greenlight サーバーを相手にテストを実施できるようにする、`gl-testing` フレームワークのスタンドアロン版です。

The `gl-testing` パッケージの目的は、開発者がローカル環境で模擬の Greenlight サーバーに対してテストを行えるようにすることです。これには以下のような数々の利点があります：

 - **Speed**：ネットワークを介さないため、遅延によるテストの低下がなく、テストを迅速に実施できます。テストはまた `regtest` ネットワーク上で実行されるため、待つことなくブロックの生成やトランザクションの確認が可能です。
 - **Cost**：`prod` ネットワークは無料ではなく、テストが任意のリソースを消費し、それらが後でクリーンアップされない（次の項目参照）ため、繰り返しテストを実行することでコストが発生する可能性があります。これを、開発中のテストを最小限に抑える悪いインセンティブと捉えており、`gl-testing` はローカルリソースのみを使用することで、テストを無料にし、より多くのテスト実施を促すことを期待しています。
 - **Reproducibility**：`prod` ネットワークでは、実際のユーザーがリソースを使用している可能性があるため、テストリソースのクリーンアップが許可されません。その結果、テスト実行間でテストアーティファクトが残存し、再現性のないテスト環境となります。ローカルで実行される `gl-testing` ではリソースのクリーンアップが可能なため、再現性のあるテストが実現できます。

しかしながら、`gl-testing` の欠点は、プログラミング言語として `python` およびテストランナーとして `pytest` に依存している点にあります。そこで登場するのが `gl-testserver` です。すべてのフィクスチャと起動ロジックをスタンドアロンのバイナリにまとめることにより、数秒でインスタンスを立ち上げ、対象に対してテストおよび開発を実施し、セッション終了時にそれを下げることが可能になります。

## How to use `gl-testserver`

おそらく、ソースツリーからスクリプトを実行するには uv を使用するのが最も簡単な方法です。まずは [`uv` installation instructions][uv/install] を参照して uv のインストール方法を確認し、その後ここに戻ってください。

`uv run gltestserver` を実行することが、このツールのエントリーポイントです：

```bash
gltestserver
Usage: gltestserver [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  run  Start a gl-testserver instance to test against.
```

現在、`run` サブコマンドのみがあります。このサブコマンドはテストサーバを起動し、スケジューラ GRPC インターフェイス、`bitcoind` RPC インターフェイス、および GRPC-web プロキシを `localhost` 上のランダムなポートにバインドします。

```bash
gltestserver run --help
Usage: gltestserver run [OPTIONS]

  Start a gl-testserver instance to test against.

Options:
  --directory PATH  Set the top-level directory for the testserver. This can
					be used to run multiple instances isolated from each
					other, by giving each isntance a different top-level
					directory. Defaults to '/tmp/'
  --help            Show this message and exit
```

現在のインスタンスのポートを特定するには、コマンドの出力の末尾に表示される整形されたキーと値のペアを確認するか、ポートの参照が含まれる `metadata.json` ファイルを `gl-testserver` サブディレクトリから読み込む方法があります（`--directory` オプションを指定していない場合は `/tmp/gl-testserver` になります）。

### Running multiple tests in parallel
	上記のヘルプテキストでも指摘されている通り、呼び出しごとに個別の `--directory` オプションを指定することで、複数のテストサーバを同時に任意の数だけ実行することが可能です。

	テストの実行速度を上げるために複数のテストを並行して実行したい場合に特に有用です。また、並行して実行されるインスタンスがポートで競合するため、各インターフェイスごとに固定のポートを使用せずにランダム化することでテスト間の分離性を確保しております。


起動すると、次の行が表示されます。

```
Writing testserver metadata to /tmp/gl-testserver/metadata.json
{
  'scheduler_grpc_uri': 'https://localhost:38209',
  'grpc_web_proxy_uri': 'http://localhost:35911',
  'bitcoind_rpc_uri': 'http://rpcuser:rpcpass@localhost:44135'
}
Server is up and running with the above config values. To stop press Ctrl-C.
```

この時点で、表示された URI を使用してサービスと対話するか、`Ctrl-C` を使用してサーバーを停止することができます。テスト環境で実行している場合は、プロセスに `SIGTERM` を送ることでプロセスを優雅にシャットダウンし、関連プロセスを終了させると同時に、テスト中に作成されたデータをディレクトリに残すことができます。

[uv/install]: https://docs.astral.sh/uv/getting-started/installation/
