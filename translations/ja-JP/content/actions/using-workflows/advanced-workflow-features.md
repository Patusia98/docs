---
title: Advanced workflow features
shortTitle: Advanced workflow features
intro: 'This guide shows you how to use the advanced features of {% data variables.product.prodname_actions %}, with secret management, dependent jobs, caching, build matrices, environments, and labels.'
redirect_from:
  - /actions/learn-github-actions/managing-complex-workflows
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
type: how_to
topics:
  - Workflows
miniTocMaxHeadingLevel: 4
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

## 概要

This article describes some of the advanced features of {% data variables.product.prodname_actions %} that help you create more complex workflows.

## シークレットを保存する

ワークフローでパスワードや証明書などの機密データを使用する場合は、これらを {% data variables.product.prodname_dotcom %} に _secrets_ として保存すると、ワークフローで環境変数として使用できます。 これは、YAML ワークフローに直接機密値を埋め込むことなく、ワークフローを作成して共有できることを示しています。

この例では、既存のシークレットを環境変数として参照し、それをパラメータとしてサンプルコマンドに送信する方法を示しています。

{% raw %}
```yaml
jobs:
  example-job:
    runs-on: ubuntu-latest
    steps:
      - name: Retrieve secret
        env:
          super_secret: ${{ secrets.SUPERSECRET }}
        run: |
          example-command "$super_secret"
```
{% endraw %}

詳しい情報については「[暗号化されたシークレットの作成と保存](/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)」を参照してください。

## 依存ジョブを作成する

デフォルトでは、ワークフロー内のジョブはすべて同時並行で実行されます。 したがって、別のジョブが完了した後にのみ実行する必要があるジョブがある場合は、`needs` キーワードを使用してこの依存関係を作成できます。 ジョブのうちの 1 つが失敗すると、依存するすべてのジョブがスキップされます。ただし、ジョブを続行する必要がある場合は、[`if`](/actions/using-jobs/using-conditions-to-control-job-execution) 条件ステートメントを使用してこれを定義できます。

この例では、`setup`、`build`、および `test` ジョブが連続して実行され、`build` と `test` は、それらに先行するジョブが正常に完了したかどうかに依存します。

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - run: ./setup_server.sh
  build:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - run: ./build_server.sh
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ./test_server.sh
```

For more information, see "[Defining prerequisite jobs](/actions/using-jobs/using-jobs-in-a-workflow#defining-prerequisite-jobs)."

## ビルドマトリックスを使用する

ワークフローでオペレーティングシステム、プラットフォーム、および言語の複数の組み合わせにわたってテストを実行する場合は、ビルドマトリックスを使用できます。 ビルドマトリックスは、ビルドオプションを配列として受け取る `strategy` キーワードを使用して作成されます。 たとえば、このビルドマトリックスは、異なるバージョンの Node.js を使用して、ジョブを複数回実行します。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [6, 8, 10]
    steps:
      - uses: {% data reusables.actions.action-setup-node %}
        with:
          node-version: {% raw %}${{ matrix.node }}{% endraw %}
```

For more information, see "[Using a build matrix for your jobs](/actions/using-jobs/using-a-build-matrix-for-your-jobs)."

{% ifversion fpt or ghec %}
## 依存関係のキャッシング

{% data variables.product.prodname_dotcom %} ホストランナーは各ジョブの新しい環境として開始されるため、ジョブが依存関係を定期的に再利用する場合は、これらのファイルをキャッシュしてパフォーマンスを向上させることを検討できます。 キャッシュが作成されると、同じリポジトリ内のすべてのワークフローで使用できるようになります。

この例は、`~/.npm` ディレクトリをキャッシュする方法を示しています。

```yaml
jobs:
  example-job:
    steps:
      - name: Cache node modules
        uses: {% data reusables.actions.action-cache %}
        env:
          cache-name: cache-node-modules
        with:
          path: ~/.npm
          key: {% raw %}${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}{% endraw %}
          restore-keys: |
            {% raw %}${{ runner.os }}-build-${{ env.cache-name }}-{% endraw %}
```

