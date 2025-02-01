---
title: "【Go言語】テストダブル入門 - 外部依存のないテストを書こう"
emoji: "🎭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "test", "unittest", "testing"]
published: true
---

# はじめに

みなさん、こんにちは！普段はGo言語でバックエンドの開発をしているエンジニアです。

テストを書いていて、こんな悩みを抱えたことはありませんか？

- 外部APIを呼び出すコードのテストを書きたいけど、実際にAPIを叩くわけにはいかない...😓
- データベースに依存する処理のテストを書きたいけど、テストデータの準備が面倒...🤔
- 特定のエラーが発生するケースをテストしたいけど、エラーを再現するのが難しい...😱

大丈夫です！そんな悩みを解決してくれるのが「テストダブル」なんです。

この記事では、テストダブルの基礎から実践的な使い方まで、具体的なコード例を交えながら詳しく解説していきます。
最後まで読むと、きっとテストの書き方が変わるはずです！

# テストダブルって何？🤔

テストダブルとは、テストの際に本物のオブジェクトの代わりに使用する「代役」のようなものです。

映画の撮影でスタントマンを使うように、テストでも本物の代わりとなる「代役」を使うことができるんです。
これにより、外部依存のない安定したテストを書くことができます。

でも、具体的にどんな問題が解決できるのでしょうか？実際のコードを見ながら確認していきましょう！

# テストダブルを使わないと...😱

まず、テストダブルを使用しない場合のコード例を見てみましょう。
以下は、外部APIを呼び出すシンプルなコードです。

このコードには、テストの観点から見ると大きく3つの問題点があります。

1つ目は、**テストが実際のHTTPリクエストを行うため、ネットワーク状態に依存してしまう**ことです。
これにより、テストの実行が不安定になってしまいます。

2つ目は、**エラーケースのテストが難しい**という点です。
ネットワークエラーや不正なレスポンスなど、様々なエラーパターンをテストしたいのですが、実際のAPIを使用していると、そういったケースを再現するのが困難です。

3つ目は、**テストの実行速度の問題**です。
外部APIの応答時間に左右されるため、テストの実行に時間がかかってしまう可能性があります。
これは、開発の効率を下げる要因となりかねません。

```go:internal/usecase/usecase.go
package usecase

import "sample/internal/apiclient"

type UseCase struct{}

func (uc *UseCase) CheckExampleCom() error {
	apiClient := &apiclient.ApiClient{}

	_, err := apiClient.Fetch()
	if err != nil {
		return err
	}

	return nil
}
```

```go:internal/usecase/usecase_test.go
package usecase

import (
	"testing"

	"github.com/google/go-cmp/cmp"
)

func TestUseCase(t *testing.T) {
	t.Run("正常ケース", func(t *testing.T) {
		useCase := &UseCase{}

		err := useCase.CheckExampleCom()

		if diff := cmp.Diff(nil, err); diff != "" {
			t.Error("errがnilであること", diff)
		}
	})
}
```

```go:internal/apiclient/apiclient.go
package apiclient

import (
	"io"
	"net/http"
)

type ApiClient struct{}

func (c *ApiClient) Fetch() (string, error) {
	response, err := http.DefaultClient.Get("https://www.example.com")
	if err != nil {
		return "", err
	}

	defer response.Body.Close()

	responseBody, err := io.ReadAll(response.Body)
	if err != nil {
		return "", err
	}

	return string(responseBody), nil
}
```

# テストダブルで解決しよう！💪

テストダブルを使えば、先ほどの問題点を一気に解決できます！具体的な実装方法を見ていきましょう。

## Step 1: インターフェースを定義しよう

Go言語でテストダブルを実装する第一歩は、インターフェースを定義することです。
これにより、本番環境とテスト環境で異なる実装を簡単に切り替えられるようになります。

先ほどの`ApiClient`を例に取ると、以下のようなインターフェースを定義できます。

```go
type ApiClient interface {
	Fetch() (string, error)
}
```

このインターフェースを使用して、先ほどのコードを以下のように書き換えます。

