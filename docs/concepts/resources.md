リソース
=========

リソースはゲーム内で使用される追加データで、コード内で記述する代わりにデータファイルを用いて保存されます。
Minecraftは2つの主要なシステムを持っています。一つはクライアント上で処理される「モデル」「テクスチャ」「ローカライゼーション」等の`assets`、もう一つは論理サーバーで処理される「レシピ」「ルートテーブル」のような`data`です。
[リソースパック][respack]は前者を、[データパック][datapack]は後者に干渉できます。

初期状態のMod開発キット(MDK)では、アセット・データはプロジェクトの`src/main/resources`フォルダに格納されます。

複数のリソースパック・データパックが有効な場合、これらは統合されます。一般的に、スタックの上位にあるデータパックがその下位にあるデータパックの要素をオーバーライドします。しかし、ローカライゼーションやタグ、データ等の特定のファイルはコンテンツごとに統合されます。Modは`resources`フォルダに基づきリソースパックとデータパックを構成しますが、これは"Mod Resources"パックのサブセットとして認識されます。Modリソースパックは無効化できませんが、他のデータパックでオーバーライドできます。Modデータパックは`/datapack` コマンドで無効化できます。これはVanillaの要素に基づいています。

Minecraft 1.11以降、全てのリソースはスネークケースな(小文字・区切りでの"_"の使用)パスとファイル名である必要があります。

`ResourceLocation`
------------------

Minecraftはリソースを`ResourceLocation`で特定します。`ResourceLocation`は次の2つのパーツで構成されます:「ネームスペース」「パス」。一般的に、`assets/<namespace>/<ctx>/<path>`とフォルダ構造を構築します。`ctx`は`ResourceLocation`の使用法に依存する属性指定のフラグメントです。  When a `ResourceLocation`が文字列として読取/書込された時、`<namespace>:<path>`のように確認できます。名前空間とコロンを省くと、`ResourceLocation`は`"minecraft"`名前空間に書込を行います。ですが、Modはそのmodidと同名の名前空間にリソースを割り当てることが推奨されています(例:`examplemod`というmodidの場合、`assets/examplemod` と`data/examplemod`にリソースを配置し、`ResourceLocation`では`examplemod:<path>`のように指定する)。 これは必須項目ではなく、いくつかの例ではMod内で複数の名前空間を使用します。`ResourceLocation`は同様にリソースシステム外でも、特定のオブジェクトの指定のために使用されることがあります(例:[レジストリ][registries])。

[respack]: ../resources/client/index.md
[datapack]: ../resources/server/index.md
[registries]: ./registries.md
