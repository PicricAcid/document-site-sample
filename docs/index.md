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
