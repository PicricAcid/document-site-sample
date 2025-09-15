---
title: GitHubによるドキュメントサイト管理システムの構築 Part.1
author: PicricAcid
tags: [GitHub, GitHubPages]
---

GitHub Pagesを用いてMarkdownドキュメントを静的サイトにして公開するシステムを構築してみた記事です。
サイト公開のしくみだけでなく、VSCodeによる執筆支援、MCPサーバによるAIとの連携についても記述するつもりです。
社内でのマニュアル管理などに用いることを想定したシステムになっています。
この記事で実装したコードは以下の[GitHub](https://github.com/PicricAcid/document-site-sample)に載せています。

## GitHub Pagesで作る理由
今回はGitHub Pagesでサイトを作成しています。理由としては以下です。

- Gitによるドキュメントのバージョン管理ができること。
	とくに、今回の想定では社内の複数の執筆者が同時に手を入れることが考えられますので、バージョン管理の恩恵は大きくなります。
	今回のシステムではmainをデプロイブランチとして保護しておいて、devブランチからのMergeは管理者によるレビュー、PRを必須として安定性を担保します。

- GitHubさえ使えれば追加ツールのインストールは基本的に不要であること。
	たとえば、GitLab + Hugoでも同様なことができますが、サーバの用意やHugoのインストールなど煩雑です。GitHub Pagesであれば、社内でGitHubが使える(かつ社内情報をアップロードできる)状況であればサーバや他のツールは不要です。強いて言えば、記事執筆に用いるVSCodeが必要なくらいです。

- Markdownでドキュメントを執筆できること。
	Github Pagesには標準でJekyllというMarkdownをサイトに変換するライブラリが付いているため、Markdownをpushすると自動でサイトにできます。
	さらに、MarkdownドキュメントであればAIに読み込ませて活用することができます。今回この記事では、ドキュメントサイトの内容をMCPサーバでGitHub Copilotに渡すことで検索、要約をさせる機構も作成します。

- ドキュメントの公開範囲を制限できること。
	社内ドキュメントをサイト化する場合、気になるのは社内情報が外に出てしまわないか、ということです。
	GitHub Enterprise Cloud(を契約している会社内で使用することを想定)ではリポジトリをPrivateにして公開範囲を制限すると、そこから生成されたサイトもリポジトリの公開範囲に従うため、社内情報が外に出るようなことはありません。

## GitHub Pagesによるサイト生成
では早速サイト生成をしてみましょう。
ただサイトを作るだけであればかなり簡単です。

- 1. GitHubにリポジトリを作成する
	GitHubリポジトリを作成します。この時、サイトを外部非公開にするにはPrivateリポジトリを作成する必要があります。

- 2. 公開のエントリポイントを作成
	エントリポイントとしてリポジトリ直下にdocs/ディレクトリを作成します。
	そして「Setting」-「Pages」-「Build and deployment」-「branch」を「main」, 「/docs」に指定します。
	

- 3. 設定ymlファイルを作成
	リポジトリ直下に_config.ymlとして以下を作成します。

```yaml
permalink: pretty
markdown: kramdown
```

- 4. 記事の作成
	docs/下に記事を作成します。一旦index.mdとしてみましょう。

```markdown
---
title: "Document Site"
---

# ドキュメントサイト

これはGitHub Pagesを使ったドキュメントサイトのサンプルです。 

```

先頭の`---`で囲まれた部分はFront Matterといい、記事のメタ情報を記載しています。後で記事を管理する際に用います。

現時点で、リポジトリの構造は以下のようになっています。

```
/
|- docs/
|	|- index.md
|	|- contents/    (後で拡張)
|
|- _config.yml	
```

- 5. コミット、デプロイ
	作成した記事をmainブランチでcommitすると、自動でサイト生成が始まります。
	[fig_2]

	デプロイで生成されたurlを開いてみましょう。以下のようなサイトが表示されます。
	[fig_3]


## サイトの整形
サイトはできましたが、殺風景なので少し体裁を整えましょう。
また、技術ドキュメントではコードブロック、数式を表示する場合が多いのでそれらの表示設定もします。

- default.htmlの追加
	体裁を整えるためのhtmlファイルを作ります。docs/_layoutsディレクトリを作成し、
	default.htmlとして以下を記載します。

```html
<!DOCTYPE html>
<html lang="ja">

<head>
  <meta charaset="UTF-8">
  <title>{{ page.title }} | {{ site.title }}</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="{{ site.baseurl }}/assets/style.css">
</head>

<body>
  <header class="site-header">
    <div class="wrapper">
      <h1 class="site-title"><a href="{{ site.baseurl }}/">{{ site.title }}</a></h1>
      <nav class="site-nav">
        <a href="https://github.com/{{ site.github.repository_nwo }}">GitHub</a>
      </nav>
    </div>
  </header>

  <div class="wrapper page">
    <main class="page-content">
      <h1 class="page-title">{{ page.title }}</h1>
      {{ content }}
    </main>
  </div>

  <footer>
    <p>©︎ {{ site.time | date: "%Y" }} {{ site.title }}</p>
  </footer>
</body>

</html>
```


これをサイトに適用するには、index.mdのFront Matterに以下を追加します。

```diff
---
title: "Document Site"
+ layout: default
---

# ドキュメントサイト
```


- _config.ymlへの追記
	コードブロックの適用のために_config.ymlに以下を追記します。
	ここでは、rougeというシンタックスハイライト適用ツールを使用しています。

```diff
permalink: pretty
markdown: kramdown
+ highlighter: rouge

+ kramdown:
+  input: GFM
+  syntax_highlighter: rouge
```

- cssの適用
	docs/assets/ディレクトリを作成し、style.cssとして以下を追加します。

```css
body {
	font-family: system-ui,-apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial, sans-serif;
	line-height: 1.6;
	margin:0;
	color:#333;
}
main {
	max-width: 960px;
	margin: 2rem auto;
	padding: 0 1rem;
}

pre {
  background: transparent;
  border: 0;
  border-radius: 0;
  padding: 0;
  margin-bottom: 1.5em;
  overflow-x: auto;
  font-size: 0.9em;
  line-height: 1.5;
}

.highlight, figure.highlight, pre.highlight {
	background: #f6f8fa;
	border: 1px solid #d0d7de;
	border-radius: 6px;
	overflow: auto;
	position: relative;
	margin: 1.25rem 0;
}

.highlight pre, figure.highlight pre {
	background: transparent;
	border: 0;
	margin: 0;
	padding: 1rem;
	font-size: 0.9em;
	line-height: 1.5;
}

.highlight pre code {
	background: transparent;
	border: 0;
	padding: 0;
}

p code, li code {
	background: rgba(0,0,0,.04);
	padding: .15em .35em;
	border-radius: 4px;
	border: 1px solid rgba(0,0,0,.06);
}

.copy-button {
	position: absolute;
	top: .5rem;
	right: .5rem;
	font-size: .8em;
	padding: .25em .6em;
	border:1px solid #cfd2d3;
	border-radius: 4px;
	background: #fff;
	cursor: pointer;
	opacity: .85;
}

.copy-button:hover {
	opacity: 1;
}
```

さらに、コードブロックのシンタックスハイライト用にassets/下にrouge.cssを作ります。

```css
.highlight {
  color: #24292e;
  background-color: #f6f8fa;
}

.highlight .c,
.highlight .cm,
.highlight .c1,
.highlight .cs {
  color: #6a737d; font-style: italic; /* コメント */
}

.highlight .err { color: #d73a49; } /* エラー */
.highlight .k,
.highlight .kd,
.highlight .kn,
.highlight .kp,
.highlight .kr,
.highlight .kt { color: #d73a49; } /* キーワード */

.highlight .o { color: #d73a49; }   /* 演算子 */
.highlight .gd { color: #b31d28; background-color:#ffeef0; }
.highlight .gi { color: #22863a; background-color:#f0fff4; }
.highlight .gu { color: #6f42c1; }

.highlight .m,
.highlight .mb,
.highlight .mf,
.highlight .mh,
.highlight .mi,
.highlight .il {
  color: #005cc5; /* 数値 */
}

.highlight .s,
.highlight .sb,
.highlight .sc,
.highlight .sd,
.highlight .s2,
.highlight .se,
.highlight .sh,
.highlight .si,
.highlight .sr,
.highlight .ss,
.highlight .sx {
  color: #032f62; /* 文字列 */
}

.highlight .na,
.highlight .nc,
.highlight .nd,
.highlight .ne,
.highlight .nf,
.highlight .ni,
.highlight .nn,
.highlight .nt {
  color: #6f42c1; /* 名前（関数/クラスなど） */
}

.highlight .no,
.highlight .nv,
.highlight .nb {
  color: #005cc5; /* 定数・変数・組み込み */
}

.highlight .w { color: #bbbbbb; } /* 空白 */

```

- 数式の適用
	数式の適用にはMathJaxを使用します。default.htmlのheadに以下を追加します。

```diff
<head>
  <meta charset="UTF-8">
  <title>{{ page.title }} | {{ site.title }}</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="{{ site.baseurl }}/assets/style.css">
  <link rel="stylesheet" href="{{ site.baseurl }}/assets/rouge.css">
+ <script>
+ window.MathJax = {
+      tex: {
+        inlineMath: [['$', '$'], ['\\(', '\\)']],
+        displayMath: [['$$', '$$'], ['\\[', '\\]']],
+        processEscapes: true
+      },
+      options: {
+        skipHtmlTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code'] 
+      }
+    };
+  </script>
+  <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js"></script>

</head>

```

- index.mdの追記
	最後にテストも兼ねてindex.mdに追記します。

```markdown
---
title: "Document Site"
layout: default
---

これはGitHub Pagesを使ったドキュメントサイトのサンプルです。 

---

## コードブロック

コードブロックのサンプルです。

\`\`\`c
#include <stdio.h>
int main(void){
  printf("Hello Manual Site!\n");
  return 0;
}
\`\`\`

## 数式

数式のサンプルです。

$$
e = \sum_{n=0}^{\infty} \frac{1}{n!}
$$
```

>[!NOTE]
>コードブロックの「```」が誤認識されてしまうのでここでは都度「\」を挿入していますが、実際には不要です。


ここまでやると以下のようにとりあえずの体裁は整います。


## まとめ、次回予告
この記事ではGitHub Pagesでドキュメントサイトを作るメリットとその取り掛かりについて述べました。これだけでも大抵のmarkdownは様になるようになりますが、まだ手を入れていきたいところです。

次回では、このサイトに目次機能、タグ機能、コメント機能、全文検索機能を実装し実用に耐えうるものにしていきます。
また、複数人で編集する場合を考えてブランチ管理もしていきたいと思います。

## 参考文献
- [Jekyll を使用して GitHub Pages サイトを作成する](https://docs.github.com/ja/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll)
- [GitHub PagesとJekyllでMarkdownを静的サイト公開](https://zenn.dev/sasakiki/articles/e4d5dd28700b16)
