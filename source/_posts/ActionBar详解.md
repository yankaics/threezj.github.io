title: ActionBar详解
date: 2015-09-29 15:13:06
tags: ["Android"]
---
## 关于ActionBar

ActionBar是个窗体功能来鉴别用户当前在app中的位置，提供给用户一些功能和导航。通过ActionBar可以方便的让系统自动的适配不同尺寸的屏幕。

ActionBar包含如下内容。

![ActionBar](http://ww2.sinaimg.cn/mw690/87dacc16gw1ewj907arxlj20nt0bzjtg.jpg)

1.  App icon
2.  Action item(项目)
3.  Action overflow(放不下的item,放在overflow里)<a id="more"></a>
<!--more-->
## 添加依赖
> *   如果API等级比11低
> `import android.support.v7.app.ActionBar`*   如果API等级比11高或者相等
> `import android.app.ActionBar`

## 加入一个ActionBar

在你的布局里引入一个ActionBar，只需要2步

1.  创建一个活动继承自`ActionBarActivity`（但是现在被`AppCompatActivity`替代，所以继承`AppCompatActivity`即可）
2.  在你的activity声明中加入
`&lt;activity android:theme=&quot;@style/Theme.AppCompat.Light&quot;&gt;`

## 删除ActionBar

你可以在程序运行时隐藏ActionBar，像这样。
```java
ActionBar actionBar = getSupportActionBar();
actionBar.hide();
```
android系统会自动调整布局来填充，隐藏的ActionBar。如果你希望它显示回来，可以使用 `show()`方法。

## 调整图标和标题

可以通过改变activity中的 logo和lable属性来替换图标和标题。

## 添加Action Items

![Action Items](http://ww3.sinaimg.cn/mw690/87dacc16gw1ewjabbeqaij20iw02ddfr.jpg)

一般ActionBar有三个Action按钮和一个overflow按钮。首先你要res/menu/目录下创建一个布局，来设置Action。

比如像这样，创建res/menu/main_activity_actions.xml
```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android" >
    <item android:id="@+id/action_search"
          android:icon="@drawable/ic_action_search"
          android:title="@string/action_search"/>
    <item android:id="@+id/action_compose"
          android:icon="@drawable/ic_action_compose"
          android:title="@string/action_compose" />
</menu>
```
然后当Activity启动，系统填充ActionItem时，会调用 `onCreateOptionsMenu()`方法。通过这个方法来加载一个菜单资源(menu resource)，按照你刚才设置的布局来加载Item。
```java
@Override
    public boolean onCreateOptionsMenu(Menu menu) {
    // Inflate the menu items for use in the action bar
    MenuInflater inflater = getMenuInflater();
    inflater.inflate(R.menu.main_activity_actions, menu);
    return super.onCreateOptionsMenu(menu);
}
```
布局文件中的`showAsAction` 属性可以确定item是在overflow里还是在外面。

```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <item android:id="@+id/action_search"
          android:icon="@drawable/ic_action_search"
          android:title="@string/action_search"
          yourapp:showAsAction="ifRoom"  />
</menu>
```

**注意** showAsAction这个属性 必须要自定义一个名字空间来使用
`xmlns:yourapp=&quot;http://schemas.android.com/apk/res-auto&quot; &gt;`
`yourapp:showAsAction=&quot;ifRoom&quot;  /&gt;`

*   ifRoom 会显示在ActionBar 如果屏幕空间不足，则藏到Overflow里
*   never 永远只在Overflow里显示，而且只显示标题
*   always 总是显示
*   withText 尽可能的显示标题。空间不足则不显示
*   collapseActionView 声明这个View被折叠到按钮。选择了按钮才会打开。配合ifRoom使用

## 监听Action Button的点击事件

一旦用户按下ActionButton，则会回调`onOptionsItemSelected()`方法。
```java
@Override
public boolean onOptionsItemSelected(MenuItem item) {
    // Handle presses on the action bar items
    switch (item.getItemId()) {
        case R.id.action_search:
            openSearch();
            return true;
        case R.id.action_compose:
            composeMessage();
            return true;
        default:
            return super.onOptionsItemSelected(item);
    }
}
```
## 使用Split Action Bar

当你的activity运行到一个狭窄的屏幕上时，这是系统会自动适配你的ActionItem.通过Split Action Bar显示在屏幕底部。如下图。

![Split Action Bar](http://ww4.sinaimg.cn/mw690/87dacc16gw1ewkb0bhog8j20nc0e8abm.jpg)

只要两步你就能启用Split Action Bar

1.  如果你的环境是API14及以上，只要加`uiOptions=&quot;splitActionBarWhenNarrow&quot;`属性到每一个`activity`元素中，或者`&lt;application&gt;`2.  为了支持老的版本，你还需要加一个 `&lt;meta-data&gt;`元素到`activity`中，如下定义。
```xml
<manifest>
    <activity uiOptions="splitActionBarWhenNarrow" ... >
        <meta-data android:name="android.support.UI_OPTIONS"
                   android:value="splitActionBarWhenNarrow" />
    </activity>
</manifest>
```
## 使用向上导航菜单(Navigating Up)

![(Navigating Up](http://ww2.sinaimg.cn/mw690/87dacc16gw1ewk9nzmbbwj206o02o3yc.jpg)

如果窗体A中有系列的item，选择了一个item以后，跳转到窗体B，此时可通过Navigating Up来返回到窗体A.
注意Navigating Up和back键不同。back键的返回顺序是按窗体创建的时间顺序倒序返回。而不是app之间的层次结构。比如说即使你跳转到了窗体C，按了Navigating Up之后还是会跳转到窗体A。

你可以像这样启用up button
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_details);

    ActionBar actionBar = getSupportActionBar();
    actionBar.setDisplayHomeAsUpEnabled(true);
    ...
}
```
然而此时并没有什么用，还需要在activity中注册父活动。也就是说返回的时候会返回到父活动。
API16以上只要直接设置`parentActivityName`属性即可
```xml
<application ... >
    ...
    <!-- The main/home activity (has no parent activity) -->
    <activity
        android:name="com.example.myfirstapp.MainActivity" ...>
        ...
    </activity>
    <!-- A child of the main activity -->
    <activity
        android:name="com.example.myfirstapp.DisplayMessageActivity"
        android:label="@string/title_activity_display_message"
        android:parentActivityName="com.example.myfirstapp.MainActivity" >
        <!-- Parent activity meta-data to support API level 7+ -->
        <meta-data
            android:name="android.support.PARENT_ACTIVITY"
            android:value="com.example.myfirstapp.MainActivity" />
    </activity>
</application>
```
这两步完成 upbutton就会正常工作了。

如果你允许其他App来启动这个activity则要覆写`onOptionsItemSelected()`这个方法
```java
@Override
public boolean onOptionsItemSelected(MenuItem item) {
    switch (item.getItemId()) {
    // Respond to the action bar's Up/Home button
    case android.R.id.home:
        Intent upIntent = NavUtils.getParentActivityIntent(this);
        if (NavUtils.shouldUpRecreateTask(this, upIntent)) {
            // This activity is NOT part of this app's task, so create a new task
            // when navigating up, with a synthesized back stack.
            TaskStackBuilder.create(this)
                    // Add all of this activity's parents to the back stack
                    .addNextIntentWithParentStack(upIntent)
                    // Navigate up to the closest parent
                    .startActivities();
        } else {
            // This activity is part of this app's task, so simply
            // navigate up to the logical parent activity.
            NavUtils.navigateUpTo(this, upIntent);
        }
        return true;
    }
    return super.onOptionsItemSelected(item);
}
```
## 添加Action View

Action View对复杂的动作可提供快速的访问，而不需要改变activity或者fragment，而且不用改变ActionBar。

![Action View](http://ww3.sinaimg.cn/mw690/87dacc16gw1ewkailhmbhj20iw07imxi.jpg)

要定义一个Action View，必须在`actionLayout`或者`actionViewClass`属性中指定布局资源或者widge类。比如，添加SearchView
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <item android:id="@+id/action_search"
          android:title="@string/action_search"
          android:icon="@drawable/ic_action_search"
          yourapp:showAsAction="ifRoom|collapseActionView"
          yourapp:actionViewClass="android.support.v7.widget.SearchView" />
</menu>
```
注意`howAsAction`包含了一个`&quot;collapseActionView&quot;`值，这个选项表示Action View可折叠。
如果你希望配置添加的ActionView，像这样获取
```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main_activity_actions, menu);
    MenuItem searchItem = menu.findItem(R.id.action_search);
    SearchView searchView = (SearchView) MenuItemCompat.getActionView(searchItem);
    // Configure the search info and add any event listeners
    ...
    return super.onCreateOptionsMenu(menu);
}
```
API11及以上,可这样获取

`menu.findItem(R.id.action_search).getActionView()`

## 处理可折叠的Action View

Action View展开后，按back键或者upbutton可折叠。覆写`OnActionExpandListener`中的`onMenuItemActionCollapse`和`onMenuItemActionExpand`可分别监听展开和折叠。
```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.options, menu);
    MenuItem menuItem = menu.findItem(R.id.actionItem);
    ...

    // When using the support library, the setOnActionExpandListener() method is
    // static and accepts the MenuItem object as an argument
    MenuItemCompat.setOnActionExpandListener(menuItem, new OnActionExpandListener() {
        @Override
        public boolean onMenuItemActionCollapse(MenuItem item) {
            // Do something when collapsed
            return true;  // Return true to collapse action view
        }

        @Override
        public boolean onMenuItemActionExpand(MenuItem item) {
            // Do something when expanded
            return true;  // Return true to expand action view
        }
    });
}
```
## Action Provider

和ActionView差不多，Action Provider同样可以把一个action按钮替换成自定义的布局。然而不同的是，Action Provider能完全控制动作的行为和展示一个子菜单。
要定义一个Action Provider，需在`menu`的`item`标签中指定`actionViewClass`属性为`ActionProvider`
你可以建立自定义的Action Provider通过继承`ActionProvider`类，一般来说，android会提供一些预建的Action Provider，比如`ShareActionProvider`，如下图。

![ActionProvider](http://i.imgur.com/Whp0ugg.png)

因为每一个ActionProvider有自己明确的动作，所以不必监听`onOptionsItemSelected()`，如果你非要监听也是可以的,但注意要`return false`,以便ActionProvider，回调`onPerformDefaultAction`方法来执行动作。

然而，如果你的ActionProvider会启用子菜单，则不必监听`onOptionsItemSelected()`。

## 使用ShareActionProvider

第一步，上文中已经说了，在`menu`的`item`标签中指定`actionViewClass`属性为`ActionProvider`。
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <item android:id="@+id/action_share"
          android:title="@string/share"
          yourapp:showAsAction="ifRoom"
          yourapp:actionProviderClass="android.support.v7.widget.ShareActionProvider"
          />
    ...
</menu>
```
第二步 定义你希望用来分享的`Intent`
```java
private Intent getDefaultIntent() {
    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.setType("image/*");
    return intent;
}
```
第三步 覆写`onCreateOptionsMenu`方法,**注意，当Intent发生改变时一定要重新执行一次`setShareIntent()`.**
```java
private ShareActionProvider mShareActionProvider;

@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main_activity_actions, menu);

    // Set up ShareActionProvider's default share intent
    MenuItem shareItem = menu.findItem(R.id.action_share);
    mShareActionProvider = (ShareActionProvider)
            MenuItemCompat.getActionProvider(shareItem);
    mShareActionProvider.setShareIntent(getDefaultIntent());

    return super.onCreateOptionsMenu(menu);
}
```
ShareActionProvider会按照使用的频繁程度来排序菜单。

## 创建自定义Acion Provider

你也可以自定义Acion Provider通过继承自ActionProvider类且实现一些适当的回调方法。如下是一些比较重要的方法。

1.  `ActionProvider(Context context)`

     构造函数不多说。

2.  `public View onCreateActionView (MenuItem forItem)`

     通过这个方法来定义ActionView，像这样。

```java
public View onCreateActionView(MenuItem forItem) {
    // Inflate the action view to be shown on the action bar.
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    View view = layoutInflater.inflate(R.layout.action_provider, null);
    ImageButton button = (ImageButton) view.findViewById(R.id.button);
    button.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            // Do something...
        }
    });
    return view;
}
```
3.  `public boolean onPerformDefaultAction ()`

     当Action overflow中的菜单项被选中时会调用，或者action provider希望执行默认动作通过菜单项。

## 添加Navigation Tabs

用户可以方便地使用Navigation Tabs来浏览不同的View。更重要的是，Tabs能完美适配不同尺寸的屏幕，像这样。

![Tabs wide](http://i.imgur.com/kaspxl7.png)

![Tabs narrows](http://i.imgur.com/OquLbbu.png)

实现Navigation Tabs需要两步。

1.  实现`ActionBar.TabListener`接口，这个接口为Tabs事件提供一些回调方法，比如说滑动标签的时候。
2.  如果你要添加一个标签，必须实例化标签，并且调用`setTabListener()`,和tab的标题`setText()`
3.把Tab加到ActionBar中，通过`addTab()`

下面是一个关于实现`ActionBar.TabListener`接口的Demo
```java
public static class TabListener<T extends Fragment> implements ActionBar.TabListener {
    private Fragment mFragment;
    private final Activity mActivity;
    private final String mTag;
    private final Class<T> mClass;

