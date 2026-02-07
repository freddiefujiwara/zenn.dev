---
title: " GAS×OpenAPIの「デバッグしづらい」を解消する、軽量APIテスターを自作した話"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gas,openapi,vue,swagger]
published: true
---
## はじめに
Google Apps Script（GAS）でAPIを作ると、**「Swagger UIだと動かない」「リダイレクトで詰まる」「JSONPに弱い」**といったデバッグ課題に直面しがちです。

そこで、**OpenAPI定義を直接読み取り、そのままリクエストを投げられる専用の軽量UI**を自作しました。本記事では、**実際のソースコードを踏まえて技術的に工夫したポイント**を中心に解説します。

---

## ざっくり構成
主要コンポーネントは以下です。

- **OpenAPI YAMLのエディタ**（左ペイン）
- **エンドポイント選択 + パラメータ入力UI**（右上）
- **レスポンス表示領域**（右下）

Vueで1画面完結のツールとして実装しています。
- [ツール](http://freddiefujiwara.com/openapi-fetch/)
- [ソース](https://github.com/freddiefujiwara/openapi-fetch/)

---

## 1. OpenAPIを解析してUIを自動生成

OpenAPI YAMLを読み取り、

- `servers` から **Base URL** を抽出
- `paths` から **Path / Method** を抽出
- `parameters` から **Query Params** を抽出

することで、UIを完全に自動生成しています。

### 実装ポイント：`parseOpenAPI` のミニマム設計

```js
// src/utils/openapiParser.js
import yaml from 'js-yaml';

export function parseOpenAPI(yamlString) {
  const doc = yaml.load(yamlString);
  const servers = doc.servers || [];
  const baseUrls = servers.map(s => s.url);

  // servers未指定でも動くようにフォールバック
  if (baseUrls.length === 0) {
    baseUrls.push('');
  }

  const paths = doc.paths || {};
  const endpoints = [];

  for (const [path, pathItem] of Object.entries(paths)) {
    const pathParameters = pathItem.parameters || [];

    for (const [method, operation] of Object.entries(pathItem)) {
      if (['get', 'post', 'put', 'delete', 'patch', 'options', 'head'].includes(method.toLowerCase())) {
        const operationParameters = operation.parameters || [];

        // Pathレベル/Operationレベルのパラメータを統合
        const combinedParams = new Map();
        pathParameters.forEach(p => combinedParams.set(`${p.name}-${p.in}`, p));
        operationParameters.forEach(p => combinedParams.set(`${p.name}-${p.in}`, p));

        const queryParams = Array.from(combinedParams.values())
          .filter(p => p.in === 'query')
          .map(p => ({
            name: p.name,
            description: p.description,
            schema: p.schema,
            required: p.required
          }));

        endpoints.push({
          path,
          method: method.toUpperCase(),
          queryParams
        });
      }
    }
  }

  return { baseUrls, endpoints };
}
```

**工夫ポイント**
- `servers` 未指定でも落ちないように `baseUrls` に空文字をフォールバック
- `path.parameters` と `operation.parameters` を **重複排除しつつ統合**
- **queryパラメータだけを抽出**し、フォーム入力UIに反映

---

## 2. GASのJSONPを考慮したリクエスト実行

GASのAPIで、**JSONPで返すケースがありますよね(自分だけかな)**です。
通常の `fetch` ではJSONPは扱えないので、**callbackパラメータがある場合は scriptタグで実行**するようにしています。

```js
// src/App.vue (抜粋)
const executeRequest = async () => {
  const callbackName = queryParamsValues.value['callback'];

  if (callbackName && callbackName.trim().length > 0 && method === 'GET') {
    const name = callbackName.trim();
    const script = document.createElement('script');

    window[name] = (data) => {
      responseData.value = JSON.stringify(data, null, 2);
      isLoading.value = false;
      script.remove();
      delete window[name];
    };

    script.src = url;
    script.onerror = () => {
      responseData.value = 'JSONP Error: Failed to load script.';
      isLoading.value = false;
      script.remove();
      delete window[name];
    };

    document.body.appendChild(script);
    return;
  }

  // JSONPでなければ普通にfetch
  const res = await fetch(url);
  const responseText = await res.text();
  responseData.value = responseText;
};
```

**工夫ポイント**
- `callback` が指定されていれば **自動的にJSONPモード**に切り替え
- `window[callback]` を動的に登録してレスポンスを受け取る

---

## 3. CORS回避のためにGETは“最小fetch”

GASのエンドポイントはCORSで引っかかることが多いため、
GETのリクエストは **プリフライトが走らないように最小構成でfetch**しています。

```js
// src/App.vue (抜粋)
if (method === 'GET') {
  // CORS preflight回避のため最小構成
  res = await fetch(url);
} else {
  res = await fetch(url, { method });
}
```

**工夫ポイント**
- GETは余計なヘッダを付けずに送る
- それ以外のメソッドは最低限の `method` 指定だけにとどめる

---

## 4. URL共有を実現するYAML圧縮

OpenAPI YAMLを毎回貼り付けるのは面倒なので、
**YAMLを圧縮してURLパスに埋め込む**仕組みを作っています。

```js
// src/utils/compression.js
import LZString from "lz-string";

export const encodeMarkdownToPath = (value) => {
  return value ? LZString.compressToEncodedURIComponent(value) : "";
};

export const decodeMarkdownFromPath = (encoded) => {
  const decompressed = LZString.decompressFromEncodedURIComponent(encoded);
  return decompressed || decodeURIComponent(encoded);
};
```

```js
// src/App.vue (抜粋)
watch(yamlContent, (newValue) => {
  const compressed = encodeMarkdownToPath(newValue);
  router.replace({ params: { compressedData: compressed } });
});

watch(() => route.params.compressedData, (newVal) => {
  const decoded = newVal ? decodeMarkdownFromPath(newVal) : DEFAULT_YAML;
  yamlContent.value = decoded;
}, { immediate: true });
```

**工夫ポイント**
- `lz-string` でURIエンコードに最適化した圧縮を実施
- URLに埋めることで **共有リンク化**
- 誰かにURLを渡すだけで同じOpenAPI定義を再現可能

---

## 5. エンドポイント選択UIの自動生成

OpenAPI解析結果から、`Base URL / Path / Method` のセレクトを動的に生成しています。

```html
<select v-model="selectedBaseUrl">
  <option v-for="url in parsedData.baseUrls" :key="url" :value="url">
    {{ url }}
  </option>
</select>

<select v-model="selectedPath">
  <option v-for="path in uniquePaths" :key="path" :value="path">
    {{ path }}
  </option>
</select>

<select v-model="selectedMethod">
  <option v-for="method in availableMethods" :key="method" :value="method">
    {{ method }}
  </option>
</select>
```

**工夫ポイント**
- `computed` で `uniquePaths` / `availableMethods` を切り替え
- YAMLを書き換えるとUIが即更新される

---

## まとめ

GASのAPIは便利ですが、**標準のOpenAPIツールと相性が悪い**のが悩みどころです。そこで、

- OpenAPI YAMLを**直接読み取ってUI化**
- JSONPやCORSなど**GAS特有の制約に対応**
- YAMLをURLに埋め込み**共有可能な開発体験**

という工夫を入れた専用ツールを作りました。

AIで `Code.js` からOpenAPIを生成 → そのままこのUIに貼って検証、という流れは、
**GAS + AI時代の新しい開発ワークフロー**として非常に相性が良いと感じています。

---

## 付録：AIでOpenAPIを生成するプロンプト例

```txt
あなたはAPI設計の専門家です。
以下のGASのCode.jsを読み取り、OpenAPI 3.0形式のYAMLを出力してください。
GAS特有のリダイレクトやJSONP(callback)が必要な場合はそれも考慮してください。
```
