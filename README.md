# crates.io の Trusted Publishing（公式ドキュメント日本語訳）

> **これは crates.io の公式ドキュメント [Trusted Publishing](https://crates.io/docs/trusted-publishing) の、非公式な日本語訳です。** crates.io チームや Rust Project による確認や公認を受けたものではありません。
>
> **読むのはこちら: https://rustsecjp.github.io/trusted-publishing-ja/**
>
> This is an unofficial Japanese translation of the crates.io "Trusted Publishing" documentation.

Trusted Publishing は、長期間有効な API トークンをリポジトリに置かずに、GitHub Actions や GitLab CI/CD から OIDC でクレートを公開するための仕組みです。公開アカウントやトークンの乗っ取りによるサプライチェーン攻撃への対策になります。

原文には独立した README が無いので、この README は日本語訳の案内だけを載せています。

## 訳した版

| | |
|---|---|
| 原文の版 | 2026-11-20 時点（コミット [`97cd67a`](https://github.com/rust-lang/crates.io/commit/97cd67a99f1c2e97708ad3bff717e7397fd106cb)） |
| 翻訳日 | 2026-11-20 |

原文がこの版より後に更新されている場合、訳は古い内容のままです。

## 出典とライセンス

- 原文: https://crates.io/docs/trusted-publishing （ソース: [rust-lang/crates.io](https://github.com/rust-lang/crates.io) の `svelte/src/routes/docs/trusted-publishing/+page.svelte`）
- 原著者: crates.io チーム（The Rust Project）
- ライセンス: MIT または Apache-2.0。全文は [LICENSE-MIT.txt](LICENSE-MIT.txt) と [LICENSE-APACHE.txt](LICENSE-APACHE.txt)。
- 訳した版: 2026-11-20 時点（コミット 97cd67a99f1c2e97708ad3bff717e7397fd106cb）
- 変更点: 日本語への翻訳。HTML から Markdown への書式の変換。

誤訳の指摘や訳語の提案は、Issue か Pull Request でお願いします。

RustSecJP は非公式のコミュニティです。crates.io、Rust Project、Rust Foundation とは提携していません。
