---
title: "GitHub ActionsからCloudflare Workersのpreviewを作成する。"
emoji: "🪂"
type: "tech"
topics:
  - "githubactions"
  - "cloudflare"
  - "cloudflareworkers"
published: true
published_at: "2025-05-02 13:17"
---

# 初めに
本記事では、Cloudflare Workers の本番環境ではなく、プレビュー環境に GitHub Actions を用いてデプロイする方法を紹介します。Dashboardからrepoへの接続もできますが、自分の環境ではgithub organizationにおいて、複数アカウントがCloudflareのGitHub Appに接続できなかったため、Actionsを利用してWorkersにデプロイしています。
主たる想定シーンは、Pull Request などの際に、本番環境へのデプロイは不要であるものの、URL を共有して他者に確認してもらいたい場合などです。

## GitHub Actionsとは
GitHub Actionsとは、GitHubが提供するワークフロー自動化サービスです。public repoであれば基本的に無料で利用できます。

## Cloudflare Workersとは
CLoudflare Workersとは、Cloudflareが提供するサーバーレスプラットフォームで、JavaScript(V8)などが動きます。[複数のフレームワーク](https://developers.cloudflare.com/workers/frameworks/)がサポートされており、無料枠が大きく商用利用も可能なので、気軽にページをデプロイするのに向いています。

Workersへのデプロイなどについては、公式のドキュメントに手順が詳しく記載されているためそちらを参考に進めていくのが良いと思います。以下のリンクはworkersにおけるReact Router v7の進め方です。
https://developers.cloudflare.com/workers/frameworks/framework-guides/react-router/


# GitHub Actionsでの手順
まず、[wrangler-action](https://github.com/cloudflare/wrangler-action)を利用して、previewにデプロイします。
previewにデプロイするには[`versions upload`](https://developers.cloudflare.com/workers/wrangler/commands/#upload)を利用します。
```yaml 
- name: Deploy workers preview
  id: deploy-workers-preview
  uses: cloudflare/wrangler-action@v3
  with:
    apiToken: ${{ secrets.CF_API_KEY }}
    accountId: ${{ secrets.CF_ACCOUNT_ID }}
    command: versions upload --message "Deployed pages by GitHub Actions branch ${{ github.ref_name }}"
```

上記のactionsにより、cloduflare workersにアップロードできます。
メッセージは適当に付与しているだけなので別のものでも問題ないです。

## pagesの場合
今回は、cloduflare workersと書いてありますが一応pagesの場合も以下に示します。
しかし、静的なページであってもstatic assetsなどを利用してworkersにデプロイすることが[推奨されています](https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/)。

```yaml
- name: Deploy pages preview
  id: deploy-pages-preview
  uses: cloudflare/wrangler-action@v3
  with:
    apiToken: ${{ secrets.CF_API_KEY }}
    accountId: ${{ secrets.CF_ACCOUNT_ID }}
    command: pages deploy ./dist --project-name "${projectName}" --branch="${{ github.head_ref }}"
```
`${projectName}`を適切な名前に置き換えてください。

pull requestの場合`${{ github.head_ref }}`でbranchの名前が取得できるので取得してきて指定しています。

個人的には、pagesのbranchごとに発行されるurlが便利で気に入っていたのでworkersにないのが残念です。

## pull requestにコメントをつける
上記の方法でデプロイまでは実装できました。
しかし、いちいち発行されたurlを確認しに行くのは面倒です。
以下の画像のようなDashboardからの連携の際に発行されるコメントが欲しくなってきます(これはpagesの連携)。
![Github appを利用した際に発行されるコメント(url付き)
](https://storage.googleapis.com/zenn-user-upload/bc2332189215-20250502.png)
そこで、actionsからurlを取得し、pull requestにコメントを付与することとします。
```yaml
- name: Create comment file
  id: create-comment-file
  env:
    DEPLOYMENT_URL: '${{ steps.deploy-workers-preview.outputs.deployment-url }}'
    PAGES_DEPLOYMENT_URL: '${{ steps.deploy-pages-preview.outputs.deployment-url }}'
    PAGES_BRANCH_URL: '${{ steps.deploy-pages-preview.outputs.deployment-alias-url }}'
  run: |
          cat  << EOF > comment.md
          ## 🚀 Deploying ${{ github.event.repository.name }} with Cloudflare Workers
          <table>
            <tr>
              <th scope="row">Workers Preview URL</th>
              <td><a href="$DEPLOYMENT_URL">$DEPLOYMENT_URL</a></td>
            </tr>
            <tr>
              <th scope="row">Pages Preview URL</th>
              <td><a href="$PAGES_DEPLOYMENT_URL">$PAGES_DEPLOYMENT_URL</a></td>
            </tr>
            <tr>
              <th scope="row">Pages branch URL</th>
              <td><a href="$PAGES_BRANCH_URL">$PAGES_BRANCH_URL</a></td>
            </tr>
          </table>
          EOF
```

ここでは、urlの取得を初めに行い、envにある変数に入れています。
`DEPLOYMENT_URL: '${{ steps.deploy-workers-preview.outputs.deployment-url }}'`がこれに当たります。`deploy-workers-preview`はstepのidを指定して取得します。
pagesの場合は、`steps.<id>.outputs.deployment-alias-url`でbranch用のurlを取得できます。

その後、url書き込んだmarkdownファイルを作成して、`gh pr`コマンドを利用してコメントを付与しています。

```yaml
- name: Create PR comment
  if: >-
    ${{ steps.deploy-workers-preview.outcome == 'success' &&
    steps.deploy-pages-preview.outcome == 'success' }}
  run: 'gh pr comment ${{ github.event.number }} --body-file comment.md'
  env:
    GITHUB_TOKEN: '${{ secrets.GITHUB_TOKEN }}'
```

なお、書き込みには権限が必要なので、`Settings > Actions > General`の`Workflow permissions`を適切に設定し、workflowファイルにもpermissionを付与する必要があります。
```yaml
permission:
  contents: write
  pull-requests: write
```

:::message
補足
実装からいくつか書き直したのでもしかしたらtypoなどがあるかもしれません。
私の実環境で動かしている[file](https://github.com/Nlkomaru/color-palette/blob/748c25ab79ee290b7bb5465e313f000e1f07b122/.github/workflows/preview.yml)をこちらに置いておきます。
あと、`target="_blank`をaタグに付与していますが、無効化されるみたいなので意味はなかったです。
:::

## 参考文献

https://docs.github.com/ja/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
https://community.cloudflare.com/t/is-cloudflare-pages-workers-free-plan-free-for-commercial-use/291741
https://x.com/yusukebe/status/1917869496267915641
https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/
https://github.com/cloudflare/wrangler-action/issues/81