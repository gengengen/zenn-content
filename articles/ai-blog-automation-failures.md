---
title: "AIブログ完全自動化で盛大にコケた5つの失敗【PHP+Claude API+cron】"
emoji: "💥"
type: "tech"
topics: ["PHP", "Claude", "API", "自動化", "アフィリエイト"]
published: true
---

プログラマーなら一度は夢見る「完全自動化ブログ」。

記事生成、画像生成、SEO対策、SNS投稿、商品データ同期——すべてをAIとcronジョブに任せて、自分は寝ているだけで記事が量産され、アフィリエイト収益が入ってくる。

そんな甘い夢を見て、実際にシステムを組み上げた。PHP + Claude API + Gemini API + 楽天/Amazon API。19本のcronジョブが24時間365日動き続ける仕組みだ。

結論から言うと、**動いている。今は。**

100記事以上、700枚以上の画像が完全自動で生成され、毎週6記事が勝手に公開されている。でもここに至るまでに、数え切れないほど盛大にコケた。

この記事では、そのなかでも特に「これはヤバかった」という5つの失敗を、実際のPHPコードとともに正直に書く。AIで副業を考えている人、自動化に興味があるエンジニアの参考になれば嬉しい。

## 失敗1: AIが「使ってもいない商品」の体験談を捏造した

ある日、自動生成されたキャンプ用品のレビュー記事を読み返して、血の気が引いた。

> 「実際に使ってみると、このテントは想像以上に広く、大人4人でもゆったり過ごせました」

AIはテントを使ったことがない。そもそも物理的な体を持っていない。

これはGoogleが重視するE-E-A-T（Experience, Expertise, Authoritativeness, Trustworthiness）の「Experience」を偽装していることになる。ペナルティリスクだけでなく、読者への誠実さの問題でもある。

厄介なのは、AIに「嘘を書くな」と指示しても効果が薄いこと。システムプロンプトに「架空の体験談は禁止」と書いても、AIは「レビューを要約しているだけ」という体で、一人称の体験談をナチュラルに生成してしまう。LLMはプロンプトの「意図」を汲むのではなく、学習データ中の「レビュー記事の文体」を再現しようとする。結果、人間のレビュアーと見分けがつかない架空体験談が量産される。

### どう解決したか

「禁止する」のではなく、「書けない仕組み」を作った。

テーマごとに「禁止パターン」と「推奨パターン」を定義する`ThemeProfile`というアーキテクチャを導入した。`ThemeProfileInterface`を実装するクラスがテーマごとに存在し、`ThemeProfileFactory`で取得する設計だ。

```php
// CampingThemeProfile.php - E-E-A-T信号定義

public function getEEATSignals(): array
{
    return [
        'evidence_patterns' => [
            '楽天レビュー{N}件中、"{keyword}"という評価が{percent}%を占めます',
            'メーカー公称値{spec}。同クラスの平均が{average}なので、約{diff}%{better_worse}です',
            'レビューでは{scenario}での使用報告が多く、{feedback}という声が目立ちます',
            '購入者レビューによると、{target}層からの評価が特に高い傾向があります',
        ],
        'prohibited_patterns' => [
            '実際に使ってみると',
            'フィールドで試した結果',
            '初めて使ったときは',
            '筆者が',
            '私が',
        ],
        'instruction' => "## 体験描写のルール（厳守）
- 架空の一人称体験談は絶対に書かない
  （「実際に使ってみると〜」「筆者が試したところ〜」は禁止）
- 代わりに、レビューデータ・メーカー公称値・比較データに基づく
  エビデンスベースの記述を使う
- 「レビューによると〜」「メーカー公称値では〜」
  「購入者の声として〜」等の形式で書く",
    ];
}
```

この`getEEATSignals()`の返り値は、Claude APIのシステムプロンプト構築時にそのまま注入される。

```php
// ClaudeArticleService.php - システムプロンプト構築

private function buildSystemPrompt(string $type, string $theme = 'camping'): string
{
    $profile = ThemeProfile\ThemeProfileFactory::get($theme);
    $themeAddendum = $profile->getSystemPromptAddendum();
    $eeatSignals = $profile->getEEATSignals();

    $base = "{$themeAddendum}
    // ...
    {$eeatSignals['instruction']}
    // ...
    ";
}
```

ポイントは「禁止パターン」と「推奨パターン」の**両方**を渡していること。「これを書くな」だけではLLMは代替表現を思いつけない。「代わりにこう書け」と具体的なテンプレートを示すことで、はじめてAIは「架空体験なしで説得力のある文章」を書けるようになる。

