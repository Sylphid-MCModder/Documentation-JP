レジストリ
==========

レジストレーション(登録)はModの要素(ブロック、アイテム、バイオーム等)をゲームに認識させるための機構です。これはとても重要で、このプロセスを省くとModの要素がゲーム側に認識されないのはもちろん、ゲームの動作に支障を及ぼしたりクラッシュを発生させたりする場合があります。

多くの登録が必要なオブジェクトはForgeレジストリで定義されています。レジストリは、キーと値の単純な関連付けです。Forgeは[`ResourceLocation`][ResourceLocation]キーをオブジェクト登録時に使用します。これはオブジェクトに対し`ResourceLocation`が"レジストリ名"にアクセスできるようにするためです。

全ての登録可能なオブジェクトは固有のレジストリを持っています。Forgeレジストリの内部構造を見たい場合、`ForgeRegistries`クラスをご覧ください。全てのレジストリ名は固有のものである必要があります。しかし、異なるタイプのレジストリにおいて名前は競合しません。例として、`Block`レジストリと`Item`レジストリにて、`Block`と`Item`の間では同じ名称`example:thing`を使って登録しても競合しません。(ただし同じレジストリ内で同様のことを行った場合、後発のオブジェクトが先発のオブジェクトをオーバーライドします。)

レジストリ用メソッド
------------------

登録のためには以下二つの有効な手段があります。:
* `DeferredRegister`クラス
* `RegisterEvent`イベント

### DeferredRegister

`DeferredRegister`は登録の際に推奨されるメソッドです。登録の際に想定される問題を回避しつつ、静的な初期化の利便性と安定性を高めます。エントリーのサプライヤーのリストを保持し、`RegisterEvent`中にそれらを登録するだけです。

ブロックの登録例:

```java
private static final DeferredRegister<Block> BLOCKS = DeferredRegister.create(ForgeRegistries.BLOCKS, MODID);

public static final RegistryObject<Block> ROCK_BLOCK = BLOCKS.register("rock", () -> new Block(BlockBehaviour.Properties.of(Material.STONE)));

public ExampleMod() {
  BLOCKS.register(FMLJavaModLoadingContext.get().getModEventBus());
}
```

### `RegisterEvent`

`RegisterEvent`はオブジェクトを登録するための2つ目の手段です。この[イベント]はModのコンストラクタの起動後、およびコンフィグのロード前に発動します。オブジェクトは「レジストリキー」「レジストリオブジェクト名」「オブジェクト本体」を渡すことにより、`#register`に登録されます。追加の`#register`オーバーロードをすることで、「名称を指定してオブジェクトを登録するプロセス」に使用されたヘルパーを取得することもできます。不必要なオブジェクトの作成を回避するのに有用です。

例: (このイベントハンドラは*Modイベントバス*で登録されています)

```java
@SubscribeEvent
public void register(RegisterEvent event) {
  event.register(ForgeRegistries.Keys.BLOCKS,
    helper -> {
      helper.register(new ResourceLocation(MODID, "example_block_1"), new Block(...));
      helper.register(new ResourceLocation(MODID, "example_block_2"), new Block(...));
      helper.register(new ResourceLocation(MODID, "example_block_3"), new Block(...));
      // ...
    }
  );
}
```

### Forge以外のレジストリ

全てのレジストリがForgeで実装されているわけではありません。`LootItemConditionType`のような静的レジストリとして実装することで安全に使用できます。また、`ConfiguredFeature`や幾つかのワールド生成レジストリのような動的レジストリを用いることでJSON形式の設定を設けることもできます。 `DeferredRegister#create`は、Modderに対し任意のVanillaレジストリキーを指定し`RegistryObject`を作成できるようにするオーバーロードを持っています。レジストリメソッドとModイベントバスへのアタッチは他の`DefferedRegister`と同様です。

!!! important
    動的レジストリオブジェクトはデータファイルを通して**のみ**登録可能です(例:JSON, YAML)。コード内宣言は**できません**。

```java
private static final DeferredRegister<LootItemConditionType> REGISTER = DeferredRegister.create(Registries.LOOT_CONDITION_TYPE, "examplemod");

public static final RegistryObject<LootItemConditionType> EXAMPLE_LOOT_ITEM_CONDITION_TYPE = REGISTER.register("example_loot_item_condition_type", () -> new LootItemConditionType(...));
```

