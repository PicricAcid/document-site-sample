---
title: GitHubによるドキュメントサイト管理システムの構築　Part.2
author: PicricAcid
date: 2025-09-23
lastmod: 2025-09-23
tags: [github, github_pages]
---

GitHub Pagesで作るMarkdownドキュメントサイトシステムの第2回です。
第1回の続きとなっていますので、よろしければ第一回もご覧ください。

前回の記事ではリポジトリを作成し、それをサイトとして公開する手順と、Markdownで記載した記事のコードブロックや数式の修飾を行いました。
今回の記事ではサイトの構造化と各種機能の追加、ブランチ管理について述べます。
ソースコードは以下の[GitHub](https://github.com/PicricAcid/document-site-sample)で閲覧できます。

## ドキュメントサイトの構造化
ドキュメントサイトの使い勝手を上げるため、エントリページを起点として各記事に飛べるように構造化していきましょう。

リポジトリを以下のような構造にしていきます。

```
/-
 |- docs/
 |    |- index.md 　　　　　　　　　　(エントリページ)
 |    |- contents/　　　　　　　　　　(各記事を格納)
 |    |- _layouts/
 |    |     |- default.html　　　　 (ページの基本構造を記載(前回実装))
 |    |
 |    |- assets/
 |          |- style.css           (ページの基本デザインを記載(前回実装))
 |          |- rouge.css           (コードブロックのデザインを記載(前回実装))
 |
 |- _config.yml　　　　　　　　　　　　(サイトの設定を記載(前回実装))
```

まず、index.mdをエントリページに改良しましょう。コードブロックや数式は削除して以下のようにします。

```markdown
---
title: "Document Site"
layout: default
---

これはGitHub Pagesを使ったドキュメントサイトのサンプルです。 

---

## 🗒️ 記事一覧

<ul>
	{% assign pages = site.pages | where_exp: "p", "p.path contains 'contents/'" %}
	{% for p in pages %}
		<li>
			<a href="{{ site.baseurl }}{{ p.url }}">{{ p.title }}</a>
		</li>
	{% endfor %}
</ul>

---

```

`\{% \%}`や`{{ }}`で囲っている箇所は、GitHub Pages上で動いているサイト変換ツール(Jekyll)で使うことのできるLiquidというテンプレート記法によるものです。`{{ }}`で囲った部分は内容をHTMLに埋め込むことができ、`\{% \%}`で囲った部分ではforやifなどの制御文を使うことができます。
ここでは、記事を置いている`docs/contents/`から記事をすべて取得し、リンクとしてHTMLに埋め込んでいます。

次に`docs/contents/`に実際にMarkdownドキュメントを格納していきます。
ここでは確認のため、前回の記事(リンク)をMarkdownにしたものを置きます。
Markdownを置く際は、先頭にFrontmatterとして以下を追加します。

```markdown
---
title: [記事タイトル]
---
```

それではcommit, deployしてみましょう。
[fig_1]

`contents/`下の記事が取得、表示できていますね。リンクになっているのでリンクを踏んでみると
[fig_2]

Markdownがサイトページに変換されています。前回のdefault.htmlの設定により、Frontmatterのtitle欄に入力したタイトルが記事の先頭に出るようになっています。

## 目次機能、コメント機能の追加
サイトに機能を追加していきましょう。
### 目次機能
まず、記事内の各章に飛べる目次機能を作りましょう。
実装方法としては、サイトを作る前にMarkdown内のh2, h3要素(`##`や`###`から始まる)を取得し、リンクをHTMLに埋め込むようにします。

`docs/contents/assets/toc.js`として以下を追加します。

```javascript
document.addEventListener("DOMContentLoaded", function () {
	const tocContainer = document.querySelector("#toc");
	const content = document.querySelector("main") || document.body;
	const headings = content.querySelectorAll("h2, h3");

	if (!tocContainer || headings.length === 0) {
		if (tocContainer) tocContainer.style.display = "none";
		return;
	}

	const toc = document.createElement("nav");
	toc.innerHTML = "<h2>目次</h2><ul></ul>";
	const ul = toc.querySelector("ul");

	headings.forEach(h => {
		if (!h.id) {
			h.id = h.textContent.trim().toLowerCase().replace(/\s+/g, "-");
		}
	
		const li = document.createElement("li");
		li.style.marginLeft = h.tagName === "H3" ? "1em" : "0";
		li.innerHTML = `<a href="#${h.id}">${h.textContent}</a>`;
		ul.appendChild(li);
	});

	tocContainer.appendChild(toc);
});
```

これを`default.html`に埋め込みます。

```diff
			{{ content }}
		</main>

+		<aside class="page-sidebar">
+			<div class="page-toc" id="toc"></div>
+		</aside>
	</div>

	<footer>
		<p>©︎ {{ site.time | date: "%Y" }} {{ site.title }}</p>
	</footer>
	
+	<script src="{{ site.baseurl }}/assets/toc.js"></script>
</body>
```

さらに、`style.css`に装飾を記述します。

```diff
+.page-sidebar {
+	flex: 1;
+	min-width: 200px;
+	max-width: 300px;
+	position: sticky;
+	top: 1rem;
+	display: flex;
+	flex-direction: column;
+	gap: 1rem;
+	height: fit-content;
+}

+.page-toc {
+	background: #f9f9f9;
+	padding: 1rem;
+	border: 1px solid #ddd;
+	border-radius: 6px;
+	font-size: 0.9em;
+}
```

`position: sticky;`にすることでスクロールに追従するようにしています。

### コメント機能
記事の投稿者に対して何かフィードバックできる機能があると便利です。QiitaやZennのようなコメント機能は実装するとなると動的な処理が必要になり難しいです。そこで、記事に関するGitHubのIssueを建てられるボタンを作ります。

`default.html`に以下を追加します。

```diff
<aside class="page-sidebar">
	<div class="page-toc" id="toc"></div>

+	<div class="feedback">
+		<a class="feedback-button" 
+			href="https://github.com/[リポジトリのURL]/issues/new?title={{ page.title | uri_escape }} のフィードバック&body=このページに関するフィードバックを入力してください: {{ site.url }}{{ page.url }}%0A%0A@{{ site.data.authors[page.author].github | page.author }}"
+			target="_blank">
+			📝 フィードバックを送る
+		</a>
+	</div>
</aside>
```

Issue生成のリンクを踏ませているだけですね。
url後半で`{{ site.data.authors[page.author].github | page.author }}`としている箇所があります。これは作成したIssueを投稿者にメンションするためのものです。
これを実現するには記事ごとに`page.author`を設定する必要がありますので、記事のFrontmatterに著者を追加します。

```markdown
---
title: [記事タイトル]
author: [著者]
---
```

また、記事の著者名とメンションに必要なGitHubアカウント名が一致しない場合も考えられます。そこで著者名とGitHubアカウント名の関係を`docs/_data/authors.yml`に記述します。

```yaml
[著者名]:
	github: [GitHubアカウント名]

# e.g.
PicricAcid:
	github: PicricAcid
```

これを`site.data.authors[page.author].github`として読み込んでいます。

目次機能、コメント機能を実装し、画面全体のcssを整えるとサイトは以下のようになります。
ちょっとずつ使えるものになってきました。
[fig_3]

また、サイト中の「フィードバックを送る」を押すとIssueが建ちます。
[fig_4]

## タグ機能の追加
今はまだ記事が少ないですが、記事が増えてくるとキーワードで検索したり、類似記事を読みたい、なんてことも出てきます。そこで、タグ機能を作りましょう。
投稿者が記事に数個のタグをつけることで、閲覧者は同一タグがついた記事の一覧を見ることができるようになります。

`default.html`に以下を追加します(ついでに著者名の表示も追加しています)。

```diff
<div class="wrapper page">
	<main class="page-content">
	<h1 class="page-title">{{ page.title }}</h1>
+	<p class="page-meta">
+		{% if page.author %}🖋 {{ page.author }} {% endif %}
+	</p>
	
+	{% if page.tags %}
+		<p class="page-tags">
+		🏷 :
+			{% for tag in page.tags %}
+				<a class="tag-link" href="{{ site.baseurl }}/tags/{{ tag | downcase | uri_escape }}">[{{ tag }}]</a>
+			{% endfor %}
+		</p>
+	{% endif %}
  
	{{ content }}
</main>
```

`page.tags`に設定されたタグをページに埋め込むようになっています。つまり、記事のFrontmatterにtags情報が必要です。

```markdown
---
title: [記事タイトル]
author: [著者]
tags: [[タグ1], [タグ2]]
---
```

また、タグ別記事一覧(タグページ)を作成します。
`docs/tags/`ディレクトリを作成し、タグページを配置します。タグページは以下のようにFrontmatterのみ記述し、一覧情報などは自動生成します。

```markdown
---
layout: tag
tag: [タグ名]
title: [タグページタイトル]
---
```

タグページ用のHTMLを`_layouts/tag.html`として作成します。

```html
---
layout: default
---

{% assign count = 0 %}
{% for p in site.pages %}
	{% if p.tags and p.tags contains page.tag %}
	{% assign count = count | plus: 1 %}
	{% endif %}
{% endfor %}

<h1>{{ page.title }} ({{ count }})</h1>

<ul>
	{% for p in site.pages %}
		{% if p.tags and p.tags contains page.tag %}
			<li>
				<a href="{{ site.baseurl }}{{ p.url }}">{{ p.title }}</a>
			</li>
		{% endif %}
	{% endfor %}
</ul>
```

commit, deployしてみると以下のようにタイトルの下にタグが表示されます。
[fig_5]

また、タグをクリックするとタグページに飛ぶことができます。
[fig_6]

## ブランチ制御
記事の機能が整ってきたところで、拡張は一旦止め、ブランチ制御について考えてみましょう。
現状、サイトを生成しているリポジトリはmainブランチのみで、mainブランチにpushするとすぐにサイトのbuild, deployが始まってしまいます。本番環境にそのままpushしていることになるので健全ではありません。
そこで、開発用ブランチとしてdevブランチを作成し、dev -> mainへはPRを必須としてmainブランチを保護します。

### devブランチの作成
GitHubでブランチを作成する方法は説明するまでもないですが、codeページ左上のブランチのプルダウンから「Find or create branch...」に作りたいブランチ名を入力し、Createを押します。
[fig_7]

### mainブランチの保護
devブランチで作業するとし、mainブランチをPRによるmerge以外に編集できないよう保護します。
ブランチ保護の設定は「Setting」-「Branches」タブにある「Branch protection rules」から行います。
「Add branch rule set」と「Add classic branch protection rule」があるのですが、今回は「Add classic branch protection rule」から作ります。

>[!Tip]
>Add branch rule setのほうが新しい設定なのですが、classicの方が情報が多く、新しい方とどう対応しているかいまいちわからないため、classicの方を選択しています。

「Branch name pattern」に「main」と入力し、「Require a pull request before merging」、「Require review from Code Owners」にチェックを入れ、「Create」ボタンを押します。
[fig_8]

>[!NOTE]
>複数人で管理する場合、さらに「Require approvals」を設定してしておくと良いでしょう。決められた人数からのコードレビューがないとPRを承認できなくなる設定です。

これでmainは直接編集できなくなりました(直接編集すると使い捨ての別ブランチの変更になります)。

## まとめ

今回は
- サイトの構造化
- 目次機能
- コメント機能
- タグ機能
- mainブランチ保護
を行いました。これでかなり使えるサイトになったかと思います。

次回は構造化で煩雑になった記事投稿手順を自動化するVSCode拡張を作成します。

## 参考文献
- [保護されたブランチについて](https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
