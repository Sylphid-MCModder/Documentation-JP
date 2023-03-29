Modライフサイクル
==============

Modのロードプロセス中、Modイベントバス上にてあまたのライフサイクルイベントが発動します。多くのアクションはこの再発動します。例えば、[オブジェクトの登録][registering]、[データ生成][datagen]、あるいは[他Modとの通信][imc]など。

イベントリスナーは`@EventBusSubscriber(bus = Bus.MOD)` かModのコンストラクタを使用して登録されるべきです。<br>例:

```Java
@Mod.EventBusSubscriber(modid = "mymod", bus = Mod.EventBusSubscriber.Bus.MOD)
public class MyModEventSubscriber {
  @SubscribeEvent
  static void onCommonSetup(FMLCommonSetupEvent event) { ... }
}

@Mod("mymod")
public class MyMod {
  public MyMod() {
    FMLModLoadingContext.get().getModEventBus().addListener(this::onCommonSetup);
  } 

  private void onCommonSetup(FMLCommonSetupEvent event) { ... }
}
```

!!! warning
    多くのライフサイクルイベントは並列して発動します。全てのModは同じタイミングで同じイベントを受け取ります。
    
    Modは、他ModのAPIへの通信やVanillaのシステムへの干渉などでは、*必ず*スレッドセーフであることを確認してください。後から実行するために`ParallelDispatchEvent#enqueueWork`を介しコードを先延ばしにします。

レジストリイベント
---------------

レジストリイベント(`NewRegistryEvent`と`RegisterEvent`の2つ)はModコンストラクタのロード後に発動されます。これらのイベントはModのロード中に同期しながら発動されます。

`NewRegistryEvent`は`RegistryBuilder`クラスを使用し、カスタムレジストリの登録を可能にします。

`RegisterEvent`は[登録オブジェクト][registering]をレジストリに登録します。このイベントは各レジストリに対して発動します。

データ生成
---------------

ゲームが[データジェネレータ][datagen]の実行をセットアップした時、最後に`GatherDataEvent`が発動されます。このイベントは、登録中のModデータの提供元を関連したデータジェネレータに登録します。このイベントは同期的に発動されます。

コモン・セットアップ
------------

`FMLCommonSetupEvent`は、[ケーパビリティー][capabilities]の登録のように、クライアントとサーバー両者に対してアクションを発動します。

特定サイドセットアップ
-----------

特定サイドセットアップイベントは[物理的サイド][sides]に基づきセットアップを行います: `FMLClientSetupEvent`はクライアントに、`FMLDedicatedServerSetupEvent`はサーバーに対して。これはクライアントに対するキーバインド設定のように、アクション・初期化が実行されるべきサイドを考慮する際に使用します。

InterModComms(Mod間内部通信)
-------------

これはMod間での通信の際に、どのModにメッセージを渡すかを指定します。以下の2イベントが定義済みです: `InterModEnqueueEvent`・`InterModProcessEvent`。

`InterModComms`はModのメッセージを保持する責任を持つクラスです。これらのメソッドは`ConcurrentMap`により、ライフサイクルイベントの発動中に実行しても安全です。

`InterModEnqueueEvent`の発動中、`InterModComms#sendTo`を用いることで他のModにメッセージを伝達できます。これらのメソッドはメッセージ伝達の際にModIDを使用し、そのキーはメッセージデータと関連づけられ、サプライヤはメッセージデータを保持します。さらに、メッセージ送信者はそのModIDで指定可能です。

そして`InterModProcessEvent`発動中、`InterModComms#getMessages`の使用で受信済メッセージのストリームを取得可能です。提供されたModIDは、ほとんどの場合メソッドが呼出されるModのModIDになります。さらに、Predicateはメッセージキーのフィルターに指定できます。これは 送信者のデータとデータキー、さらにそのデータ自身を含む`IMCMessage`のストリームを返します。
!!! note
    次の二つのライフサイクルイベントが定義されています: 
    * `FMLConstructModEvent`はModインスタンスの構築後、および`RegisterEvent`との発動前に発動します。
    * `FMLLoadCompleteEvent`は`InterModComms`イベント発動後に発動し、Modロードのプロセスが完了したことを通知します。

[registering]: ./registries.md#methods-for-registering
[capabilities]: ../datastorage/capabilities.md
[datagen]: ../datagen/index.md
[imc]: ./lifecycle.md#intermodcomms
[sides]: ./sides.md