```go:internal/apiclient/apiclient.go
package apiclient

import (
	"io"
	"net/http"
)

// - `ApiClient`インターフェースの追加
//     - `Fetch()`メソッドを持つインターフェースを定義
//     - これにより、テスト時に実装を差し替えることが可能に
type ApiClient interface {
	Fetch() (string, error)
}

// apiClient is concrete implementation of ApiClient
type apiClient struct{}

func NewApiClient() ApiClient {
	return &apiClient{}
}

func (c *apiClient) Fetch() (string, error) {
	response, err := http.DefaultClient.Get("https://www.example.com")
	if err != nil {
		return "", err
	}

	defer response.Body.Close()

	responseBody, err := io.ReadAll(response.Body)
	if err != nil {
		return "", err
	}

	return string(responseBody), nil
}
```

```go:internal/usecase/usecase.go
package usecase

import "sample/internal/apiclient"

type UseCase struct{}

func (uc *UseCase) CheckExampleCom() error {
	// 現状では`NewApiClient()`を直接呼び出しているため、テストダブルの注入が難しい構造となっています。
	// より良いテスタビリティを実現するためには、以下のような改善が考えられます：
	// 
	// 1. `UseCase`構造体に`ApiClient`インターフェースをフィールドとして持たせる
	// 2. コンストラクタ経由で`ApiClient`の実装を注入できるようにする
	apiClient := apiclient.NewApiClient()

	_, err := apiClient.Fetch()
	if err != nil {
		return err
	}

	return nil
}
```

## Step 2: テストダブルを作ろう

インターフェースを定義したら、次はテストダブルを作成します。
今回は最もシンプルな「スタブ」を作ってみましょう。

```go:internal/apiclient/apiclient_stub.go
package apiclient

type apiClientStub struct{}

func NewApiClientStub() ApiClient {
	return &apiClientStub{}
}

func (c *apiClientStub) Fetch() (string, error) {
	return "", nil
}
```

`apiclient_stub.go`は、`ApiClient`インターフェースを実装したテストダブル(スタブ)です。

:::message
テストダブルにはいくつか種類があり、スタブはテストダブルの種類の１つです。
テストダブルの種類についてはここでは解説しません。以下の記事が参考になります。

https://goyoki.hatenablog.com/entry/20120301/1330608789
:::

このスタブには以下のようなメリットがあります。

1. 外部依存の排除
   - 実際のHTTPリクエストを行わないため、テストが高速で安定します
   - インターネット接続がない環境でもテストを実行できます

2. テストの制御性向上
   - 戻り値を固定値にすることで、テストシナリオを簡単にコントロールできます
   - エラーケースなど、実環境では再現が難しい状況もシミュレート可能です

3. テストの信頼性向上
   - 外部サービスの状態に依存せず、一貫した結果が得られます
   - テストの実行結果が予測可能になります

このようなスタブを利用することで、ユニットテストの品質と保守性を高めることができます。

## Step 3: テストを改善しよう

テストダブルを作成したら、実際にテストコードで使ってみましょう。
これにより、テストがどれだけ改善されるか見ていきます。

まずはusecase.goを変更します。
これまでは`CheckExampleCom`関数内で`apiClient := apiclient.NewApiClient()`を実行していましたが、
インスタンスの作成時に`ApiClient`インターフェースへの依存を受け取る形になっています。

```go:internal/usecase/usecase.go
package usecase

import "sample/internal/apiclient"

// - 依存性の注入
//     - `UseCase`構造体に`ApiClient`インターフェースを依存として追加
//     - これにより、実装とテストダブルを切り替えやすくなります
type UseCase struct {
	ApiClient apiclient.ApiClient
}

func (uc *UseCase) CheckExampleCom() error {
	_, err := uc.ApiClient.Fetch()
	if err != nil {
		return err
	}

	return nil
}
```

次にusecase_test.goを変更します。
これまでは`useCase.CheckExampleCom()`を実行しているだけでしたが、
usecase.goの変更に合わせて、テストダブルを作成し、UseCaseの依存として渡すようになっています。

