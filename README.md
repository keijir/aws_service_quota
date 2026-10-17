# AWS Service Quota 共通データセット

AWS Service Quota Explorer と AWS Architecture Workbench が共有する、AWS公式情報のデータです。
ユーザーの判断 (プロジェクト) は含みません。すべてJSONで、ほかのツールからも読み込めます。

| ファイル | 内容 | 更新方法 |
|---|---|---|
| `quotas.json` | 全サービスのQuota (公式表から自動取得) | `npm run fetch-quotas` |
| `changes.json` | 取得のたびの差分 (追加・削除・値などの変化) | `npm run fetch-quotas` が追記 |
| `curated.json` | 公式ドキュメントで個別に確認した値・公開上限・適用範囲 | 手作業 |
| `relations.json` | サービス間の依存ルール (関連Quotaの発見に使用) | 手作業 |
| `ja.json` | Quota名・説明・表の注記の日本語参考訳 (AI翻訳)。キーは英語の原文 | 翻訳時に追加 (原文が変わった・新しいQuotaは英語のみで表示) |

## 取得元と範囲

- 取得元: [AWS General Reference — Service endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/aws-service-information.html) の各サービスページと、そこからリンクされた各サービスのガイドのQuotaページ (公開ページ、AWSアカウント不要)。General Reference にないサービス (Site-to-Site VPN, Client VPN) はガイドのページを直接取得。
- 値は `ap-northeast-1` (東京) の既定値。リージョン別の記載がない場合は全リージョン共通の値。
- **含まれないもの**: アカウントに適用されている値 (認証が必要な Service Quotas API でのみ取得可能)、引き上げ後の上限値 (公式表に記載がない。`curated.json` の個別確認分のみ)、各サービスのユーザーガイドだけに記載されたQuota。
- 網羅性は実証されていません。収録状況は `quotas.json` の各サービスの `status` で確認できます。

## quotas.json

```
{ schemaVersion: 1, region, fetchedAt, source, tocPages,
  services: [{
    id,                 // ページ名由来 (例: "lambda-service")
    service, fullName?, source,
    status,             // ok | external (Quota表がなく各ガイドに案内) | no-quotas (公式に「Quotaなし」) | no-quota-info | error (取得失敗。前回の値を保持)
    error?, serviceCodes, quotaLinks: [{ title, url }], guideErrors?,  // ガイドの取得失敗 (前回の行を保持)
    quotas: [{
      id,               // "<service id>/<Quotaコード>" またはコードがなければ "<service id>/<名前のslug>"
      name, default,    // default: 表の記載 (東京の値) の原文。null = 東京/全リージョン共通の値を特定できない
      byRegion?, defaultRaw?,   // リージョン別の記載がある場合は原文を保持
      value, unit,      // default を数値と単位に分解。数値にできない場合 value = null
      adjustable,       // true / false / null (表に記載なし)。true でも上限値が公開されているとは限らない
      serviceCode?, code?,      // Service Quotas のサービスコードとQuotaコード (L-…)
      description, extra?,      // 解釈していない列 (Notes 等) は原文のまま
      source?           // ガイドのページから取得した行のみ、そのURL
    }]
  }] }
```

## curated.json

`annotates` があるものは `quotas.json` のQuotaへの補足 (`verifiedValue` は確認時の値。現在値と異なれば要再確認)。
`annotates` がないものは公式表にないQuotaで、`serviceId` のサービスに属します。

## relations.json

`from` のサービスを使うと `to` のサービスの `quotas` に影響しうる、というルールです。

- `type`: `direct` (常に関係) / `conditional` (`condition` に当てはまる場合) / `possible` (関係する可能性)
- `certainty`: `documented` (依存関係が `source` の公式資料に記載) / `rule` (本ツールの経験則)
- `reason` (なぜ関係するか)、`check` (何を確認するか)、`priority`

## changes.json

`entries` は新しい順。各エントリは `from`/`to` (取得日時)、`added` (QuotaID)、`removed`、`changed` (`{ id, field, before, after }`)、`services` (収録状況の変化)。
取得に失敗したサービスは比較から除外するため、一時的なエラーで「削除」とは記録されません。