    /** Constructor used each time a new tab is created.
      * @param activity  The host Activity, used to instantiate the fragment
      * @param tag  The identifier tag for the fragment
      * @param clz  The fragment's Class, used to instantiate the fragment
      */
    public TabListener(Activity activity, String tag, Class<T> clz) {
        mActivity = activity;
        mTag = tag;
        mClass = clz;
    }

    /* The following are each of the ActionBar.TabListener callbacks */

    public void onTabSelected(Tab tab, FragmentTransaction ft) {
        // Check if the fragment is already initialized
        if (mFragment == null) {
            // If not, instantiate and add it to the activity
            mFragment = Fragment.instantiate(mActivity, mClass.getName());
            ft.add(android.R.id.content, mFragment, mTag);
        } else {
            // If it exists, simply attach it in order to show it
            ft.attach(mFragment);
        }
    }

    public void onTabUnselected(Tab tab, FragmentTransaction ft) {
        if (mFragment != null) {
            // Detach the fragment, because another one is being attached
            ft.detach(mFragment);
        }
    }

    public void onTabReselected(Tab tab, FragmentTransaction ft) {
        // User selected the already selected tab. Usually do nothing.
    }
}
```

需要弄明白的是`android.R.id.content`的是意思。

> ‘android.R.id.content’ gives you the root element of a view,without having to
>  know its actual name/type/ID. Check out Get root view from current activity

可以说`android.R.id.content`是一个`Viewgroup`包含`View`的容器，可表示所有`View`

下面是添加Tabs的Demo
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // Notice that setContentView() is not used, because we use the root
    // android.R.id.content as the container for each fragment

    // setup action bar for tabs
    ActionBar actionBar = getSupportActionBar();
    actionBar.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);
    actionBar.setDisplayShowTitleEnabled(false);

    Tab tab = actionBar.newTab()
                       .setText(R.string.artist)
                       .setTabListener(new TabListener<ArtistFragment>(
                               this, "artist", ArtistFragment.class));
    actionBar.addTab(tab);

    tab = actionBar.newTab()
                   .setText(R.string.album)
                   .setTabListener(new TabListener<AlbumFragment>(
                           this, "album", AlbumFragment.class));
    actionBar.addTab(tab);
}
```

