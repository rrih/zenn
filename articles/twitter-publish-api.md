---
title: "Twitterの埋め込みツイート取得API叩いてみる"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["twitter", "next.js"]
published: false # 公開設定
---

## Twitterのツイート埋め込みの仕様について

https://publish.twitter.com

endpoint  
https://publish.twitter.com/oembed?url={url}&partner=&hide_thread=false

![](https://storage.googleapis.com/zenn-user-upload/7bd2eeafa6a1-20230615.png)

公開されていて、特に制限がなさそう（？）  
→叩いてみる

```bash
% curl "https://publish.twitter.com/oembed?url=https%3A%2F%2Ftwitter.com%2FTwitterJP%2Fstatus%2F1587301348604841987&partner=&hide_thread=false" | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1181  100  1181    0     0   7249      0 --:--:-- --:--:-- --:--:--  7474
{
  "url": "https://twitter.com/TwitterJP/status/1587301348604841987",
  "author_name": "Twitter Japan",
  "author_url": "https://twitter.com/TwitterJP",
  "html": "<blockquote class=\"twitter-tweet\"><p lang=\"ja\" dir=\"ltr\">ジブリさん (<a href=\"https://twitter.com/JP_GHIBLI?ref_src=twsrc%5Etfw\">@JP_GHIBLI</a>) に書き下ろしていただいたヘッダーを、ジブリパークのオープンを記念して、みなさんにも使っていただけることになりました🤩<br><br>常識の範囲内でつかって下さい。 <a href=\"https://t.co/HQ94KU3YAI\">pic.twitter.com/HQ94KU3YAI</a></p>&mdash; Twitter Japan (@TwitterJP) <a href=\"https://twitter.com/TwitterJP/status/1587301348604841987?ref_src=twsrc%5Etfw\">November 1, 2022</a></blockquote>\n<script async src=\"https://platform.twitter.com/widgets.js\" charset=\"utf-8\"></script>\n",
  "width": 550,
  "height": null,
  "type": "rich",
  "cache_age": "3153600000",
  "provider_name": "Twitter",
  "provider_url": "https://twitter.com",
  "version": "1.0"
}
```

- input(query param)
    - url
        - TweetのURLをエンコードした値
- output(json)
    - author_name
    - author_url
    - html
        - これを使ってみる
    - cache_age
    - height
    - width
    - provider_name
    - provider_url
    - type
    - url
    - version

## Next.jsのページに組み込んでHTMLを取得して表示までやる

src/pages/test/index.tsx
```typescript
import { useState } from "react";
import parse from "html-react-parser";

const TestIndex = () => {
  const [url, setUrl] = useState("");
  const [tweetHTML, setTweetHTML] = useState("");

  const handleURLChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setUrl(e.target.value);
  }

  const submit = async (url: string) => {
    const res = await fetch("/api/test", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ url: url }),
    });

    setTweetHTML((await res.json()).response.html);
    // ツイート表示用のスクリプトを追加
    const head = document.getElementsByTagName("head")[0] as HTMLElement;
    const script = document.createElement("script");
    script.src = "https://platform.twitter.com/widgets.js";
    script.async = true;
    script.type = "text/javascript";
    head.appendChild(script);
  }

  return (
    <>
      <h2>Tweet 埋め込みHTML 取得ページ</h2>
      <input type="text" value={url} onChange={handleURLChange} />
      <button onClick={() => submit(url)}>submit</button>
      <div>{parse(tweetHTML)}</div>
    </>
  );
};

export default TestIndex;
```
src/pages/api/test/index.ts
```typescript
import fetch from "node-fetch";
import { NextApiRequest, NextApiResponse } from "next";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== "POST") {
    res.status(405).json({ message: "Method Not Allowed" });
    return;
  }
  if (req.body == null) {
    res.status(405).json({ message: "Bad Request" });
  }
  const tweetURL: string = req.body.url;
  const publishTwitterAPIURL = `https://publish.twitter.com/oembed?url=`;
  const param = `&partner=&hide_thread=false`;
  const endpoint = publishTwitterAPIURL + encodeURIComponent(tweetURL) + param;
  const response = await fetch(endpoint);
  const resData = (await response.json());
  res.status(200).json({ response: resData });
}
```

![](https://storage.googleapis.com/zenn-user-upload/9deb18a29a52-20230615.png)

## 注意

レスポンスのHTMLをそのまま`parse(html)`しても正しく描画されない。以下のようにTwitterのスクリプトを非同期に読み込む必要があった

## さいごに

特定用途にツイートのURLを大量に保存しておいて、表示させたいときに使えそう（？）