詳しい情報については、「<a href="/actions/guides/caching-dependencies-to-speed-up-workflows" class="dotcom-only">ワークフローを高速化するための依存関係のキャッシュ</a>」を参照してください。
{% endif %}

## データベースとサービスコンテナの利用

ジョブにデータベースまたはキャッシュサービスが必要な場合は、[`services`](/actions/using-jobs/running-jobs-in-a-container) キーワードを使用して、サービスをホストするための一時コンテナを作成できます。 この例は、ジョブが `services` を使用して `postgres` コンテナを作成し、`node` を使用してサービスに接続する方法を示しています。

```yaml
jobs:
  container-job:
    runs-on: ubuntu-latest
    container: node:10.18-jessie
    services:
      postgres:
        image: postgres
    steps:
      - name: Check out repository code
        uses: {% data reusables.actions.action-checkout %}
      - name: Install dependencies
        run: npm ci
      - name: Connect to PostgreSQL
        run: node client.js
        env:
          POSTGRES_HOST: postgres
          POSTGRES_PORT: 5432
```

詳しい情報については、「[データベースおよびサービスコンテナを使用する](/actions/configuring-and-managing-workflows/using-databases-and-service-containers)」を参照してください。

## ラベルを使用してワークフローを転送する

この機能では、特定のホストランナーにジョブを割り当てることができます。 特定のタイプのランナーがジョブを処理することを確認したい場合は、ラベルを使用してジョブの実行場所を制御できます。 You can assign labels to a self-hosted runner in addition to their default label of `self-hosted`. Then, you can refer to these labels in your YAML workflow, ensuring that the job is routed in a predictable way.{% ifversion not ghae %} {% data variables.product.prodname_dotcom %}-hosted runners have predefined labels assigned.{% endif %}

この例は、ワークフローがラベルを使用して必要なランナーを指定する方法を示しています。

```yaml
jobs:
  example-job:
    runs-on: [self-hosted, linux, x64, gpu]
```

A workflow will only run on a runner that has all the labels in the `runs-on` array. The job will preferentially go to an idle self-hosted runner with the specified labels. If none are available and a {% data variables.product.prodname_dotcom %}-hosted runner with the specified labels exists, the job will go to a {% data variables.product.prodname_dotcom %}-hosted runner.

To learn more about self-hosted runner labels, see ["Using labels with self-hosted runners](/actions/hosting-your-own-runners/using-labels-with-self-hosted-runners)."

{% ifversion fpt or ghes %}
To learn more about {% data variables.product.prodname_dotcom %}-hosted runner labels, see ["Supported runners and hardware resources"](/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources).
{% endif %}

{% ifversion fpt or ghes > 3.3 or ghae-issue-4757 or ghec %}
## Reusing workflows
{% data reusables.actions.reusable-workflows %}
{% endif %}

## 環境の使用

保護ルールとシークレットを持つ環境を設定できます。 ワークフロー内の各ジョブは、1つの環境を参照できます。 この環境を参照するとジョブがランナーに送信される前に、環境に設定された保護ルールをパスしなければなりません。 For more information, see "[Using environments for deployment](/actions/deployment/using-environments-for-deployment)."

## Using starter workflows

{% data reusables.actions.workflow-template-overview %}

{% data reusables.repositories.navigate-to-repo %}
{% data reusables.repositories.actions-tab %}
1. リポジトリに既存のワークフローが既に存在する場合: 左上隅にある [**New workflow（新しいワークフロー）**] をクリックします。 ![新規ワークフローの選択](/assets/images/help/repository/actions-new-workflow.png)
1. Under the name of the starter workflow you'd like to use, click **Set up this workflow**. ![このワークフローを設定します](/assets/images/help/settings/actions-create-starter-workflow.png)

## 次のステップ

To continue learning about {% data variables.product.prodname_actions %}, see "[Sharing workflows, secrets, and runners with your organization](/actions/learn-github-actions/sharing-workflows-secrets-and-runners-with-your-organization)."