!!! note
    いくつかのクラスは自己による登録をできません。その代わりに、`*Type`が登録され、前者のコンストラクタで使用されます。例えば、[`BlockEntity`][blockentity]は`BlockEntityType`、`Entity`は`EntityType`を所持しています。`*Type`クラスは、必要に応じて包容型を生成するだけです。
    
    これらのファクトリーは`*Type$Builder`クラスにより作成されます。例: (`REGISTER`は`DeferredRegister<BlockEntityType>`を指す)
    ```java
    public static final RegistryObject<BlockEntityType<ExampleBlockEntity>> EXAMPLE_BLOCK_ENTITY = REGISTER.register(
      "example_block_entity", () -> BlockEntityType.Builder.of(ExampleBlockEntity::new, EXAMPLE_BLOCK.get()).build(null)
    );
    ```

登録済オブジェクトのリフレッシュ
------------------------------

登録したオブジェクトは作成時・登録時にフィールドに格納しないでください。これらは`RegisterEvent`の発動のたびに、常に新しく作成・登録されます。これは来るべきForgeの新バージョンにおけるModの動的ロード/アンロードのためです。

登録済オブジェクトは`RegistryObject`か`@OPbjectHolder`フィールドを通し参照される必要があります。

### RegistryObjectsの使用

`RegistryObject`は登録済オブジェクトが利用可能になった際に、その参照を得るために使用できます。これは`DeferredRegister`が登録済オブジェクトの参照を取得するのに使用されます。この参照は`RegisterEvent`が発動した後に、`@ObjectHolder`アノテーションの内容に従い更新されます。

`RegistryObject`の入手には、`RegistryObject#create`を`ResourceLocation`とレジストリ可能オブジェクトの`IForgeRegistry`とともに使用します。代わりにレジストリ名を使用することで、カスタムレジストリを使うことができます。`RegistryObject`を`public static final`フィールドに格納し、必要に応じ`#get`を実行してください。

`RegistryObject`の使用例:

```java
public static final RegistryObject<Item> BOW = RegistryObject.create(new ResourceLocation("minecraft:bow"), ForgeRegistries.ITEMS);

// 'neomagicae:mana_type'が有効なレジストリであり、'neomagicae:coffeinum'がこのレジストリ内の有効なオブジェクトであることが前提。
public static final RegistryObject<ManaType> COFFEINUM = RegistryObject.create(new ResourceLocation("neomagicae", "coffeinum"), new ResourceLocation("neomagicae", "mana_type"), "neomagicae"); 
```

### @ObjectHolderの使用

レジストリの登録済オブジェクトは`@ObjectHolder`アノテーションをクラス・フィールドに設定し、`ResourceLocation`に指定のレジストリのオブジェクトを認識させるため適切な情報を提供するすることで`public static`フィールドにインジェクトできます。

`@ObjectHolder`の使用規則は以下のとおりです。:

* `@ObjectHolder`アノテーションがクラスを修飾する際、明示的な宣言がない場合はこの値は全てデフォルトの名前空間に配置されます。
* `@Mod`アノテーションがクラスを修飾する時、 明示的な宣言がない場合は`modid`はデフォルトの名前空間で宣言されます。
* 次の場合、フィールドは注入対象とみなされます:
  * `public static`がある場合;
  * **フィールド**が`@ObjectHolder`アノテーションで修飾されており、かつ:
    * 名前の値が明示的に宣言されている; 
    * レジストリ名の値が明示的に宣言されている
  * _フィールドに対し対応する値がない場合、コンパイル時例外が発生します。_
* _`ResourceLocation`の結果が不完全または不正な場合(パスに不正な文字があった時)、例外が発生します。_
* エラーや例外がなかった場合、インジェクションは成功します。
* 上記規則が一つでも違反された場合、アクションは発生しません(そしてメッセージがログに残ります)。

`@ObjectHolder`で修飾されたフィールドは`RegisterEvent`イベント発生後に`RegistryObject`とともにインジェクトされます。

!!! note
    インジェクト時にオブジェクトが存在しなかった場合、オブジェクトはインジェクトされず、代わりにログにDEBUGタイプのメッセージが記録されます。

As these rules are rather complicated, here are some examples:

