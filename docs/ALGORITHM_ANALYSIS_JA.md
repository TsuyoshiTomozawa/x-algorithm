# X For You アルゴリズム 解説（日本語）

本リポジトリ（2026年8月13日公開版）のコードを読み解き、For You タイムラインのロジックと、アカウントを伸ばすための実践的な指針をまとめたドキュメントです。

記載している数値はすべて、本リポジトリ内のコードに書かれた**本番デフォルト値**（`home-mixer/params/param.rs` などにcronで同期されている値）に基づいています。実際の配信では実験（A/Bテスト）により一部ユーザーで異なる値が使われることがあります。

## 目次

- [1. 全体像](#1-全体像--for-youはリクエストごとに組み立てられる)
- [2. スコアの中身](#2-スコアの中身--実際の重み)
- [3. スコア補正の仕掛け](#3-スコア補正の4つの仕掛け)
- [4. フィルタリング](#4-フィルタリング--シャドウバンの実体)
- [5. ラベルはどう付くか](#5-ラベルはどう付くか-labeling-path)
- [6. アカウントを伸ばす方法](#6-アカウントを伸ばす方法コードから導かれる結論)
- [7. まとめ](#7-一行でまとめると)
- [参照ファイル一覧](#参照ファイル一覧)

---

## 1. 全体像 — For Youはリクエストごとに組み立てられる

タイムラインはリクエストのたびに、以下のパイプラインで一から生成されます
（[`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`](../home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs)）。

```
① クエリ水和   → 閲覧者の直近の行動履歴・フォローリスト・ブロック/ミュート・既読投稿
② 候補取得     → In-Network  : thunder（フォロー中の最近の投稿、最大 1,200 件）
                 Out-of-Network: phoenix retrieval（最大 1,000 件）/ simclusters
③ 事前フィルタ → 17 種（48時間超、既読、ミュートワード、ブロック相手 …）
④ スコアリング → Phoenix が「各アクションの発生確率」を予測 → 重み付き線形和
⑤ 補正        → 相互フォローブースト / 著者多様性 / OON割引 / 新規著者コールドスタート
⑥ VMRanker    → DPP（行列式点過程）で似た投稿を分散させる
⑦ 可視性フィルタ→ visibility-filtering の判定で drop / interstitial
⑧ ブレンド    → 広告・おすすめユーザー・プロンプトを差し込み → 最終 35 件
```

主要な定数（[`home-mixer/params/config.rs`](../home-mixer/params/config.rs)）:

| 定数 | 値 | 意味 |
| --- | --- | --- |
| `MAX_POST_AGE` | `48 * 60 * 60` | 投稿は48時間で候補から消える |
| `RESULT_SIZE` | 35 | 1レスポンスの投稿数 |
| `TOP_K_CANDIDATES_TO_SELECT` | 50 | スコア上位から選ぶ件数 |
| `WHO_TO_FOLLOW_POSITION` | 6 | おすすめユーザーの挿入位置 |
| `NEGATIVE_SCORES_OFFSET` | 0.001 | 負のスコアを正の領域へ写像するオフセット |

### 設計思想として押さえるべき2点

1. **順位（ランキング）と可視性（表示可否）は完全に別系統**
   スコアがどれだけ高くても、`visibility-filtering` が `drop` と答えれば表示されません。逆に可視性が通っても、スコアが低ければ35件に入りません。この2つを混同しないことが理解の鍵です。

2. **候補同士は互いを参照しない**（Candidate Isolation）
   Transformer の推論時、候補は閲覧者コンテキストにしか attention しません。ある投稿のスコアは、同じバッチに何が入っているかに依存しない＝一貫性がありキャッシュ可能。

---

## 2. スコアの中身 — 実際の重み

`RankingScorer` の式は極めてシンプルです
（[`home-mixer/scorers/ranking_scorer.rs`](../home-mixer/scorers/ranking_scorer.rs)）。

```
最終スコア = Σ ( 重みᵢ × P(アクションᵢ) )
```

Phoenix が予測するアクションは以下のカテゴリに分かれます。

```
エンゲージメント  いいね・リプライ・リポスト・引用・共有・DM共有・リンクコピー共有
クリック          投稿・プロフィール・リンク・画像拡大・動画オープン・引用元
アテンション      動画完視聴(VQV)・dwell・滞在秒数・クリック後滞在秒数・アクティブ秒数
著者              著者フォロー
ネガティブ        興味なし・ミュート・ブロック・通報・素通り(not dwelled)
```

### ポジティブな重み（[`home-mixer/params/param.rs`](../home-mixer/params/param.rs)）

| アクション | パラメータ | 重み |
| --- | --- | --- |
| **リンクコピーで共有** | `share_via_copy_link` | **20.0** |
| **リプライ** | `reply` | **5.0**（相互フォローの原投稿は +15.0 = **20.0**） |
| **引用** | `quote` | **5.0** |
| **DMで共有** | `share_via_dm` | **5.0** |
| **著者をフォロー** | `follow_author` | **4.0** |
| 共有 | `share` | 2.0 |
| リポスト | `retweet` | 1.0 |
| **いいね** | `favorite` | **0.5** |
| 投稿クリック | `click` | 0.4 |
| リンクを開く | `open_link` | 0.2 |
| 画像拡大 | `photo_expand` | 0.05 |
| 動画オープン | `video_open` | 0.05 |
| 動画完視聴 | `vqv` | 0.05 |
| 引用元クリック | `quoted_click` | 0.05 |
| 未探索ボーナス | `post_unexplored` | 0.02（フォロー中のみ） |
| 滞在秒数 | `cont_dwell_time` | 0.004 |
| プロフィールクリック | `profile_click` | **0.0**（現在無効） |
| dwell（フラグ） | `dwell` | **0.0**（現在無効） |
| クリック後滞在 | `cont_click_dwell_time` | **0.0**（現在無効） |
| 引用元動画完視聴 | `quoted_vqv` | **0.0**（現在無効） |

### ネガティブな重み

| アクション | パラメータ | 重み |
| --- | --- | --- |
| **通報** | `report` | **-234.0** |
| ミュート | `mute_author` | -58.8 |
| 興味がない | `not_interested` | -43.2 |
| ブロック | `block_author` | -31.2 |
| 素通り（滞在せず） | `not_dwelled` | -0.02 |

### ⚠ 重みの読み方に関する重要な注意

コード内のコメントに明記されています。

> These weights reflect a combination of how much an action is valued in ranking and typical propensities of these actions across the X network (e.g. negative feedback is overall rare).
>
> （これらの重みは、ランキングにおけるそのアクションの価値と、Xネットワーク全体でのそのアクションの起こりやすさ ― 例えばネガティブフィードバックは全体的に稀 ― の組み合わせを反映している）

つまり「リプライはいいねの10倍の価値がある」という単純な読み方は誤りです。スコアは **確率 × 重み** なので、**稀にしか起きないアクションほど重みが大きく設定されている**（期待値のスケールを揃えるため）という側面があります。

ただし、**実務上の結論は変わりません**。稀で重い行動（リプライ・引用・共有・フォロー）を引き起こす投稿ほど、スコアの上振れ幅が大きくなります。

### 負スコアの扱い

```rust
fn offset_score(combined_score, w) -> f64 {
    if w.total_sum == 0.0        { combined_score.max(0.0) }
    else if combined_score < 0.0 { (combined_score + w.negative_sum) / w.total_sum * NEGATIVE_SCORES_OFFSET }
    else                         { combined_score + NEGATIVE_SCORES_OFFSET }
}
```

正味スコアが負になった投稿は `0 〜 0.001` の極小レンジに圧縮されます。実質的にタイムラインから排除されるのと同義です。

---

## 3. スコア補正の4つの仕掛け

### (A) 相互フォロー・ブースト — 最大のレバー

2026年7月に導入された変更で、経緯は [`docs/BIDIRECTIONAL_BOOST_CHANGE.md`](BIDIRECTIONAL_BOOST_CHANGE.md) に diff 付きで公開されています。

```rust
fn bidirectional_boost_eligible(candidate: &PostCandidate) -> bool {
    candidate.in_reply_to_tweet_id.is_none()            // リプライ投稿ではない
        && candidate.retweeted_tweet_id.is_none()       // リポストではない
        && candidate.is_mutual_follow_author == Some(true)  // 相互フォロー相手
}
```

条件を満たすと **リプライ重みが 5.0 → 20.0（+15.0）に4倍化** します。

| 日付 | ブースト値 | 備考 |
| --- | --- | --- |
| 2026-07-10 | 5 / 10 / 15 / 20 | A/Bテスト開始（大多数は 0） |
| 2026-07-13 | 20 | 多数のユーザーへ展開 |
| 2026-07-24 | **15** | ワールドカップの話題がフォロー外で見えにくいという声を受け引き下げ |

同時にテストされた `bidirectional_follow_dwell_weight_boost` は現在 **0.0**（未採用）です。

**条件の厳しさが最重要ポイント**：リプライ投稿・リポストには適用されません。**相互フォロー相手の「オリジナル投稿」のみ**が対象です。

### (B) 著者多様性ディケイ

同じ著者の投稿が候補に複数あると、2件目以降が減衰します。

```
倍率 = (1 - floor) × decay^k + floor
      = (1 - 0.25) × 0.5^k + 0.25       （k = その著者の何件目か、0始まり）
```

| k | 倍率 |
| --- | --- |
| 0（1件目） | 1.000 |
| 1（2件目） | 0.625 |
| 2（3件目） | 0.438 |
| 3（4件目） | 0.344 |
| 4以降 | 0.25（下限） |

k はスコア順にソートした結果で数えられるため、**同じ著者の投稿が候補に多いほど互いを削り合います**。

### (C) Out-of-Network（OON）割引

| 対象 | 倍率 | パラメータ |
| --- | --- | --- |
| フォロー外の投稿 | **× 0.75** | `oon_weight_factor` |
| トピック文脈での OON | × 0.5 | `topic_oon_weight_factor` |
| **フォロー中アカウントのリプライ投稿・リポスト** | **× 0.75** | `enable_oon_rescore_for_in_network_replies_retweets = true` |
| 新規ユーザー（閲覧者）向け OON | × 0.00001 | `NEW_USER_OON_WEIGHT_FACTOR`（`new_user_age_threshold_secs = 0` のため現在は実質無効） |

3行目が見落とされがちですが重要です。**フォロー中の相手であっても、リプライ投稿とリポストは25%減点される**。オリジナル投稿が構造的に有利です。

### (D) 新規著者コールドスタート — 小規模アカウントの救済枠

[`home-mixer/scorers/author_cold_start.rs`](../home-mixer/scorers/author_cold_start.rs)。以下を**すべて**満たす投稿のうち、スコア最良の **1件だけ** を、**15位相当のスコアまで引き上げ** ます（1リクエストにつき1件）。

| 条件 | 値 | パラメータ |
| --- | --- | --- |
| 著者のフォロワー数 | **≤ 1,000** | `cold_start_follower_cap` |
| その投稿の表示回数 | **< 1,000** | `cold_start_impression_threshold` |
| 投稿の種類 | **オリジナル投稿のみ** | リプライ・リポストは除外 |
| 元の順位 | 候補の上位 **85%** 以内 | `low_impressions_max_boost_position_ratio = 0.85` |
| 引き上げ先の順位 | 15〜16位相当 | `cold_start_slot_min / max` |
| 投稿の鮮度（treatment） | 24時間以内 | `cold_start_max_post_age_secs = 86400` |

機能自体は `enable_viewer_cold_start_boost = true` でデフォルト有効です。

> **フォロワー1,000人未満のアカウントには、コード上に明示的な露出枠が用意されている**、ということです。ただしリプライ・リポストは対象外です。

### (E) VMRanker（多様性の最終調整）

スコア確定後、[`vm-ranker/`](../vm-ranker/) が DPP（行列式点過程）で並べ替えます。**スコアを少し犠牲にして、隣接する投稿同士の類似度を下げる**処理です。似た投稿を連続で出さないための仕組みで、著者多様性ディケイとは別レイヤーで働きます。

---

## 4. フィルタリング — 「シャドウバン」の実体

### 事前フィルタ（[`home-mixer/filters/`](../home-mixer/filters/)、実行順）

| フィルタ | 除外するもの |
| --- | --- |
| `DropDuplicatesFilter` | 複数ソースから返った同一投稿 |
| `CoreDataHydrationFilter` | 本文・メタデータの取得に失敗した投稿 |
| `AgeFilter` | **48時間より古い投稿** |
| `SelfTweetFilter` | 閲覧者自身の投稿 |
| `OONRetweetReplyFilter` | フォロー外のリポスト・リプライ、親が欠けたリプライ |
| `OONNsfwSimclustersFilter` | 成人向けフラグの付いた著者の SimClusters 投稿 |
| `RetweetDeduplicationFilter` | 同一投稿の重複リポスト |
| `IneligibleSubscriptionFilter` | 閲覧権のないサブスク限定投稿 |
| `PreviouslySeenPostsFilter` / `...BackupFilter` | 既に表示済みの投稿 |
| `PreviouslyServedPostsFilter` | セッション内で既に配信済み |
| `MutedKeywordFilter` | ミュートキーワード一致 |
| `AuthorSocialgraphFilter` | ブロック・ミュートしている相手 |
| `VideoFilter` / `TopicIdsFilter` | リクエスト条件に合わないもの |
| `NewUserMinEngagementFilter` | 新規ユーザー向け、エンゲージメント閾値未満の OON 投稿 |
| `InventoryHoldoutFilter` | 実験用に決定論的に一定割合を除外 |

### 可視性フィルタ（[`visibility-filtering/rules/registry.rs`](../visibility-filtering/rules/registry.rs)）

**2段構えの構造**になっており、これが「シャドウバン」と呼ばれる現象の正体です。

#### 第1段：`timeline_home` ポリシー（全員に適用）

凍結・削除・消去済みアカウント、非公開アカウント、ブロック/ミュート相手、法的削除（DMCA・現地法）、`PDNA_DROP`、`SPAM_DROP`、`FOSNR_*`（ヘイト・暴力的発言・虐待・市民的統合性）、期限切れ投稿、年齢確認が必要な成人向けコンテンツ、NSFW/グロのインタースティシャル表示 など。

#### 第2段：`timeline_home_recommendations` ポリシー（**フォロー外へのおすすめ時のみ追加適用、drop のみ可能**）

```
SPAM_HIGH_RECALL_DROP                    スパム（高再現率＝広く引っかける）
DO_NOT_AMPLIFY_DROP                      増幅しない
MALICIOUS_URL_DROP                       悪質URL
NSFW_HIGH_RECALL / HIGH_PRECISION / TEXT / CARD_IMAGE
GORE_AND_VIOLENCE_HIGH_PRECISION_DROP
FOSNR_ABUSE_INSULTS_OON_DROP             侮辱（OON限定）
IMPERSONATION_HIGH_PRECISION_USER_DROP   なりすまし
COMPROMISED_USER_DROP                    乗っ取られアカウント
READ_ONLY_USER_DROP                      読み取り専用制限中
ABUSIVE_HIGH_RECALL_USER_DROP            攻撃的（高再現率）
NSFW_AVATAR_IMAGE_USER_DROP              アイコン画像が NSFW
NSFW_BANNER_IMAGE_USER_DROP              ヘッダー画像が NSFW
NSFW_NEAR_PERFECT_USER_DROP
DO_NOT_AMPLIFY_NON_FOLLOWER_USER_DROP
```

評価ルール：

- **最初に `drop` と答えたルールで評価は終了**する
- 第2段のルールは **`drop` しかできない**（`allow` で上書きはできない）
- **同じ投稿が、フォロワーには表示され、フォロー外のおすすめでは落とされる**

> 「投稿は普通に見えているのに、リーチだけが伸びない」現象のコード上の実体がこれです。

---

## 5. ラベルはどう付くか（Labeling Path）

可視性フィルタが読むラベルを生成するシステム群です。

| コンポーネント | 何を見るか |
| --- | --- |
| [`agatha/`](../agatha/) | 他者の反応からアカウントをラベル付け。**`ブロック数 ÷ いいね数`**、**`通報数 ÷ いいね数`**、**`スパム通報数 ÷ いいね数`** の比率。分母は **フォロー外からのいいね**（`readOutOfNetworkFavs`）。通報は180日窓、平滑化パラメータ 0.1 でベイズ的に補正 |
| [`bdsm/`](../bdsm/) | アカウントの行動シーケンスから、非真正・悪質な挙動を検出 |
| [`user-cred-v2/`](../user-cred-v2/) | フォローグラフ＋エンゲージメントエッジ上の PageRank をアカウントスコアに変換 |
| [`grox/`](../grox/) | 投稿公開時に LLM で分類。`flows/reply_spam/`（リプライスパム・**協調的スパム**）、`flows/ptos/`（安全性カテゴリ・スパム検出）、`flows/upa/`（"banger" スクリーニング）、`flows/mm_emb/`（マルチモーダル埋め込み） |
| [`media-model-proxy/`](../media-model-proxy/) / [`clip/`](../clip/) | 画像・動画の分類（成人向け・暴力・ヘイトシンボル・被写体） |
| [`botmaker/`](../botmaker/) / [`scarecrow/`](../scarecrow/) | イベント発生時にルールベースでラベルを付与 |
| [`abuse-enforcement-service/`](../abuse-enforcement-service/) | モデルスコアに基づきラベル付与・チャレンジ・凍結を実行 |

### agatha の比率設計が示すこと

分母が「いいね」、分子が「ブロック・通報」であるため、**インプレッションを稼いでもいいねが伴わず、代わりにブロックや通報が増えるような投稿は、比率が悪化してアカウントラベルが付きます**。

釣り・煽りによる短期のバズが、`ABUSIVE_HIGH_RECALL_USER_DROP` や `SPAM_HIGH_RECALL_USER_DROP` を通じて**長期のフォロー外リーチを恒久的に削る**構造になっています。

### grox の協調的スパム検出

[`grox/flows/reply_spam/classifier_coordinated_spam.py`](../grox/flows/reply_spam/classifier_coordinated_spam.py) には `CoordinatedSpamScorer` があり、スレッド全体をレンダリングして「協調的なスパムリプライ群」を判定します。単発では判定せず、`_drop_singleton()` で単独のものは除外し、複数アカウントによる協調パターンを検出する設計です。

### Under the Hood（透明性ツール）

[`under-the-hood/`](../under-the-hood/) は、自分のアカウントと投稿に付いた可視性影響ラベルの集計を確認できるツール（[x.com/i/under_the_hood](https://x.com/i/under_the_hood)）のバックエンドです。

> **リーチ改善の実務としては、まずここで自分にラベルが付いていないか確認するのが最優先です。** 施策を打つ前に、そもそもフィルタで落とされていないかを切り分けられます。

---

## 6. アカウントを伸ばす方法（コードから導かれる結論）

### 最優先

#### ① 相互フォローのネットワークを作る

リプライ重み 5.0 → **20.0（4倍）**。単一の補正としては群を抜いて大きい。ただし**オリジナル投稿にのみ**適用されます。

「相互フォローを増やす」ことが、そのまま自分の投稿の配信力になります。フォロワー数そのものより、**相互フォロー関係の数**が効きます。

#### ② リプライを誘発する投稿を書く

いいね（0.5）より遥かに重いリプライ（5.0 / 相互なら 20.0）。問いかけ、意見が割れる論点、経験談を引き出す形式が構造的に有利です。

#### ③ 「保存・共有される」投稿を作る

`share_via_copy_link` が **20.0** で全アクション中最大、`share_via_dm` が 5.0。

**「誰かに送りたい」「後で見返したい」** と思わせる実用情報・まとめ・図解が、公開エンゲージメント（いいね）よりずっと重く評価されます。**いいねを狙うより、DM転送とリンクコピーを狙う**方が正しい。

#### ④ オリジナル投稿を主軸にする

リプライ投稿・リポストは、たとえフォロワー相手でも **× 0.75** されます。さらに：

- 相互フォローブースト → 対象外
- コールドスタート枠 → 対象外

**リポストの連投は、タイムライン上での自分の露出を削ります。**

### 次点

#### ⑤ ネガティブを何よりも避ける

通報 -234.0、ミュート -58.8、興味なし -43.2、ブロック -31.2。ポジティブの最大値 20.0 と比べて桁が違います。

加えて agatha が比率でアカウントラベルを付け、可視性フィルタが**フォロー外への配信を恒久的に止める**という二重構造です。**一度の炎上より、継続的な低ブロック率が効きます。**

#### ⑥ 1日の投稿を分散させる

著者多様性ディケイにより、同一候補プール内の2件目は 0.625 倍、4件目は 0.34 倍。**短時間の連投は自分同士で共食い**します。時間を空けて投稿し、1件あたりの質を上げる方が総リーチは大きくなります。

#### ⑦ 48時間サイクルを回す

投稿は48時間で候補プールから完全に消えます（`MAX_POST_AGE`）。ストック型の資産にはならないため、継続的な投稿が前提の設計です。

#### ⑧ フォロワー1,000人未満なら「コールドスタート枠」を狙う

「フォロワー ≤ 1,000」かつ「表示回数 < 1,000」かつ「オリジナル投稿」の条件を満たす投稿は、1リクエストにつき1件、15位相当まで引き上げられます。

**この段階ではリプライ・リポストが対象外なので、特にオリジナル投稿に集中すべきです。**

#### ⑨ 滞在時間を稼ぐ／素通りを避ける

`cont_dwell_time` が正（0.004）、`not_dwelled` が負（-0.02）。冒頭のフックでスクロールを止めさせ、本文で読ませる。長文・スレッド・画像は滞在を伸ばします。

#### ⑩ プロフィールを整える

`follow_author` の重みは 4.0。**投稿からフォローに至る確率が直接スコアになります。** プロフィール文・固定投稿は、そのまま配信量に効きます。

なお `profile_click` の重みは現在 0.0 なので、プロフィールを見せること自体ではなく、**フォローに転換させること**が重要です。

### やってはいけないこと

| NG行動 | 該当する仕組み |
| --- | --- |
| 定型リプライの大量投下、相互いいね互助会 | `grox/flows/reply_spam/`（`CoordinatedSpamScorer`）→ `SPAM_HIGH_RECALL_DROP` |
| 釣り・煽りでインプレッションを稼ぐ | agatha のブロック/いいね比率が悪化 → OON配信停止 |
| アイコン・ヘッダーのきわどい画像 | `NSFW_AVATAR_IMAGE_USER_DROP` / `NSFW_BANNER_IMAGE_USER_DROP` |
| 短時間の連投 | 著者多様性ディケイ（0.625 → 0.44 → 0.34 → 0.25） |
| リポスト中心の運用 | × 0.75 ＋ ブースト対象外 ＋ コールドスタート対象外 |
| 短縮URLや怪しい外部リンク | `MALICIOUS_URL_DROP` |

### 補足：リンクと動画についての通説の検証

- **外部リンク**：`open_link` は **0.2 で正の重み**。「リンクを貼るとアルゴリズムに嫌われる」という通説は、少なくとも現在のコードには根拠がありません。ただし `MALICIOUS_URL_DROP` はあるので、リンク先の品質は問われます。
- **クリック後の滞在**：`cont_click_dwell_time` は現在 0.0（無効）。低いいね率へのペナルティ機構（`click_dwell_low_fav_rate_penalty`）も現在は off ですが、コードとしては実装済みです（＝将来的に「クリックさせるが滞在しない釣りリンク」が減点対象になり得る）。
- **動画**：`vqv` 0.05、`video_open` 0.05 と重みは低め。さらに動画完視聴のボーナスは **10秒超の動画**（`min_video_duration_ms = 10_000`）かつ **閲覧者のフォロワーが1万人未満**（`MAX_FOLLOWERS_THRESHOLD = 10_000`）の場合のみ適用されます（[`home-mixer/util/candidates_util.rs`](../home-mixer/util/candidates_util.rs)）。**動画は重み面では特別優遇されていません。**

---

## 7. 一行でまとめると

> **相互フォロー相手に届く、リプライと「誰かに共有したい」を引き起こす、48時間以内のオリジナル投稿を、時間を空けて継続的に出す。そしてブロック・ミュート・通報を一切招かない。**

これが、公開されているコードから導かれる最適戦略です。

---

## 参照ファイル一覧

| 内容 | ファイル |
| --- | --- |
| パイプラインの組み立て | [`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`](../home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs) |
| スコア計算・補正のロジック | [`home-mixer/scorers/ranking_scorer.rs`](../home-mixer/scorers/ranking_scorer.rs) |
| 重みなどのパラメータ本体 | [`home-mixer/params/param.rs`](../home-mixer/params/param.rs) |
| 定数（48時間、結果件数など） | [`home-mixer/params/config.rs`](../home-mixer/params/config.rs) |
| 新規著者コールドスタート | [`home-mixer/scorers/author_cold_start.rs`](../home-mixer/scorers/author_cold_start.rs) |
| 動画重みのゲート条件 | [`home-mixer/util/candidates_util.rs`](../home-mixer/util/candidates_util.rs) |
| 事前フィルタ群 | [`home-mixer/filters/`](../home-mixer/filters/) |
| 可視性ルールの登録簿 | [`visibility-filtering/rules/registry.rs`](../visibility-filtering/rules/registry.rs) |
| アカウント単位のドロップラベル | [`visibility-filtering/rules/user_label_drops.rs`](../visibility-filtering/rules/user_label_drops.rs) |
| ブロック/いいね比率のラベル生成 | [`agatha/scalding/labels/rate_based_labels/RateBasedLabels.scala`](../agatha/scalding/labels/rate_based_labels/RateBasedLabels.scala) |
| 協調的スパム検出 | [`grox/flows/reply_spam/classifier_coordinated_spam.py`](../grox/flows/reply_spam/classifier_coordinated_spam.py) |
| Phoenix モデル本体 | [`phoenix/README.md`](../phoenix/README.md) |
| 相互フォローブーストの変更履歴 | [`docs/BIDIRECTIONAL_BOOST_CHANGE.md`](BIDIRECTIONAL_BOOST_CHANGE.md) |

---

*本ドキュメントは公開リポジトリのコードを読んで作成した非公式の解説です。パラメータは実験により変動するため、最新の値は各ソースファイルを直接ご確認ください。*
