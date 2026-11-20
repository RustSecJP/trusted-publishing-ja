{% raw %}
# Trusted Publishing

## Trusted Publishing とは

Trusted Publishing は、GitHub Actions や GitLab CI/CD などの CI/CD プラットフォームから、API トークンを手作業で管理せずに、Rust のクレートを安全に公開するための仕組みです。OpenID Connect（OIDC）を使って、ワークフローがあなたのリポジトリから実行されていることを検証し、公開用の短命なトークンを発行します。

長期間有効な API トークンをリポジトリのシークレットに保存する代わりに、CI/CD プラットフォームが、ワークフローの身元を証明する暗号署名つきのトークンを使って、crates.io に直接認証します。

**注:** GitLab CI/CD への対応は、現在パブリックベータです。

### セキュリティ上の利点

- 管理やローテーションが必要な**長期間有効な API トークンが無い**
- **トークンは 30 分で自動的に失効する**
- **リポジトリとワークフローの検証**により、許可されていない公開を防ぐ
- プラットフォームの公開 JWKS を使った、**OIDC に基づく暗号学的な検証**
- 追加のアクセス制御のための、**任意の environment（環境）**

## クイックスタート

次の手順で、クレートに Trusted Publishing を設定します。

1. crates.io の設定で、**クレートに Trusted Publishing を設定する**
2. 必要な権限と認証を備えた **CI/CD のワークフローを用意する**
3. 自動化したワークフローで**クレートを公開する**

### 前提条件

- クレートがすでに crates.io に公開されていること（最初の公開には API トークンが必要です）
- crates.io で、そのクレートの所有者であること
- リポジトリが GitHub または GitLab にあること

## Trusted Publishing を設定する

crates.io でクレートを設定します。

1. クレートの Settings → Trusted Publishing を開く
2. 「Add」ボタンを押し、プラットフォーム（GitHub または GitLab）を選ぶ
3. プラットフォームごとの項目を入力し、設定を保存する

### GitHub の設定

- **Repository owner:** GitHub のユーザー名または組織名
- **Repository name:** リポジトリの名前
- **Workflow filename:** GitHub Actions のワークフローのファイル名（例: 「release.yml」）
- **Environment:** GitHub の environment を使っている場合の、environment の名前（任意）

### GitLab の設定

- **Namespace:** GitLab のユーザー名またはグループのパス
- **Project:** プロジェクトの名前
- **Workflow filepath:** GitLab CI/CD のワークフローの、完全なファイルパス（例: 「ci/release.yml」）
- **Environment:** GitLab CI/CD の environment を使っている場合の、environment の名前（任意）

## GitHub Actions の設定

`.github/workflows/release.yml` にワークフローのファイルを作ります。次の例は、バージョンのタグ（`v1.0.0` など）を push するたびに、クレートを自動で公開するワークフローです。

```yaml
name: Publish to crates.io
on:
  push:
    tags: ['v*']  # 「v」で始まるタグを push したときに実行する
jobs:
  publish:
    runs-on: ubuntu-latest
    environment: release  # 任意: セキュリティを強めるため
    permissions:
      id-token: write     # OIDC トークンの交換に必要
    steps:
    - uses: actions/checkout@v6
    - uses: rust-lang/crates-io-auth-action@v1
      id: auth
    - run: cargo publish
      env:
        CARGO_REGISTRY_TOKEN: ${{ steps.auth.outputs.token }}
```

**任意:** セキュリティを強めるには、リポジトリの設定で「release」という名前の GitHub Actions の environment を作り、必須のレビュアーやデプロイ用ブランチなどの保護ルールを付けます。

## GitLab CI/CD の設定（パブリックベータ）

`.gitlab-ci.yml` にワークフローのファイルを作ります。次の例は、バージョンのタグ（`v1.0.0` など）を push するたびに、クレートを自動で公開するワークフローです。

```yaml
publish:
  image: rust:1.91.0-alpine
  environment: release
  only:
    - tags  # タグを push したときだけ実行する
  id_tokens:
    CRATES_IO_ID_TOKEN:
      aud: crates.io
  before_script:
    - apk add --no-cache bash curl jq
  script:
    # OIDC トークンを公開用のトークンに交換する
    - CARGO_REGISTRY_TOKEN=$(bash exchange-token.sh)
    # crates.io に公開する
    - CARGO_REGISTRY_TOKEN="$CARGO_REGISTRY_TOKEN" cargo publish
```

リポジトリのルートに、補助スクリプト `exchange-token.sh` を作ります。

```bash
#!/bin/bash
set -e

# Exchange JWT token
echo "Exchanging OIDC token..." >&2
RESPONSE=$(curl -s -X POST https://crates.io/api/v1/trusted_publishing/tokens \
  -H "Content-Type: application/json" \
  -H "User-Agent: gitlab-trusted-publishing (your@email.com)" \
  -d "{\"jwt\": \"$CRATES_IO_ID_TOKEN\"}")

# Extract publish token
CRATES_IO_PUBLISH_TOKEN=$(echo "$RESPONSE" | jq -r '.token')

if [ "$CRATES_IO_PUBLISH_TOKEN" = "null" ] || [ -z "$CRATES_IO_PUBLISH_TOKEN" ]; then
  echo "Failed to get upload token" >&2
  echo "$RESPONSE" >&2
  exit 1
fi

echo "$CRATES_IO_PUBLISH_TOKEN"
```

**任意:** セキュリティを強めるには、リポジトリの設定で「release」という名前の GitLab CI/CD の environment を作り、必須のレビュアーやデプロイ用ブランチなどの保護ルールを付けます。

## セキュリティとベストプラクティス

- **ワークフローのファイル名を具体的に指定し**、攻撃対象領域を減らす
- 慎重に扱うべき公開には、**保護ルールつきの environment を使う**
- **ワークフローのトリガーを**、特定のタグや保護されたブランチに**限定する**
- リリース用のワークフローで**使うアクションをすべて確認する**
- crates.io のメール通知で、**公開の動きを監視する**

**しくみ:** CI/CD プラットフォームが、ワークフローの身元を証明する OIDC トークンを生成します。GitHub の場合は、`rust-lang/crates-io-auth-action` がこれを 30 分間有効なアクセストークンに交換します。GitLab の場合は、上のスクリプトが crates.io の API を通じてトークンを交換します。`cargo publish` は、このトークンを自動的に使います。

## API トークンからの移行

API トークンから移行するには、上の手順で Trusted Publishing を設定し、動作を確かめてから、リポジトリのシークレットから API トークンを削除します。移行の間は、両方の方法を同時に使えます。

## 関連資料

- [RFC 3691: Trusted Publishing for crates.io](https://rust-lang.github.io/rfcs/3691-trusted-publishing-cratesio.html)
- [GitHub: About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [GitHub: Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [GitLab: OpenID Connect (OIDC) authentication](https://docs.gitlab.com/ee/ci/secrets/id_token_authentication.html)
- [GitLab: Environments and deployments](https://docs.gitlab.com/ee/ci/environments/)
{% endraw %}