```go:internal/usecase/usecase_test.go
package usecase

import (
	"sample/internal/apiclient"
	"testing"

	"github.com/google/go-cmp/cmp"
)

func TestUseCase(t *testing.T) {
	t.Run("正常ケース", func(t *testing.T) {
		// - テストダブルの導入
		//     - `apiclient.NewApiClientStub()`を使用して、本物のAPIクライアントの代わりにスタブを使用
		//     - これにより、実際のHTTPリクエストを行わずにテストを実行可能に
		apiClient := apiclient.NewApiClientStub()

		// - 依存性の注入
		//     - `UseCase`構造体の初期化時に、スタブを`ApiClient`フィールドに注入
		//     - インターフェースを使用することで、本番用の実装とテストダブルを簡単に切り替え可能
		// - テストの信頼性向上
		//     - 外部システムへの依存を排除することで、テストの実行が安定化
		//     - ネットワークの状態に関係なく、一貫した結果を得られる
		useCase := &UseCase{
			ApiClient: apiClient,
		}

		err := useCase.CheckExampleCom()

		if diff := cmp.Diff(nil, err); diff != "" {
			t.Error("errがnilであること", diff)
		}
	})
}
```

この変更で、テスト時には外部システムへGETアクセスが行われなくなり、テストの実行が安定化しました。
同時に、テスト実行時のネットワークの状態に関係なく、一貫した結果を得られるようになりました。

## おまけ：エラーケースもテストしよう！

ここまでの実装で、テストの基本的な改善はできました。でも、まだできることがあります！
テストダブルを使えば、エラーケースのテストも簡単にできるんです。

```go:internal/apiclient/apiclient_stub.go
package apiclient

// このフィールドに、`Fetch()`メソッドが返すエラーを保持
type apiClientStub struct {
	fetchErr error
}


// - `fetchErr`を引数に取り、`apiClientStub`のインスタンスを生成
//     - これにより、テストコードから柔軟にエラーケースを制御可能
func NewApiClientStub(fetchErr error) ApiClient {
	return &apiClientStub{
		fetchErr: fetchErr,
	}
}

// 正常系の場合は`nil`、異常系の場合は任意のエラーを返却可能
func (c *apiClientStub) Fetch() (string, error) {
	return "", c.fetchErr
}
```

```go:internal/usecase/usecase_test.go
package usecase

import (
	"fmt"
	"sample/internal/apiclient"
	"testing"

	"github.com/google/go-cmp/cmp"
	"github.com/google/go-cmp/cmp/cmpopts"
)

func TestUseCase(t *testing.T) {
	t.Run("正常ケース", func(t *testing.T) {
		// CheckExampleCom関数がnilを返却するように初期化
		apiClient := apiclient.NewApiClientStub(nil)

		useCase := &UseCase{
			ApiClient: apiClient,
		}

		err := useCase.CheckExampleCom()

		if diff := cmp.Diff(nil, err); diff != "" {
			t.Error("errがnilであること", diff)
		}
	})

	t.Run("異常ケース", func(t *testing.T) {
		fetchErr := fmt.Errorf("fetch error")

		// CheckExampleCom関数がfetchErrを返却するように初期化
		apiClient := apiclient.NewApiClientStub(fetchErr)

		useCase := &UseCase{
			ApiClient: apiClient,
		}

		err := useCase.CheckExampleCom()

		if diff := cmp.Diff(fetchErr, err, cmpopts.EquateErrors()); diff != "" {
			t.Error("期待通りのエラーが返却されること", diff)
		}
	})
}
```

この変更で、apiclient_stub.goが任意のエラーを返却するようになったため、
usecase_test.goに異常ケースを追加することができるようになりました。

# まとめ：テストダブルのメリット🎯

テストダブルを導入することで、以下のような素晴らしいメリットが得られました：

1. 外部サービスに依存せずテストが実行できる！
2. テストの実行時間が大幅に短縮！
3. エラーケースを含む様々なシナリオのテストが簡単に！

特にGo言語では、インターフェースを活用することで、テストダブルを非常にシンプルに実装できます。
これにより、保守性が高く、信頼性の高いテストを書くことができるんです。

テストダブルを使って、より良いテストを書いていきましょう！

# 参考資料📚

テストダブルについてもっと詳しく知りたい方は、以下の記事がおすすめです：

https://goyoki.hatenablog.com/entry/20120301/1330608789

---

この記事が皆さんのテスト実装の参考になれば幸いです。
もし質問や感想があれば、コメントで教えてください！
