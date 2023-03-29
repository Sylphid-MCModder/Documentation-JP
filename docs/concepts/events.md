イベント
======

ForgeはVanillaとModの動作からイベントを傍受できるイベントバスを使用します。

例: Vanillaな棒を右クリックした時に、アクションを発生させるイベント

主要なイベントは`MinecraftForge#EVENT_BUS`にあります。`FMLJavaModLoadingContext#getModEventBus`には、特殊な用例のイベントが記載されています。更なる情報はここから得られるでしょう。

全てのイベントはこれらイベントバスを介して発動します: 大半のイベントはForgeのイベントバスを介しますが、一部のイベントはMod固有のイベントバスを介して発動します。

「イベントハンドラー」は、イベントをイベントバスに登録するためのメソッド群です。

イベントハンドラーの作成
-------------------------

イベントハンドラーは1つのパラメータを持ち、返り値を持ちません。メソッドは実装によってはstaticやinstanceになる可能性があります。

イベントハンドラーは`IEventBus#addListener`、またはジェネリックイベント向けの`IEventBus#addGenericListener`を用いて登録できます(`GenericEvent<T>`サブクラスにより示される)。いずれのリスナー追加者も、メソッドへの参照を表すコンシューマーを受け取ります。ジェネリックイベントハンドラーはジェネリッククラスを指定する必要があります。イベントハンドラーはすべて、メインクラスに含まれるコンストラクタで登録される必要があります。

```java
// ExampleModのメインクラス

// Modのイベントバス
private void modEventHandler(RegisterEvent event) {
	// 動作を記述
}

// Forgeのイベントバス
private static void forgeEventHandler(AttachCapabilitiesEvent<Entity> event) {
	// 略
}

// Modのコンストラクタ
modEventBus.addListener(this::modEventHandler);
forgeEventBus.addGenericListener(Entity.class, ExampleMod::forgeEventHandler);
```

### Instance Annotated Event Handlers

This event handler listens for the `EntityItemPickupEvent`, which is, as the name states, posted to the event bus whenever an `Entity` picks up an item.

```java
public class MyForgeEventHandler {
	@SubscribeEvent
	public void pickupItem(EntityItemPickupEvent event) {
		System.out.println("Item picked up!");
	}
}
```

To register this event handler, use `MinecraftForge.EVENT_BUS.register(...)` and pass it an instance of the class the event handler is within. If you want to register this handler to the mod specific event bus, you should use `FMLJavaModLoadingContext.get().getModEventBus().register(...)` instead.

### Static Annotated Event Handlers

An event handler may also be static. The handling method is still annotated with `@SubscribeEvent`. The only difference from an instance handler is that it is also marked `static`. In order to register a static event handler, an instance of the class won't do. The `Class` itself has to be passed in. An example:

```java
public class MyStaticForgeEventHandler {
	@SubscribeEvent
	public static void arrowNocked(ArrowNockEvent event) {
		System.out.println("Arrow nocked!");
	}
}
```

which must be registered like this: `MinecraftForge.EVENT_BUS.register(MyStaticForgeEventHandler.class)`.

### Automatically Registering Static Event Handlers

A class may be annotated with the `@Mod$EventBusSubscriber` annotation. Such a class is automatically registered to `MinecraftForge#EVENT_BUS` when the `@Mod` class itself is constructed. This is essentially equivalent to adding `MinecraftForge.EVENT_BUS.register(AnnotatedClass.class);` at the end of the `@Mod` class's constructor.

You can pass the bus you want to listen to the `@Mod$EventBusSubscriber` annotation. It is recommended you also specify the mod id, since the annotation process may not be able to figure it out, and the bus you are registering to, since it serves as a reminder to make sure you are on the correct one. You can also specify the `Dist`s or physical sides to load this event subscriber on. This can be used to not load client specific event subscribers on the dedicated server.

An example for a static event listener listening to `RenderLevelStageEvent` which will only be called on the client:

```java
@Mod.EventBusSubscriber(modid = "mymod", bus = Bus.FORGE, value = Dist.CLIENT)
public class MyStaticClientOnlyEventHandler {
	@SubscribeEvent
	public static void drawLast(RenderLevelStageEvent event) {
		System.out.println("Drawing!");
	}
}
```

!!! note
    This does not register an instance of the class; it registers the class itself (i.e. the event handling methods must be static).

Canceling
---------

If an event can be canceled, it will be marked with the `@Cancelable` annotation, and the method `Event#isCancelable()` will return `true`. The cancel state of a cancelable event may be modified by calling `Event#setCanceled(boolean canceled)`, wherein passing the boolean value `true` is interpreted as canceling the event, and passing the boolean value `false` is interpreted as "un-canceling" the event. However, if the event cannot be canceled (as defined by `Event#isCancelable()`), an `UnsupportedOperationException` will be thrown regardless of the passed boolean value, since the cancel state of a non-cancelable event event is considered immutable.

!!! important
    Not all events can be canceled! Attempting to cancel an event that is not cancelable will result in an unchecked `UnsupportedOperationException` being thrown, which is expected to result in the game crashing! Always check that an event can be canceled using `Event#isCancelable()` before attempting to cancel it!

Results
-------

Some events have an `Event$Result`. A result can be one of three things: `DENY` which stops the event, `DEFAULT` which uses the Vanilla behavior, and `ALLOW` which forces the action to take place, regardless if it would have originally. The result of an event can be set by calling `#setResult` with an `Event$Result` on the event. Not all events have results; an event with a result will be annotated with `@HasResult`.

!!! important
    Different events may use results in different ways, refer to the event's JavaDoc before using the result.

Priority
--------

Event handler methods (marked with `@SubscribeEvent`) have a priority. You can set the priority of an event handler method by setting the `priority` value of the annotation. The priority can be any value of the `EventPriority` enum (`HIGHEST`, `HIGH`, `NORMAL`, `LOW`, and `LOWEST`). Event handlers with priority `HIGHEST` are executed first and from there in descending order until `LOWEST` events which are executed last.

Sub Events
----------

Many events have different variations of themselves. These can be different but all based around one common factor (e.g. `PlayerEvent`) or can be an event that has multiple phases (e.g. `PotionBrewEvent`). Take note that if you listen to the parent event class, you will receive calls to your method for *all* subclasses.

Mod Event Bus
-------------

The mod event bus is primarily used for listening to lifecycle events in which mods should initialize. Each event on the mod bus is required to implement `IModBusEvent`. Many of these events are also ran in parallel so mods can be initialized at the same time. This does mean you can't directly execute code from other mods in these events. Use the `InterModComms` system for that.

These are the four most commonly used lifecycle events that are called during mod initialization on the mod event bus:

* `FMLCommonSetupEvent`
* `FMLClientSetupEvent` & `FMLDedicatedServerSetupEvent`
* `InterModEnqueueEvent`
* `InterModProcessEvent`

!!! note
    The `FMLClientSetupEvent` and `FMLDedicatedServerSetupEvent` are only called on their respective distribution.

These four lifecycle events are all ran in parallel since they all are a subclass of `ParallelDispatchEvent`. If you want to run run code on the main thread during any `ParallelDispatchEvent`, you can use the `#enqueueWork` to do so.

Next to the lifecycle events, there are a few miscellaneous events that are fired on the mod event bus where you can register, set up, or initialize various things. Most of these events are not ran in parallel in contrast to the lifecycle events. A few examples:

* `RegisterColorHandlersEvent`
* `ModelEvent$BakingCompleted`
* `TextureStitchEvent`
* `RegisterEvent`

A good rule of thumb: events are fired on the mod event bus when they should be handled during initialization of a mod.