```java
class Holder {
  @ObjectHolder(registryName = "minecraft:enchantment", value = "minecraft:flame")
  public static final Enchantment flame = null;     // Annotation present. [public static] is required. [final] is optional.
                                                    // Registry name is explicitly defined: "minecraft:enchantment"
                                                    // Resource location is explicitly defined: "minecraft:flame"
                                                    // To inject: "minecraft:flame" from the [Enchantment] registry

  public static final Biome ice_flat = null;        // No annotation on the field.
                                                    // Therefore, the field is ignored.

  @ObjectHolder("minecraft:creeper")
  public static Entity creeper = null;              // Annotation present. [public static] is required.
                                                    // The registry has not been specified on the field.
                                                    // Therefore, THIS WILL PRODUCE A COMPILE-TIME EXCEPTION.

  @ObjectHolder(registryName = "potion")
  public static final Potion levitation = null;     // Annotation present. [public static] is required. [final] is optional.
                                                    // Registry name is explicitly defined: "minecraft:potion"
                                                    // Resource location is not specified on the field
                                                    // Therefore, THIS WILL PRODUCE A COMPILE-TIME EXCEPTION.
}
```

Creating Custom Forge Registries
--------------------------------

Custom registries can usually just be a simple map of key to value. This is a common style; however, it forces a hard dependency on the registry being present. It also requires that any data that needs to be synced between sides must be done manually. Custom Forge Registries provide a simple alternative for creating soft dependents along with better management and automatic syncing between sides (unless told otherwise). Since the objects also use a Forge registry, registration becomes standardized in the same way.

Custom Forge Registries are created with the help of a `RegistryBuilder`, through either `NewRegistryEvent` or the `DeferredRegister`. The `RegistryBuilder` class takes various parameters (such as the registry's name, id range, and various callbacks for different events happening on the registry). New registries are registered to the `RegistryManager` after `NewRegistryEvent` finishes firing.

Any newly created registry should use its associated [registration method][registration] to register the associated objects.

### Using NewRegistryEvent

When using `NewRegistryEvent`, calling `#create` with a `RegistryBuilder` will return a supplier-wrapped registry. The supplied registry can be accessed after `NewRegistryEvent` has finished posting to the mod event bus. Getting the custom registry from the supplier before `NewRegistryEvent` finishes firing will result in a `null` value.

### With DeferredRegister

The `DeferredRegister` method is once again another wrapper around the above event. Once a `DeferredRegister` is created in a constant field using the `#create` overload which takes in the registry name and the mod id, the registry can be constructed via `DeferredRegister#makeRegistry`. This takes in  a supplied `RegistryBuilder` containing any additional configurations. The method already populates `#setName` by default. Since this method can be returned at any time, a supplied version of an `IForgeRegistry` is returned instead. Getting the custom registry from the supplier before `NewRegistryEvent` is fired will result in a `null` value.

!!! important
    `DeferredRegister#makeRegistry` must be called before the `DeferredRegister` is added to the mod event bus via `#register`. `#makeRegistry` also uses the `#register` method to create the registry during `NewRegistryEvent`.

Handling Missing Entries
------------------------

There are cases where certain registry objects will cease to exist whenever a mod is updated or, more likely, removed. It is possible to specify actions to handle the missing mapping through the third of the registry events: `MissingMappingsEvent`. Within this event, a list of missing mappings can be obtained either by `#getMappings` given a registry key and mod id or all mappings via `#getAllMappings` given a registry key.

!!! important
    `MissingMappingsEvent` is fired on the **Forge** event bus.

For each `Mapping`, one of four mapping types can be selected to handle the missing entry:

| Action | Description |
| :---:  |     :---    |
| IGNORE | Ignores the missing entry and abandons the mapping. |
|  WARN  | Generates a warning in the log. |
|  FAIL  | Prevents the world from loading. |
| REMAP  | Remaps the entry to an already registered, non-null object. |

If no action is specified, then the default action will occur by notifying the user about the missing entry and whether they still would like to load the world. All actions besides remapping will prevent any other registry object from taking the place of the existing id in case the associated entry ever gets added back into the game.

[ResourceLocation]: ./resources.md#resourcelocation
[registration]: #methods-for-registering
[イベント]: ./events.md
[blockentity]: ../blockentities/index.md
