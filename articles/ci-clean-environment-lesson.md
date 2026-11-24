---
title: "「自分の環境で通る」と「まっさらな環境でも再現する」は別物。SwitchBotサーキュレーターとCIの教訓"
emoji: "🧹"
type: "tech"
topics: ["ci", "githubactions", "テスト", "switchbot"]
published: true
published_at: 2026-10-29 12:00
---

この記事は、個人ブログに掲載した記事の転載です。

https://freddiefujiwara.com/blog/1UJziyYzHPuIZGVTSNUL2AXP0_oakUNmAhIUhNm0WHbM

楽天市場で買った SwitchBot のスマートサーキュレーターが届いたので、自宅のシステムに早速取り込みました。

セットアップに失敗したので調べてみると、GitHub Actions のテストが思いっきりコケていました。

久々に触るコードってこういうことがありますよね。

ユニットテストに助けられました、、

教訓は、「自分の環境でテストが通る」と「まっさらな環境でも再現する」は別物だということです。ユニットテストとCIがあったから、原因の特定も修正の確認も早く済みました。本当に大事です！

買ったのはこちらです（楽天ROOM）

https://room.rakuten.co.jp/freddiefujiwara/1700398325097907