## 添加 Drop-down Navigation

ActionBar提供的另外一种导航模式Drop-down Navigation，也被称作Spinner。

![Drop-down Navigation](http://i.imgur.com/qDlNKXv.png)

使用Drop-down Navigation的步骤和Tabs大同小异

1.  创建`SpinnerAdapter`
```java
    SpinnerAdapter mSpinnerAdapter = ArrayAdapter.createFromResource(this,
        R.array.action_list, android.R.layout.simple_spinner_dropdown_item);
```
2.  实现`ActionBar.OnNavigationListener`接口
```java
    mOnNavigationListener = new OnNavigationListener() {
        // Get the same strings provided for the drop-down's ArrayAdapter
        String[] strings = getResources().getStringArray(R.array.action_list);

        @Override
        public boolean onNavigationItemSelected(int position, long itemId) {
        // Create new fragment from our own Fragment class
        ListContentFragment newFragment = new ListContentFragment();
        FragmentTransaction ft = getSupportFragmentManager().beginTransaction();

        // Replace whatever is in the fragment container with this fragment
        // and give the fragment a tag name equal to the string at the position
        // selected
        ft.replace(R.id.fragment_container, newFragment, strings[position]);

        // Apply changes
        ft.commit();
        return true;
        }
    };
```

3.  调用`setNavigationMode(NAVIGATION_MODE_LIST)`方法。

4.调用`setListNavigationCallbacks()`方法

`actionBar.setListNavigationCallbacks(mSpinnerAdapter, mNavigationCallback);`

本文主要内容来自Android Doc，我自己进行了加工理解。
