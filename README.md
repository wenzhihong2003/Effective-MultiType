# 前言

在开发我的 **[TimeMachine](https://github.com/drakeet/TimeMachine)** 时，我有一个复杂的聊天页面，于是我设计了我的类型池系统，它是完全解耦的，于是我能够轻松将它抽离出来分享，并给它取名为 **MultiType**.

从前，**比如我们写一个类似微博列表页面**，这样的列表是十分复杂的：有纯文本的、带转发原文的、带图片的、带视频的、带文章的等等，甚至穿插一条可以横向滑动的好友推荐条目。不同的 Item 类型众多，而且随着业务发展，还会更多。如果我们使用传统的开发方式，经常要做一些繁琐的工作，代码可能都堆积在一个 `Adapter` 中：我们需要覆写 `RecyclerView.Adapter` 的 `getItemViewType` 方法，罗列一些 `type` 整型常量，并且 `ViewHolder` 转型、绑定数据也比较麻烦。一旦产品需求有变，或者产品设计说需要增加一种新的 Item 类型，我们需要去代码堆里找到我们原来的逻辑去修改，或者找到正确的位置去增加代码。这些过程都比较繁琐，侵入较强，需要小心翼翼，以免改错影响到其他地方。

现在好了，我们有了 **MultiType**，简单来说，**MultiType** 就是一个多类型列表视图的中间分发框架。它本是为聊天页面开发的，聊天页面的消息类型也是有大量不同种类，并且新增频繁，而 **MultiType** 能够轻松胜任，代码模块化，随时可拓展新的类型进入列表当中。它内建了 `类型` - `View` 的复用池系统，支持 `RecyclerView`，使用简单灵活，令代码清晰、拥抱变化。

因此，我写了这篇文章，目的有几个：一是以作者的角度对 **MultiType** 进行入门和进阶详解。二是传递我开发过程中的思想、设计理念，这些偏细腻的内容，即使不使用 **MultiType**，想必也能带来很多启发。最后就是把我自觉得不错的东西分享给大家，试想如果你制造的东西很多人在用，即使没有带来任何收益，也是一件很自豪的事情。

# 目录

- [MultiType 的特性](#multitype-的特性)
- [总览](#总览)
- [MultiType 基础用法](#multitype-基础用法)
- [设计思想](#设计思想)
- [高级用法](#高级用法)
  - [使用 MultiTypeTemplates 插件自动生成代码](#使用-multitypetemplates-插件自动生成代码)
  - [使用 全局类型池](#使用-全局类型池)
  - [一个类型对应多个 ViewProvider](#一个类型对应多个-viewprovider)
  - [与 ViewProvider 通讯](#与-viewprovider-通讯)
  - [使用断言，比传统 Adapter 更加易于调试](#使用断言比传统-adapter-更加易于调试)
  - [支持 Google AutoValue](#支持-google-autovalue)
  - [对 class 进行二级分发](#对-class-进行二级分发)
  - [MultiType 与下拉刷新、加载更多、HeaderView、FooterView、Diff](#multitype-与下拉刷新加载更多headerviewfooterviewdiff)
  - [实现 RecyclerView 嵌套横向 RecyclerView](#实现-recyclerview-嵌套横向-recyclerview)
  - [实现线性布局和网格布局混排列表](#实现线性布局和网格布局混排列表)
  - [数据扁平化处理](#数据扁平化处理)
- [更多示例](#更多示例)
  - **仿造微博的数据结构和二级 ViewProvider**
  - drakeet/about-page
  - 线性和网格布局混排
  - drakeet/TimeMachine
  - 类似 Bilibili iOS 端首页
- [Q & A](#q--a)
  - Q: 全局类型池的主要作用是什么，能取消全局的使用吗？
  - Q: 使用全局类型的话，只能是在 Application 中进行注册吗？
  - Q: 为什么不全然使用全局类型池？
  - Q: 觉得 MultiType 不够精简，应该怎么做？
- [感谢](#感谢)
- [引用文献](#引用文献)

# MultiType 的特性

- 轻盈，整个类库只有 10 个类文件，`aar` 或 `jar` 包大小只有 10KB
- 周到，支持 局部类型池 和 全局类型池，并支持二者共用，当出现冲突时，以局部的为准
- 灵活，几乎所有的部件(类)都可被替换、可继承定制，面向接口/抽象编程
- 纯粹，只负责本分工作，专注多类型的列表视图 类型分发
- 高效，没有性能损失，内存友好，最大限度发挥 `RecyclerView` 的复用性
- 可读，代码清晰干净、设计精巧，极力避免复杂化，可读性很好，为拓展和自行解决问题提供了基础

# 总览

[![](http://ww4.sinaimg.cn/large/86e2ff85gw1f9bf092eraj21kw0xr1el.jpg)](http://ww2.sinaimg.cn/large/86e2ff85gw1f9bekb34xfj21kw0y3av4.jpg)

# MultiType 基础用法

可能有的新手看到以上特性介绍说什么 "冲突"、抽象编程的，还有那看不懂的总览图，都是一脸懵逼，完全不要紧，不懂可以回过头来再看，我们先从基础用法入手，其实 **MultiType** 使用起来特别简单。使用 **MultiType** 一般情况下只要 maven 引入 + 三个小步骤。之后还会介绍使用插件生成代码方式，步骤将更加简化：

### 引入

在你的 `build.gradle`:

```groovy
dependencies {
    compile 'me.drakeet.multitype:multitype:2.2.1'
}
```

> 注：**MultiType** 内部引用了 `recyclerview-v7:24.2.1`，如果你不想使用这个版本，可以使用 `exclude` 将它排除掉，再自行引入你选择的版本。示例如下：

```groovy
dependencies {
    compile('me.drakeet.multitype:multitype:2.2.1', {
       exclude group: 'com.android.support'
    })
    compile 'com.android.support:recyclerview-v7:你选择的版本'
}
```

## 使用

**Step 1**. 创建一个 `class implements Item`，它将是你的数据类型或 Java bean/model. 

 这是一个类似 Java `Serializable` 接口，只要显式 `implements` 即可，除此之外什么都不用做。它的作用是让 MultiType 把你的所有实体类都视为 `Item` 接口，而且由于它仅是个接口，你仍然可以随意安排你的继承关系。示例如下：

```java
public class Category implements Item {

    @NonNull public String text;

    public Category(@NonNull final String text) {
        this.text = text;
    }
}
```

**Step 2**. 创建一个 `class` 继承 `ItemViewProvider`. 

 `ItemViewProvider` 是个抽象类，其中 `onCreateViewHolder` 方法用于生产你的 Item View Holder, `onBindViewHolder` 用于绑定数据到 `View`s. 一般一个 `ItemViewProvider` 类在内存中只会有一个实例对象，MultiType 内部将复用这个 provider 对象来生产所有相关的 Item Views 和绑定数据。示例：

```java
public class CategoryViewProvider
    extends ItemViewProvider<Category, CategoryViewProvider.ViewHolder> {

    @NonNull @Override
    protected ViewHolder onCreateViewHolder(
        @NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        View root = inflater.inflate(R.layout.item_category, parent, false);
        return new ViewHolder(root);
    }

    @Override
    protected void onBindViewHolder(
        @NonNull ViewHolder holder, @NonNull Category category) {
        holder.category.setText(category.text);
    }

    static class ViewHolder extends RecyclerView.ViewHolder {

        @NonNull private final TextView category;

        ViewHolder(@NonNull View itemView) {
            super(itemView);
            this.category = (TextView) itemView.findViewById(R.id.category);
        }
    }
}
```

**Step 3**. 好了，你不必再创建新的类文件了，在 `Activity` 中加入 `RecyclerView` 和 `List` 并注册你的类型就完事了，示例：

```java
public class MainActivity extends AppCompatActivity {

    private MultiTypeAdapter adapter;

    /* Items 等价于 ArrayList<Item> */
    private Items items;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.list);

        items = new Items();
        adapter = new MultiTypeAdapter(items);

        /* 注册类型和 View 的对应关系 */
        adapter.register(Category.class, new CategoryViewProvider());
        adapter.register(Song.class, new SongViewProvider());

        /* 模拟加载数据，也可以稍后再加载，然后使用
         * adapter.notifyDataSetChanged() 刷新列表 */
        for (int i = 0; i < 20; i++) {
            items.add(new Category("Songs"));
            items.add(new Song("小艾大人", R.drawable.avatar_dakeet));
            items.add(new Song("许岑", R.drawable.avatar_cen));
        }

        recyclerView.setAdapter(adapter);
    }
}
```

大功告成！这就是 **MultiType** 的基础用法了，简单、符合直觉。其中 `onCreateViewHolder` 和 `onBindViewHolder` 方法名沿袭了使用 `RecyclerView` 的习惯，令人一目了然，减少了新人的学习成本。


# 设计思想

**MultiType** 设计伊始，我给它定了几个原则：

- 要简单，便于他人阅读代码。

  因此我极力去避免将它复杂化，比如引入显性 item id 机制（MultiType 内部有隐性 id），比如加入许多不相干的内容，比如使用 apt + 注解完成类型和 View 自动绑定、自动注册，再比如，使用反射。这些我都是拒绝的。我想写人人可读的代码，使用简单的方式，去实现复杂的需求。过多不相干、没必要的代码，将会使项目变得令人晕头转向，难以阅读，遇到需要定制、解决问题的时候，无从下手。

- 要灵活，便于拓展和适应各种需求

  很多人会得意地告诉我，他们把 **MultiType** 源码精简成三四个类，甚至一个类，以为代码越少就是越好，这我也是不能赞同的。**MultiType** 考虑得比他们更远，这是一个提供给大众使用的类库，过度的精简只会使得灵活性大幅失去。**它或许不是使用起来最简单的，但很可能是使用起来最灵活的。** 在我看来，灵活性的优先级大于简单性。因此，**MultiType** 各个组件都是以接口或抽象进行连接，这意味着它所有的角色、组件都可以被替换，或者被拓展和继承。如果你觉得它使用起来还不够简单，完全可以通过继承来封装出更具体符合你使用需求的方法。它已经暴露了足够丰富、周到的接口以供自行实现，我们不应该直接去修改源码，这会导致一旦后续发现你的精简版满足不了你的需求时，已经没有回头路了。

- 要直观，使用起来能令项目代码更清晰、模块化

  **MultiType** 提供的 `ItemViewProvider` 沿袭了 `RecyclerView Adapter` 的接口命名，使用起来更加舒适，符合习惯。另外，手动写一个新的 `ItemViewProvider` 需要提供了 类型 泛型，虽然略微有点儿麻烦，但能带来一些好处，指定泛型之后，我们不再需要自己做强制转型，而且代码能够显式表明 `ItemViewProvider` 和 `Item class` 的对应关系，遵循了简单可依赖的原则。另外，现在我们有 **MultiTypeTemplates** 插件来自动生成代码，这个过程变得更加顺滑简单。


# 高级用法

## 使用 MultiTypeTemplates 插件自动生成代码

之前我们介绍了通过 3 个步骤完成 **MultiType** 的初次接入使用，实际上这个过程可以更加简化，**MultiType** 提供了 Android Studio 插件来自动生成代码：**MultiTypeTemplates**，源码也是开源的，[https://github.com/drakeet/MultiTypeTemplates](https://github.com/drakeet/MultiTypeTemplates)，不仅提供了一键生成 `Item` 和 `ItemViewProvider`，而且**是一个很好的利用代码模版自动生成代码的示例。**其中使用到了官方提供的代码模版 API，也用到了我自己发明的更灵活修改模版内容的方法，有兴趣做这方面插件的可以看看。

话说回来，安装和使用 **MultiTypeTemplates** 非常简单：

**Step 1.** 打开 Android Studio 的`设置` -> `Plugin` -> `Browse repositories`，搜索 `MultiTypeTemplates` 即可获得下载安装：

![](http://ww4.sinaimg.cn/large/86e2ff85gw1f935l0kwilj21kw0t3akm.jpg)

**Step 2.** 安装完成后，重启 Android Studio. 右键点击你的 package，选择 `New` -> `MultiType Item`，然后输入你的 `Item` 名字，它就会自动生成 `Item` and `ItemViewProvider` 文件和代码。

比如你输入的是 "Category"，它就会自动生成 `Category.java` 和 `CategoryViewProvider.java`.

特别方便，相信你会很喜欢它。未来这个插件也将会支持自动生成布局文件，这是目前欠缺的，但不要紧，其实 AS 在这方面已经很方便了，对布局 `R.layout.item_category` 使用 `alt + enter` 快捷键即可自动生成布局文件。


## 使用 全局类型池

在基础用法中，我们并没有提到 全局类型池，实际上，**MultiType** 支持 局部类型池 和 全局类型池，并支持二者共用，当出现冲突时，以局部的为准。使用局部类型池就如上面的示例，调用 `adapter.register()` 即可。而使用全局类型池也是很容易的，**MultiType** 提供了一个内置的 `GlobalMultiTypePool` 作为全局类型池来存储类型和 view 关系，使用如下：

只要在使用你的全局类型之前任意位置注册类型，通过调用 `GlobalMultiTypePool.register(...)` 静态方法完成注册。推荐统一在 `Application` 初始便进行注册，这样代码便于寻找和阅读。

之后回到你的 `Activity`，调用 `adapter.applyGlobalMultiTypePool()` 方法应用你注册过的全局类型即可。

`GlobalMultiTypePool` 让一些普适性的类型能够全局共用，但使用全局类型池不当也会带来问题，这是没有全然采用全局类型池的原因。问题在于全局类型池是静态的，如果你在 `Activity` 中注册**全局**类型（虽然并不推荐。因为全局类型最好统一在一个地方注册，便于管理），并传入带 `Activity` 引用的变量进去，就可能造成内存泄露。举个例子，如下是一个很常见的场景，我们把一个点击回调传递给 `provider`，并注册到全局类型池：

```java
public class LeakActivity extends Activity {

    private MultiTypeAdapter adapter;
    private Items items;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.list);
        items = new Items();
        adapter = new MultiTypeAdapter(items);

        OnClickListener listener = new OnClickListener() {
            @Override
            public void onClick(View v) {
                // ...
            }
        }

        /* 在 applyGlobalMultiTypePool 之前注册全局 */
        GlobalMultiTypePool.register(Post.class, new PostViewProvider(listener));
        
        adapter.applyGlobalMultiTypePool(); // <- 使全局的类型加入到局部中来

        recyclerView.setAdapter(adapter);
    }
}
```

由于 Java 匿名内部类 或 非静态内部类，都会默认持有 外部类 的引用，比如这里的 `OnClickListener` 匿名类对象会持有 `LeakActivity.this`，当 `listener` 传递给 `new PostViewProvider()` 构造函数的时候，`GlobalMultiTypePool` 内置的静态类型池将长久持有 `provider -> listener -> LeakActivity.this` 引用链，若没有及时释放，就会引起内存泄露。

因此，**在使用全局类型池时，最好不要给 `provider` 传递回调对象或者外部引用，否则就应手动释放或使用弱引用(`WeakReference`)。**除此之外，全局类型池没有什么其他问题，类型池都只会持有 `class` 和非常轻薄的 `provider` 对象。我做过一个试验，就算拥有上万个类型和 `provider`，内存占用也是很少的，索引速度也很快，在主线程连续注册一万个类型花费不过 10 毫秒的时间，何况一般一个应用根本不可能有这么多类型，完全不必担心这方面的问题。

另外一个特性是，不管是全局类型池还是局部类型池，都支持重复注册类型。当发现重复时，之后注册的会把之前注册的类型覆盖掉，因此对于全局类型池，需要谨慎进行重复注册，以免影响到其他地方。


## 一个类型对应多个 `ViewProvider`

> 注：本文所有的 `ViewProvider` 都指的是 `ItemViewProvider`.

**MultiType** 天然支持一个类型对应多个 `ViewProvider`，但仅限于在不同的列表中。比如你在 `adapter1` 中注册了 `Post.class` 对应 `SinglePostViewProvider`，在另一个 `adapter2` 中注册了 `Post.class` 对应 `PostDetailViewProvider`，这便是一对多的场景。只要是在不同的局部类型池中，无论如何都不会相互干扰，都是允许的。

而对于在 同一个列表中 一对多的问题，首先这种场景非常少见，再者不管支不支持一对多，开发者都要去判断哪个时候运用哪个 `ViewProvider`，这是逃不掉的，否则程序就无所适从了。因此，**MultiType** 不去特别解决这个问题，**如果要实现同一个列表中一对多，只要空继承你的类型，然后把它视为新的类型，注册到你的类型池中即可**。


## 与 `ViewProvider` 通讯

`ItemViewProvider` 对象可以接受外部类型、回调函数，只要在使用之前，传递进去即可，例如：

```java
OnClickListener listener = new OnClickListener() {
    @Override
    public void onClick(View v) {
        // ...
    }
}
adapter.register(Post.class, new PostViewProvider(xxx, listener));
```

但话说回来，对于点击事件，能不依赖 `provider` 外部内容的话，最好就在 `provider` 内部完成。`provider` 内部能够接收到 Views 和 数据，大部分情况下，完全有能力不依赖外部 独立完成逻辑。这样能使代码更加模块化，便于解耦，例如下面便是一个完全自包含的例子：

```java
public class SquareViewProvider extends ItemViewProvider<Square, SquareViewProvider.ViewHolder> {

    @NonNull @Override
    protected ViewHolder onCreateViewHolder(
        @NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        View root = inflater.inflate(R.layout.item_square, parent, false);
        return new ViewHolder(root);
    }

    @Override
    protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull Square square) {
        holder.square = square;
        holder.squareView.setText(valueOf(square.number));
        holder.squareView.setSelected(square.isSelected);
    }

    public class ViewHolder extends RecyclerView.ViewHolder {

        private TextView squareView;
        private Square square;

        ViewHolder(final View itemView) {
            super(itemView);
            squareView = (TextView) itemView.findViewById(R.id.square);
            itemView.setOnClickListener(new View.OnClickListener() {
                @Override public void onClick(View v) {
                    itemView.setSelected(square.isSelected = !square.isSelected);
                }
            });
        }
    }
}
```

## 使用断言，比传统 Adapter 更加易于调试

**众所周知，如果一个传统的 `RecyclerView` `Adapter` 内部有异常导致崩溃，它的异常栈是不会指向到你的 `Activity`**，这给我们开发调试过程中带来了麻烦。如果我们的 `Adapter` 是复用的，就不知道是哪一个页面崩溃。而对于 `MultiTypeAdapter`，我们显然要用于多个地方，而且可能出现开发者忘记注册类型等等问题。为了便于调试，开发期快速失败，**MultiType** 提供了很方便的断言 API: `MultiTypeAsserts`，使用方式如下：

```java
import static me.drakeet.multitype.MultiTypeAsserts.assertAllRegistered;
import static me.drakeet.multitype.MultiTypeAsserts.assertHasTheSameAdapter;

public class SimpleActivity extends MenuBaseActivity {

    private Items items;
    private MultiTypeAdapter adapter;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.list);

        items = new Items();
        adapter = new MultiTypeAdapter(items);
        adapter.register(TextItem.class, new TextItemViewProvider());

        for (int i = 0; i < 20; i++) {
            items.add(new TextItem(valueOf(i)));
        }

        /* 断言所有使用的类型都已注册 */
        assertAllRegistered(adapter, items);
        recyclerView.setAdapter(adapter);
        /* 断言 recyclerView 使用的是正确的 adapter */
        assertHasTheSameAdapter(recyclerView, adapter);
    }
}
```

`assertAllRegistered` 和 `assertHasTheSameAdapter` 都是可选择性使用，`assertAllRegistered` 需要在加载或更新数据之后， `assertHasTheSameAdapter` 必须在 `recyclerView.setAdapter(adapter)` 之后。

这样做以后，`MultiTypeAdapter` 相关的异常都会报到你的 `Activity`，并且会详细注明出错的原因，而如果符合断言，断言代码不会有任何副作用或影响你的代码逻辑，这时你可以把它当作废话。关于这个类的源代码也是很简单，有兴趣可以直接看看源码：[drakeet/multitype/MultiTypeAsserts.java](https://github.com/drakeet/MultiType/blob/master/library/src/main/java/me/drakeet/multitype/MultiTypeAsserts.java)


## 支持 Google AutoValue

[AutoValue](https://github.com/google/auto/tree/master/value) 是 Google 提供的一个在 Java 实体类中自动生成代码的类库，使你更专注于处理项目的其他逻辑，它可使代码更少，更干净，以及更少的 bug. 

当我们使用传统方式创建一个 Java 模型类的时候，经常需要写一堆 `toString()`、`hashCode()`、getter、setter 等等方法，而且对于 Android 开发，大多情况下需要实现 `Parcelable` 接口。这样的结果是，我本来想要一个只有几个属性的小模型类，但出于各种原因，这个模型类方法数变得十分繁复，阅读起来很不清爽，并且难免会写错内容。AutoValue 的出现解决了这个问题，我们只需定义一些抽象类交给 AutoValue，AutoValue 会**自动**生成该抽象类的具体实现子类，并携带各种样板代码。

更详细的介绍内容和使用教程，我会在文章末尾会给出 AutoValue 的相关链接，不熟悉 AutoValue 可以借此机会看一下，在这里就不做过多介绍了。新手暂时看不懂也不必纠结，了解之后都是十分容易的。

**MultiType** 支持了 Google AutoValue，支持自动映射某个已经注册的类型的**子类**到同一 View Provider，规则是：如果子类**有**注册，就用注册的映射关系；如果子类**没**注册，则该子类对象使用注册过的父类映射关系。

## 对 class 进行二级分发

我的另外一个项目，即一开始提到的 **TimeMachine**，它是一个看起来特别像聊天软件的 SDK，但还处于非常初期阶段，大家可以不必太关心它。话说回来，在我的 **TimeMachine** 中，我的消息数据结构是 `Message` - `MessageContent`，`Message` 包含了 `MessageContent`. 因此产生了一个问题，我的 `message` 对象们都是一样的 `Message` 类型，但 `message` 包含的 `content` 对象不一样，我需要根据 `content` 来分发数据到 `ItemViewProvider`，但我加入 `Items` 中的数据都是 `Message` 对象，因此，如果什么也不做，它们会被视为同一类型。对于这种场景，我们可以继承 `MultiTypeAdapter` 并覆写 `onFlattenClass(@NonNull Item message)` 方法进行二级分发，以我的 `MessageAdapter` 为例：

```java
public class MessageAdapter extends MultiTypeAdapter {

    public MessageAdapter(@NonNull List<Message> messages) {
        super(messages);
    }


    @NonNull @Override public Class onFlattenClass(@NonNull Item message) {
        return ((Message) message).content.getClass();
    }
}
```

是不是十分简单？这样以后，我就可以直接将 `MessageContent.class` 注册进类型池，而将包含不同 `content` 的 `Message` 对象 add 进 `Items` List，`MessageAdapter` 会自动取出 `message` 的 `content` 对象，并以它为基准定位 `ItemViewProvider` 同时会把整个 `Message `对象发给 `provider`，`provider` 可进行分层，如下：

```java
public abstract class MessageViewProvider<C extends Content, V extends RecyclerView.ViewHolder>
    extends ItemViewProvider<Message, V> {

    @SuppressWarnings("unchecked") @Override
    protected void onBindViewHolder(@NonNull V holder, @NonNull Message message) {
        onBindViewHolder(holder, (C) message.content, message);
    }

    /* 留给子类的抽象方法 */
    protected abstract void onBindViewHolder(
        @NonNull V holder, @NonNull C content, @NonNull Message message);
}
```

总的来说，对 class 进行二级分发往往要伴随着对 `ItemViewProvider` 进行二级处理，对此我给出了一个详细的示例，到本文到 "示例" 章节中我们会再详细介绍 `ItemViewProvider` 二级分发的场景和更具体运用。

## MultiType 与下拉刷新、加载更多、HeaderView、FooterView、Diff

**MultiType** 设计从始至终，都极力避免往复杂化方向发展，一开始我的设计宗旨就是它应该是一个非常纯粹的、专一的项目，而非各种乱七八糟的功能都要囊括进来的多合一大型库，因此它很克制，期间有许多人给我发过一些无关特性的 Pull Request，表示感谢，但全被拒绝了。

对于很多人关心的 下拉刷新、加载更多、HeaderView、FooterView、Diff 这些功能特性，其实都不应该是 **MultiType** 的范畴，**MultiType** 的分内之事是做类型、事件与 View 的分发、连接工作，其余无关的需求，都是可以在 **MultiType** 外部完成，或者通过继承 进行自行封装和拓展，而作为一个基础、公共类库，我想它是不应该包含这些内容。

但很多新手可能并不习惯代码分工、模块化，因此在此我有必要对这几个点简单示范下如何在 **MultiType** 之外去实现：

- **下拉刷新：**

  对于下拉刷新，`Android` 官方提供了 `support.v4` `SwipeRefreshLayout`，在 `Activity` 层面，可以拿到 `SwipeRefreshLayout` 并 `setOnRefreshListener`.

- **加载更多：**

  `RecyclerView` 提供了 `addOnScrollListener` 滚动位置变化监听，要实现加载更多，只要监听并检测列表是否滚动到底部即可，有多种方式，鉴于 `LayoutManager` 本应该只做布局相关的事务，因此我们推荐直接在 `OnScrollListener` 层面进行判断。提供一个简单版 `OnScrollListener` 继承类：

  ```java
  public abstract class OnLoadMoreListener extends RecyclerView.OnScrollListener {

      private LinearLayoutManager layoutManager;
      private int itemCount, lastPosition, lastItemCount;

      public abstract void onLoadMore();

      @Override
      public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
          if (recyclerView.getLayoutManager() instanceof LinearLayoutManager) {
              layoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();

              itemCount = layoutManager.getItemCount();
              lastPosition = layoutManager.findLastCompletelyVisibleItemPosition();
          } else {
              Log.e("OnLoadMoreListener", "The OnLoadMoreListener only support LinearLayoutManager");
              return;
          }

          if (lastItemCount != itemCount && lastPosition == itemCount - 1) {
              lastItemCount = itemCount;
              this.onLoadMore();
          }
      }
  }
  ```

- **获取数据后做 Diff 更新：**

  可以在 `Activity` 中进行 Diff，或者继承 `MultiTypeAdapter` 提供接收数据方法，在方法中进行 Diff. **MultiType** 不提供内置 Diff 方案，不然需要依赖 v4 包，并且这也不应该属于它的范畴。

- **HeaderView、FooterView**

  **MultiType** 其实本身就支持 `HeaderView`、`FooterView`，只要创建一个 `Header.class` - `HeaderViewProvider` 和 `Footer.class` - `FooterViewProvider` 即可，然后把 `new Header()` 添加到 `items` 第一个位置，把 `new Footer()` 添加到 `items` 最后一个位置。需要注意的是，如果使用了 Footer View，在底部插入数据的时候，需要添加到 `最后位置 - 1`，即倒二个位置，或者把 `Footer` remove 掉，再添加数据，最后再插入一个新的 `Footer`.
  
## 实现 RecyclerView 嵌套横向 RecyclerView

**MultiType** 天生就适合实现类似 Google Play 或 iOS App Store 那样复杂的首页列表，这种页面通常会在垂直列表中嵌套横向列表，其实横向列表我们完全可以把它视为一种 `Item` 类型，这个 item 持有一个列表数据和当前横向列表滑动到的位置，类似这样：

```java
public class PostList implements Item {

    public final List<Post> posts;
    public int currentPosition;

    public PostList(@NonNull List<Post> posts) {this.posts = posts;}
}
```

对应的 `HorizontalItemViewProvider` 类似这样：

```java
public class HorizontalItemViewProvider
    extends ItemViewProvider<PostList, HorizontalItemViewProvider.ViewHolder> {

    @NonNull @Override
    protected ViewHolder onCreateViewHolder(
        @NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        /* item_horizontal_list 就是一个只有 RecyclerView 的布局 */
        View view = inflater.inflate(R.layout.item_horizontal_list, parent, false);
        return new ViewHolder(view);
    }

    @Override
    protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull PostList postList) {
        holder.setPosts(postList.posts);
    }

    static class ViewHolder extends RecyclerView.ViewHolder {

        private RecyclerView recyclerView;
        private PostsAdapter adapter;

        private ViewHolder(@NonNull View itemView) {
            super(itemView);
            recyclerView = (RecyclerView) itemView.findViewById(R.id.post_list);
            LinearLayoutManager layoutManager = new LinearLayoutManager(itemView.getContext());
            layoutManager.setOrientation(LinearLayoutManager.HORIZONTAL);
            recyclerView.setLayoutManager(layoutManager);
            /* adapter 只负责灌输、适配数据，布局交给 LayoutManager，可复用 */
            adapter = new PostsAdapter();
            recyclerView.setAdapter(adapter);
            /* 在此设置横向滑动监听器，用于记录和恢复当前滑动到的位置，略 */
            ...
        }

        private void setPosts(List<Post> posts) {
            adapter.setPosts(posts);
            adapter.notifyDataSetChanged();
        }
    }
}
```

## 实现线性布局和网格布局混排列表

这个课题其实也不属于 **MultiType** 的范畴，**MultiType** 的职责是做数据类型分发，而不是布局，但鉴于很多复杂页面都会需要线性布局和网格布局混排，我就简单讲一讲，关键在于 `RecyclerView` 的 `LayoutManager`. 虽然是线性和网格混合，但实现起来其实只要一个网格布局 `GridLayoutManager`，如果你查看 `GridLayoutManager` 的官方源码，你会发现它其实继承自 `LinearLayoutManager`. 以下是示例和解释：

```java
public class MultiGridActivity extends MenuBaseActivity {

    private final static int SPAN_COUNT = 5;
    private MultiTypeAdapter adapter;
    private Items items;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_multi_grid);
        items = new Items();
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.list);
        
        final GridLayoutManager layoutManager = new GridLayoutManager(this, SPAN_COUNT);
        
        /* 关键内容：通过 setSpanSizeLookup 来告诉布局，你的 item 占几个横向单位，
           如果你横向有 5 个单位，而你返回当前 item 占用 5 个单位，那么它就会看起来单独占用一行 */
        layoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
            @Override
            public int getSpanSize(int position) {
                return (items.get(position) instanceof Category) ? SPAN_COUNT : 1;
            }
        });
        recyclerView.setLayoutManager(layoutManager);
        
        adapter = new MultiTypeAdapter(items);
        adapter.applyGlobalMultiTypePool();
        adapter.register(Square.class, new SquareViewProvider());

        assertAllRegistered(adapter, items);
        recyclerView.setAdapter(adapter);
        loadData();
    }

    private void loadData() {
        // ...
    }
}
```

## 数据扁平化处理

在一个**垂直** `RecyclerView` 中，`Item` 们都是同级的，没有任何嵌套关系，但我们的数据结构往往存在嵌套关系，比如 `Post` 内部包含了 `Comment`s 数据，或换句话说 `Post` 嵌套了 `Comment`，就像微信朋友圈一样，"动态" 伴随着 "评论"。那么如何把 非扁平化 的数据排布在 扁平 的列表中呢？必然需要一个_数据扁平化处理_的过程，就像 `ListView` 的数据需要一个 `Adapter` 来适配，`Adapter` 就像一个油漏斗，把油引入瓶子中。我们在面对嵌套数据结构的时候，可以采用如下的扁平化处理，关于扁平化这个词，不必太纠结，简单说，就是把嵌套数据都拉出来，摊平，让 `Comment` 和 `Post` 同级，最后把它们都 add 进同一个 `Items` 容器，交给 `MultiTypeAdapter`. 示例：

假设：你的 `Post` 是这样的：

```java
public class Post implements Item {

    public String content;
    public List<Comment> comments; 
}
```

假设：你的 `Comment` 是这样的：

```java
public class Comment implements Item {

    public String content;
}
```

假设：你服务端返回的 JSON 数据是这样的：

```json
[
    {
        "content":"I have released the MultiType v2.2.1", 
        "comments":[
            {"content":"great"},
            {"content":"I love your post!"}
        ]
    }
]
```

那么你的 JSON 转成 Java Bean 之后，你拿到手应该是个 `List<Post> posts` 对象，现在我们写一个扁平化处理的方法：

```java
private List<Item> flattenData(List<Post> posts) {
    final List<Item> items = new ArrayList<>();
    for (Post post : posts) {
        /* 将 post 加进 items，Provider 内部拿到它的时候，
         * 我们无视它的 comments 内容即可 */
        items.add(post);
        /* 紧接着将 comments 拿出来插入进 items，
         * 评论就能正好处于该条 post 下面 */
        items.addAll(post.comments);
    }
    return items;
}
```

最后我们所有的 `posts` 在加入全局 MultiType `Items` 之前，都需要经过扁平化处理：

```java
items.addAll(flattenData(posts));
adapter.notifyDataSetChanged();
```

整个过程其实并不困难，相信大家都已经理解了。

# 更多示例

**MultiType** 的开源项目提供了许多的 sample (示例) 程序，这些示例秉承了一贯的代码清晰、干净的风格，十分易于阅读：
  
- [仿造**微博**的数据结构和二级 ViewProvider](https://github.com/drakeet/MultiType/tree/master/sample/src/main/java/me/drakeet/multitype/sample/weibo)

  这是一个类似微博数据结构的示例，数据两层结构，Item 也是两层结构：一层框架（包含头像用户名等），一层 content view(微博内容)，内容嵌套于框架中。微博的每一条微博 Item 都包含了这样两层嵌套关系，这样做的好处是，你不必每个 Item 都去重复制造一遍外层框架。
  
  或者换一个比喻，就像聊天消息，一条聊天消息也是两层的，一层头像、用户名、聊天气泡框，一层你的文字、图片等。另外，每一种消息都有左边和右边的样式，分别对应别人发来的消息和你发出的消息。如果左边算一种，右边又算一种，就是比较不好的设计了，会导致布局内容重复、冗余，修改操作都要做两遍。最好的方案是让他们视被为同一种类型，然后在 Item 框层次进行左右边判断和框架相关数据绑定。
  
  我提供的这个二级 `ViewProvider` 示例便是这样的两层结构。它能够让你每次新增加一个类型，只要实现内容即可，框不应该重复实现。
  
  如果再不明白，或许你可以看看我的这个示例中 微博 Item 框的布局：
  
  ![](http://ww2.sinaimg.cn/large/86e2ff85gw1f9brj5gw1jj21e211ane6.jpg)
  
  从我这个 `frame` 布局可以看出来，它内部有一个 `FrameLayout` 作为 `container` 将用于容纳不同的微博内容，而这一层框架则是共同的。
  
  这个例子算高级中的高级，但实际上也是很简单，展示了 **MultiType** 优秀的可拓展能力。完整运行结果展示如下：
 
  <img src="http://ww3.sinaimg.cn/large/86e2ff85jw1f9a7tek74lj21401z414s.jpg" width=270 height=486/> <img src="http://ww1.sinaimg.cn/mw1024/86e2ff85jw1f9a7z4yqlkj21401z4n8r.jpg" width=270 height=486/>
  
  > 注：在示例中我们并没有示范服务端 JSON 数据转为我们定义的 Weibo 对象过程，实际上对于完整链路，这个过程是需要做数据转换，我们需要在 `Weibo` 层加一个 `type` 或 `describe` 字段用于描述微博内容类型，然后再将微博内容的 JSON 文本转为具体微博内容对象交给 Weibo. 
  
- [drakeet/about-page](https://github.com/drakeet/about-page)

  一个 Material Design 的关于页面，核心基于 MultiType，包含了多种 `Item`s，美观，容易使用。

  ![](http://ww2.sinaimg.cn/large/86e2ff85gw1f93gq2tevbj21700pcjyp.jpg)

- [线性和网格布局混排](https://github.com/drakeet/MultiType/tree/master/sample/src/main/java/me/drakeet/multitype/sample/grid)

  使用 `MultiType` 和 `GridLayoutManager` 实现网格和线性混合布局，实现一个选集页面。

  <img src="http://ww1.sinaimg.cn/large/86e2ff85gw1f96yfddvksj21401z479t.jpg" width=270 height=486/>

- [drakeet/TimeMachine](https://github.com/drakeet/TimeMachine)

  TimeMachine 使用了 **MultiType** 来创建一个复杂的聊天页面，页面和需求虽然复杂，但使用 **MultiType** 显得轻松简单。

  <img src="http://ww1.sinaimg.cn/large/86e2ff85gw1f96yg01b5nj20k00zkgng.jpg" width="270" height="486"/> <img src="http://ww3.sinaimg.cn/large/86e2ff85gw1f94i6ysea2j20k00zkmzn.jpg" width="270" height="486"/>

- [类似 Bilibili iOS 端首页](https://github.com/drakeet/MultiType/tree/master/sample/src/main/java/me/drakeet/multitype/sample/bilibili)

  使用 `MultiType` 实现类似 Bilibili iOS 端首页复杂的多类型列表视图，包括嵌套横向 `RecyclerView`.

  <img src="http://ww3.sinaimg.cn/large/86e2ff85gw1f96ygmroy0j21401z4nl9.jpg" width="270" height="486"/>
  
附：一个第三方示例，[采用真实的网络请求数据演示 MultiType 框架的用法 by WanLiLi](https://github.com/WanLiLi/MultiTypeDemo). 点评：看了下，代码不够清爽，但实现效果十分不错。
  
# Q & A

- **Q: 全局类型池的主要作用是什么，能取消全局的使用吗？**

  A: 全局类型池的主要作用是，注册一些能够各个地方复用的类型，可以存一些比如：Line、Space、Header、LoadMoreFooter. 默认情况下，全局类型池是不会生效的，只有你调用 `adapter.applyGlobalMultiTypePool()` 使用全局类型池，它才会被应用，并加入到你当下的局部类型池中。没有调用这一行代码，全局的就不会参入你的局部类型池。也就是说，终归都是局部类型池，只是你确定使用全局的时候，它会把全局的拷贝一份，然后加入到你这个局部类型池中。
  
- **Q: 使用全局类型的话，只能是在 Application 中进行注册吗？**

  A: 不，只是推荐这么做而已。在 `Application` 初始注册，能够确保类型在使用之前就注册好。另外，位置统一固定，有利于寻找代码。不然出了问题，你需要到处寻找是在哪注册了全局类型。注册全局的代码如果分散到各个地方，就不好控制和追寻，因此最好统一一个地方注册。换一个角度来说，注册全局类型的动作存在着约定性，约定的东西、可被破坏的东西，有时会比较不可靠，**因此能够使用局部类型池的情况，最好使用局部类型池。**
  
- **Q: 为什么不全然使用全局类型池？**

  A: **MultiType** 最早的版本是只支持全局类型池的，因为它带来的好处诸多，但随着更多人使用，它的问题也逐渐暴露出来。一，全局类型池的注册容易分散到许多地方，这是无法约束的，会导致代码难以追寻。二，如果使用不当，可能引起内存泄漏问题，我自己是不会写出内存泄漏的代码的，但如果提供了可能性，就有很多人会趟上去。三，为了解决一对多的问题，我想了许多方案，很多几乎写好了，但都被推翻了，后来我发现，这些麻烦，都是因为一开始基于全局类型池引起的，那些方案固然都可以，但会使代码变得复杂，我不喜欢。
  
- **Q: 觉得 MultiType 不够精简，应该怎么做？**

  A: 在前面 "设计思想" 中我们谈到：_MultiType 或许不是使用起来最简单的，但很可能是使用起来最灵活的。_其中的缘由是它高度可定制、可拓展，而不是把一些路封死。作为一个基础类库，简单和灵活需要一个均衡点，过度精简便要以失去灵活性为代价。如果觉得 **MultiType** 不够精简，想将它修改得更加容易使用，我推荐的方式是去继承 `MultiTypeAdapter` 或 `ItemViewProvider`，甚至你可以重新实现一个 `TypePool` 再设置给 `MultiTypeAdapter`. 我们不应该直接到底层去修改、破坏它们。总之，利用开放接口或继承的做法不管对于 **MultiType** 还是其它开源库，都应该是定制的首选。
  
# 感谢

在 **MultiType** 开发维护过程中，很多朋友给了我很多反馈，我也非常乐意于与大家交流，有问必答，因为这是一个难得不错的项目，它比较接近我心中对于一个完美项目的要求：设计精巧，代码干净漂亮。

我向来是不太在意项目的 star 数目的，但热衷于把我的好东西分享给更多人使用，因此在[我的 GitHub](https://github.com/drakeet) 首页我不会把我一些高 star 项目摆出来，而是放一些我觉得代码相对比较好的项目。这是我的动力，我想写一份完美的代码，就像王垠的 40 行一样，达到自觉得天衣无缝、犹如天神衣袖般的优雅，嗯，要是哪天我做到了，我就停止开源，哈哈。

话说回来，这个项目，特别感谢大家的帮忙、反馈，感谢一些朋友的 PR、贡献和推荐，是你们让我觉得开源是一件除了完善自我之外 还充满了意义的一件事情 -- 能够与更多人协同，能够面向更宽广的世界，谢谢大家！以下是感谢名单：

[70kg](https://github.com/70kg)、[zubinxiong](https://github.com/zubinxiong)、[WanLiLi](https://github.com/WanLiLi)、[代码家](https://github.com/daimajia)、[CaMnter](https://github.com/CaMnter)、[android-xiaowei](https://github.com/android-xiaowei)、[burgessjp](https://github.com/burgessjp)、[lixi0912](https://github.com/lixi0912)、[simidaxu](https://github.com/simidaxu)、[咕咚](https://github.com/maoruibin)、[LuckyJayce](https://github.com/LuckyJayce)、[BelongsH](https://github.com/BelongsH)、[tmexcept](https://github.com/tmexcept)、[TellH](https://github.com/TellH)、[Ray Pan](https://github.com/Panl)、[Zack](https://github.com/DearZack)、[Chris](https://github.com/ChrisZou)

# 引用文献

- 《Android 内存泄漏案例和解析》https://drakeet.me/android-leaks
- 《Android 复杂的多类型列表视图新写法：MultiType》https://drakeet.me/multitype
- 《使用 Google AutoValue 自动生成代码》http://tedyin.me/2016/04/11/auto-value/
- **MultiType** 源代码 https://github.com/drakeet/MultiType