これを「Evidence-Based E-E-A-T」と呼んでいる。架空の体験ではなく、検証可能なエビデンスだけで記事を構成するルールだ。

さらに、生成後の`ContentPipelineValidator`でも禁止パターンのチェックを行い、すり抜けたものはフラグを立てる多段防御にしている。

**教訓：AIに「嘘をつくな」と言っても嘘をつく。「嘘がつけない仕組み」を作るしかない。**


## 失敗2: Twitter自動投稿が91件連続で失敗していた

SNS自動投稿の仕組みを意気揚々と実装した。記事が公開されたら、Claude APIで投稿文を生成し、Twitter API v2で自動ツイートする。完璧だ。

……と思っていた。

何日か経って、「あれ、Twitterのフォロワー全然増えないな」と管理画面を見たら、投稿ステータスが全部`failed`だった。91件。全滅。

原因は**文字数カウントのバグ**だった。

PHPの`mb_strlen()`で280文字制限をチェックしていたが、Twitter APIはCJK文字（日本語・中国語・韓国語）を**重み2**としてカウントする。つまりASCII文字は1だが、日本語の1文字は2としてカウントされる。これはTwitterの公式ドキュメント（[Counting characters](https://developer.x.com/en/docs/counting-characters)）に記載されているが、直感的ではない仕様だ。

```
// バグの時のコード（概念）
$len = mb_strlen($text);  // "こんにちは" → 5
if ($len <= 280) { ... }   // 5 <= 280 → OK!

// Twitter API側のカウント
// "こんにちは" → 各文字が重み2 → 10
// 日本語253文字 → 重み506 → 280超え → 拒否
```

`mb_strlen()`で「253文字だからOK」と判定していた投稿が、Twitterの重み計算では「427文字」になっていた。しかもTwitter APIはエラーレスポンスを返してくれるのだが、エラーハンドリングがログに書くだけだったので、何日も気づかなかった。

### どう解決したか

`twitterWeightedLength()`というメソッドを実装した。Unicode文字のコードポイントを見て、U+1100以上（CJK文字、全角記号、絵文字等）は重み2、それ以外は重み1としてカウントする。

```php
// SocialMediaService.php

/**
 * Twitter weighted文字数を計算
 * CJK文字（U+1100以上）は2、それ以外は1としてカウント
 */
private function twitterWeightedLength(string $text): int
{
    $weight = 0;
    $chars = mb_str_split($text);
    foreach ($chars as $c) {
        $ord = mb_ord($c);
        // CJK統合漢字、ひらがな、カタカナ、全角記号、絵文字等
        $weight += ($ord >= 0x1100) ? 2 : 1;
    }
    return $weight;
}

/**
 * Twitter weight制限に収まるようにテキストを切り詰め
 */
private function truncateToTwitterWeight(string $text, int $maxWeight): string
{
    $weight = 0;
    $chars = mb_str_split($text);
    $result = '';
    foreach ($chars as $c) {
        $ord = mb_ord($c);
        $charWeight = ($ord >= 0x1100) ? 2 : 1;
        if ($weight + $charWeight > $maxWeight) break;
        $weight += $charWeight;
        $result .= $c;
    }
    return $result;
}
```

投稿組み立て部分では、Twitter APIのt.co短縮URLの重み（常に23）も考慮して、正確な残りウェイトを計算している。

```php
// 投稿テキスト組み立て（280 weighted文字制限を考慮）
// Twitter APIはCJK文字を2、ASCII/絵文字を1としてカウント（t.co URLは常に23）
$urlWeight = 23;
$hashtagsPart = !empty($hashtags) ? "\n\n{$hashtags}" : '';
$hashtagsWeight = $this->twitterWeightedLength($hashtagsPart);
$newlineWeight = 2; // "\n\n" = 2
$maxContentWeight = 280 - $urlWeight - $hashtagsWeight - $newlineWeight;

// コンテンツをTwitter weight制限に収める
$contentWeight = $this->twitterWeightedLength($content);
if ($contentWeight > $maxContentWeight) {
    $content = $this->truncateToTwitterWeight($content, $maxContentWeight - 3) . '...';
}
```

U+1100の閾値は完全に正確ではない（Twitterの内部的にはもう少し細かいUnicodeレンジの区分がある）が、日本語コンテンツでは実用上これで十分機能している。もしハングルや特殊な絵文字の精密な処理が必要なら、[twitter-text](https://github.com/twitter/twitter-text)ライブラリのロジックを参考にすべきだ。

さらに、投稿失敗が連続したらSlack通知を飛ばす仕組みも追加した。

**教訓：「動いている」と思い込むのが一番怖い。監視は自動化の命綱。**


## 失敗3: 画像生成AIが突然全停止した

記事のアイキャッチ画像をGemini APIで自動生成していた。プロンプトから高品質な画像が生成され、記事に自動挿入される。素晴らしい仕組み——のはずだった。

ある日、記事生成バッチが画像ステップで全滅。エラーログを見ると「quota limit: 0」。有料プランなのにクォータがゼロ。

調べてみると、使っていたモデル名`gemini-3-pro-image-preview`のクォータ割り当てがゼロだった。Googleのドキュメントにはモデル名が載っているのに、実際にはクォータが割り当てられていないという罠。これはGemini APIの「preview」モデルではよくある問題で、ドキュメントに記載されているからといって使えるとは限らない。

「じゃあ正しいモデル名に変えよう」と`gemini-2.0-flash-exp-image-generation`に変更。すると今度は別のエラー。

```
aspectRatio parameter is not supported for this model
```

16:9のアスペクト比指定が、このモデルでは非対応だった。

### どう解決したか

モデルごとの対応パラメータを条件分岐で管理した。

```php
// GeminiImageService.php - callGeminiApi()

$generationConfig = [
    'responseModalities' => ['TEXT', 'IMAGE'],
    'temperature' => $temperature,
];

// gemini-2.0-flash-exp-image-generation はアスペクト比非対応
if (strpos($this->model, '2.0-flash-exp') === false) {
    $generationConfig['imageConfig'] = [
        'aspectRatio' => '16:9',
    ];
}
```

ここで`strpos($this->model, ...)`を使ってモデル名の部分一致で分岐している。最初はローカル変数`$model`を使って分岐を書いていたが、実際にはインスタンス変数`$this->model`に値が入っていたため、条件分岐が一切効かないというバグも埋め込んでいた。計4回のエラー連鎖で半日潰れた。

さらに、`.env`でモデル名を切り替えられる設計にしている。

```php
$this->model = $_ENV['IMAGE_MODEL'] ?? 'gemini-2.5-flash-image';
```

Gemini APIはモデルの追加・廃止・仕様変更が頻繁に発生する。コード内にモデル名をハードコードすると、変更のたびにデプロイが必要になる。`.env`で制御すれば、サーバー上で環境変数を書き換えるだけで対応できる。外部API依存のシステムでは「変更がコードデプロイなしで適用できるか」を常に考えるべきだ。

**教訓：外部APIは「動いている前提」で設計してはいけない。モデル更新、パラメータ変更、突然のクォータ変更。すべてが起こり得る。**


## 失敗4: 自動生成した記事が「薬機法違反」だった

AIは法律を「知っている」ようで「わかっていない」。

ワインの健康効果について書いた記事で、AIがこう書いた。

> 「ポリフェノールが豊富で、動脈硬化の予防に効果があります」

これは薬機法違反だ。食品であるワインに対して、病気の予防効果を断言することはできない。「効果がある」ではなく「～と言われている」「研究が報告されている」程度にとどめなければならない。

LLMが法令違反テキストを生成してしまう本質的な原因は、学習データに法令違反の表現が大量に含まれているからだ。ネット上のアフィリエイト記事には「〇〇に効果がある」と平気で書かれている。AIはそれを「正しい記事の書き方」として学習してしまう。プロンプトで「薬機法に注意」と書いても、学習データのバイアスには勝てない。

### どう解決したか

記事の自動公開パイプラインに「法的チェックゲート」を組み込んだ。

Claude APIに別のリクエストを送り、生成された記事を法的観点からレビューさせる。薬機法、景表法、ステマ規制の3つの観点でチェックし、100点満点でスコアリングする。

```php
// ClaudeArticleService.php - reviewArticle()

public function reviewArticle(string $content, string $title): array
{
    $systemPrompt = "あなたは日本の広告法規（薬機法・景表法・ステマ規制）の
専門レビュアーです。
記事コンテンツをレビューし、法的リスクのある表現を検出してください。

## チェック項目
1. **薬機法違反**: 医薬品でない商品への効能効果の断言
   （「効く」「治す」「改善される」等）
2. **景表法違反**: 根拠のない優良表示
   （「No.1」「最高」「100%満足」等の絶対表現）
3. **根拠なき主張**: 出典なしの統計データ、科学的根拠のない効果の主張
4. **免責表記不足**: 「個人の感想です」「参考価格」等の注記がない

## 出力形式（JSON）
{
  \"issues\": [...],
  \"score\": 85,
  \"fixed_content\": \"修正版のHTML全文（error級のみ修正）\"
}";

    // ... Claude APIにレビューリクエストを送信 ...
}
```

記事生成とレビューで**同じAI（Claude API）を使っている**ことに疑問を持つかもしれない。「生成したAIが自分の出力をチェックして意味があるのか？」と。実際に意味がある。同じモデルでも、「記事を書く」というタスクと「法的観点からレビューする」というタスクでは、活性化されるコンテキストが異なる。レビュータスクではシステムプロンプトで法的チェック項目を明示的に与えるため、生成時に見落とした問題を高確率で検出できる。

自動公開バッチでは、このスコアによって公開可否を判定する。

```php
// auto-generate-articles.php

$reviewResult = $articleService->reviewArticle($contentForReview, $titleForReview);
$reviewScore = $reviewResult['score'] ?? -1;

// 法的スコアが85未満の場合はdraftのまま保持し、人間レビューを要求
if ($reviewScore >= 0 && $reviewScore < 85) {
    echo "  [法的ゲート] 法的スコア {$reviewScore} < 85 のためdraftのまま保持します\n";

    // Slack通知で人間にレビューを要求
    $slackService->sendAlert(
        '法的レビュー要求',
        "記事ID: {$articleId}\nタイトル: {$title}\n法的スコア: {$reviewScore}/100\n\n"
        . "法的スコアが85未満のため自動公開を保留しました。",
        'warning'
    );
}
```

- **85点以上** → 自動公開OK
- **84点以下** → 下書き保存、人間レビュー送り（Slack通知付き）

実際の運用では、法的チェック通過率は約91%。つまり10記事中9記事は完全自動で公開され、1記事だけ人間がチェックする。

このバランスが重要だ。「人間を完全に排除する」のが自動化の目的ではない。**「人間が見るべき箇所を最小化する」のが正しい自動化**だ。

10記事すべてを人間がチェックするのは非現実的だが、フラグ付きの1記事だけならできる。

**教訓：「人間を排除する」のではなく「人間が見るべき箇所を最小化する」のが正しい自動化。**


## 失敗5: AIが生成したHTMLでサイトが崩壊した

Claude APIに記事を生成させると、JSON形式でレスポンスが返ってくる。そのなかにHTML本文が含まれている。

問題は、そのHTMLが壊れていることがあること。

よくあるパターン：

- `<a href="https://example.com/very-long-url-that-gets-trun`（URLが途中で切れる）
- 閉じクォートの忘れ：`<a href="https://example.com>テキスト</a>`
- `</a>`タグの欠落
- JSONの構文がHTMLのなかに混入：`"content": "<h2>見出し</h2>...`

なぜこういうことが起きるかというと、LLMのトークン生成には`max_tokens`の上限があるからだ。Claude APIの場合、レスポンスがmax_tokensに達すると途中で切断される。HTMLタグの途中であっても容赦なく切れる。さらにJSON内にHTMLを埋め込む構造のため、エスケープ処理の不整合やクォートの対応ミスも発生しやすい。

最悪のケースでは、閉じクォートが欠落した`<a>`タグのせいで、後続のHTML全体が1つのリンクの属性値として解釈され、サイト表示が完全に崩壊した。ページを開いたら、ヘッダーからフッターまで全部が1つの巨大な青いリンクになっていた。

### どう解決したか

「AIの出力を信用しない」を大前提にして、6段階のバリデーションパイプラインを構築した。

最初の防衛ラインは`sanitizeContent()`だ。AIが返してくるゴミを機械的に掃除する。

```php
// ContentPipelineValidator.php

public static function sanitize(string $content): string
{
    // 1. コードフェンス除去
    $content = self::removeCodeFences($content);

    // 2. JSON wrapper解除
    $content = self::unwrapJson($content);

    // 3. \n → 実改行、\" → " 変換
    $content = self::fixEscapedCharacters($content);

    // 4. 末尾切断タグ除去
    $content = self::fixTruncatedTags($content);

    // 5. HTMLタグバランス修正
    $content = self::balanceHtmlTags($content);

    return $content;
}
```

各メソッドの中身を見てみよう。`unwrapJson()`はJSON wrapperを解除してHTMLを取り出す。

```php
private static function unwrapJson(string $content): string
{
    $trimmed = trim($content);

    // パターン: {"content": "...", ...} 形式
    if (preg_match('/^\s*\{/', $trimmed) && preg_match('/\}\s*$/', $trimmed)) {
        $decoded = json_decode($trimmed, true);
        if ($decoded && isset($decoded['content'])) {
            return $decoded['content'];
        }

        // json_decodeが失敗しても、"content": の値を手動抽出
        if (preg_match('/"content"\s*:\s*"((?:[^"\\\\]|\\\\.)*)"/s', $trimmed, $m)) {
            $extracted = $m[1];
            // JSONエスケープを戻す
            $extracted = str_replace(
                ['\\n', '\\"', '\\\\', '\\t', '\\/'],
                ["\n", '"', '\\', "\t", '/'],
                $extracted
            );
            if (strlen($extracted) > 100) {
                return $extracted;
            }
        }
    }

    return $content;
}
```

`json_decode()`が失敗するケースがある（HTMLの中に制御文字が含まれている場合など）ため、正規表現による手動抽出をフォールバックとして用意している。

`balanceHtmlTags()`は閉じ忘れたタグを自動修復する。

```php
private static function balanceHtmlTags(string $content): string
{
    $openTags = [];
    $selfClosing = ['img', 'br', 'hr', 'input', 'meta', 'link',
                    'source', 'area', 'col', 'embed', 'wbr'];

    // タグを順に追跡
    preg_match_all('/<\/?([a-zA-Z][a-zA-Z0-9]*)[^>]*>/s',
                   $content, $matches, PREG_SET_ORDER);

    foreach ($matches as $match) {
        $fullTag = $match[0];
        $tagName = strtolower($match[1]);

        if (in_array($tagName, $selfClosing)) continue;

        if (str_starts_with($fullTag, '</')) {
            // 閉じタグ → openTagsから除去
            $lastKey = array_search($tagName, array_reverse($openTags, true));
            if ($lastKey !== false) {
                unset($openTags[$lastKey]);
                $openTags = array_values($openTags);
            }
        } elseif (!str_ends_with(rtrim($fullTag, '>'), '/')) {
            // 開始タグ → 追跡
            $openTags[] = $tagName;
        }
    }

    // 閉じられていないタグを逆順に閉じる
    foreach (array_reverse($openTags) as $tag) {
        $content .= "</{$tag}>";
    }

    return $content;
}
```

この後に`validate()`がDB書き込み前の最終ゲートとして控えている。

```php
public static function validate(string $content, string $articleType = 'other'): array
{
    $errors = [];

    // 1. JSON wrapper検出
    // 2. \nリテラル検出
    // 3. 末尾タグ切断検出
    // 4. 最低文字数チェック（knowledgeタイプは2500文字、他は500文字）
    // 5. H2存在チェック
    // 6. 禁止タグチェック（html, head, body, script）
    // 7. E-E-A-T信号チェック

    return ['valid' => empty($errors), 'errors' => $errors];
}
```

sanitize → validate → 品質チェック → 法的チェック → 画像生成 → 公開後週次巡回。6段階のパイプラインを通すことで、「サイトが崩壊する」レベルの問題はほぼ防げている。

完璧ではないが、「AIの出力をそのままDBに入れる」設計からの進歩は大きい。

**教訓：AIの出力を信用してはいけない。「信用しないシステム」を作る。**


## 5つの失敗から学んだこと

ここまで読んで気づいたかもしれないが、5つの失敗に共通するパターンがある。

**「AIに任せた部分」ではなく「AIと現実世界の接続部分」でコケている。**

- AIの出力 x Googleの評価基準（E-E-A-T偽装）
- AIの出力 x Twitter APIの文字カウント仕様
- AIの出力 x Gemini APIのモデル仕様変更
- AIの出力 x 日本の法規制
- AIの出力 x ブラウザのHTML解釈

AI単体は優秀だ。でもAIを現実世界のシステムに組み込んだ瞬間、「AIが知らないルール」との衝突が始まる。

自動化すればするほど、この「接続部分」の設計が重要になる。そしてその設計には、AIではなく人間のエンジニアリングが必要だ。

AI完全自動化は「AIに丸投げ」ではない。**「AIを信用しないシステムを設計する」こと**だ。

現在のシステムは安定稼働している。100記事以上、700枚以上の画像、19本のcronジョブ。毎週6記事が完全自動で生成・公開されている。でもそれは「AIがすごいから」ではなく、「AIがコケたときの受け皿」を何十個も積み上げた結果だ。

次回は、このシステムで自動生成した記事がGoogleからどう評価されているのか、Search Consoleの実データとともに公開予定。AIが書いた記事はインデックスされるのか？ 検索順位はつくのか？

---

この仕組みの裏側——日々のトラブルや改善を毎日つぶやいています。
AI自動化の「きれいじゃない現実」に興味がある方はぜひ。

**X（Twitter）: [@smartchoicelab](https://twitter.com/smartchoicelab)**

Zennでのフォロー・いいねもお待ちしています。質問やスクラップでの議論も歓迎です。
