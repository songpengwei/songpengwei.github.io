---
title: Android 学习笔记一：搜索框的实现
updated: 2017-12-02 17:36
---

### 目标
输入关键字，实时显示搜索结果。

### 过程
从[官方文档](https://developer.android.com/guide/topics/search/search-dialog.html)入手，由于初入门，相关术语懂得少，而该文档又非代码级实现，导致没能完整搭起搜索的架子。该文档主要讲了以下几点：
1. 两种实现方式，search dialog和search widget；对于Android 3.0 以后的机器，推荐使用后者，较为灵活。
2. 三个主要组件：
	* 搜索配置（a searchable configuration）
	* 搜索容器（a searchable Activity）（困惑点1）
	* 搜索结构（a search interface)
3. 搜索过程：
	* 接受查询（Receive the query）
	  使用Itent（困惑点2）
	* 查询数据（Search your data）
	* 呈现结果（Present the results）（困惑点3）

我想使用Search Widget方式，主要遇到以下几个困惑点：
1. 如何在`Activity`中触发搜索，就是AppBar右上角的搜索图标如何做出来。
2. 如果使用`Intent`的查询数据，如示例一般，应该不能做到实时匹配输入字符。我猜想应该有listener之类的，但是例子没给。
3. 如何呈现结果，文档建议让`SearchAbleActivity`继承`ListView`来实现，但是具体细节，如怎么接受结果，传递给`ListView`，都没有提。
等于搜索过程的三个环节都没有搞清楚，一脸懵逼。

于是搜索关键词 SearchView action bar，找到一篇帖子：[Implementing SearchView in action bar](https://stackoverflow.com/questions/21585326/implementing-searchview-in-action-bar)，反复琢磨，才弄清楚了以上几个问题。

首先，对于触发搜索，该回答使用的是具有App Bar的Activity作为SearchableActivity，并且在复写onCreateOptionsMenu函数，实例化其参数menu，并且将SearchView作为其一个item。如此一来，SearchableActivity的右上角就会有搜索按钮。相关代码如下：
**res\menu\search.xml:**
```XML
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
        <item android:id="@+id/search_menu"
            android:title="@string/search_hint"
            app:showAsAction="ifRoom|collapseActionView"
            app:actionViewClass="android.widget.SearchView" />

</menu>
```
**SearchableActivity.java**
```Java
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.search, menu);
        this.menu = menu;

        // Get the SearchView and set the searchable configuration
        SearchManager searchManager = (SearchManager) getSystemService(Context.SEARCH_SERVICE);
        SearchView searchView = (SearchView) menu.findItem(R.id.search_menu).getActionView();
        // Assumes current activity is the searchable activity
        ComponentName name = getComponentName();
        searchView.setSearchableInfo(searchManager.getSearchableInfo(name));
        searchView.setIconifiedByDefault(false);

        searchView.setOnQueryTextListener(new SearchView.OnQueryTextListener() {
            @Override
            public boolean onQueryTextSubmit(String s) {
                doMySearch(s);
                return false;
            }

            @Override
            public boolean onQueryTextChange(String s) {
                doMySearch(s);
                return false;
            }
        });

        return true;
    }
```

**AndroidManifest.xml**
```
<activity android:name=".SearchableActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEARCH" />
    </intent-filter>
    <meta-data android:name="android.app.searchable"
        android:resource="@xml/searchable"/>
</activity>
```

**res/xml/searchable.xml**
```
<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
    android:label="@string/app_label"
    android:hint="@string/app_label" >
</searchable>
```
其次是实时匹配查询结果；也是在`onCreateOptionsMenu`函数中，给`SearchView`设置listeners，具体可以见上面代码，但是return true/false暂时有什么区别还没搞清楚。

最后是展示数据；如官方文档所说，利用ListView，具体做法是通过Adapter将数据（比如List<String>）传给利用xml渲染（inflate）的ListView，该答案是用CursorAdaptor（对接数据库数据更合适）。具体代码如下：
**res/layout/item.xml**
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <TextView
    android:id="@+id/item"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
</RelativeLayout>
```

ResultAdapter.java
```Java
public class ResultAdapter extends CursorAdapter {
    private List<String> items;
    private TextView text;

    public ResultAdapter(Context context, Cursor cursor, List<String> items) {
        super(context, cursor, false);
        this.items = items;
    }

    @Override
    public void bindView(View view, Context context, Cursor cursor) {
        text.setText(items.get(cursor.getPosition()));
    }

    @Override
    public View newView(Context context, Cursor cursor, ViewGroup parent) {
        LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View view = inflater.inflate(R.layout.item, parent, false);
        text = view.findViewById(R.id.item);
        return view;
    }
}
```
**SearchableActivity.java**
```Java
private void doMySearch(String query){
    String[] columns = new String[] { "_id", "text" };
    Object[] temp = new Object[] { 0, "default" };

    MatrixCursor cursor = new MatrixCursor(columns);
    for(int i = 0; i < items.size(); i++) {

        temp[0] = i;
        temp[1] = items.get(i);
        cursor.addRow(temp);

    }

    // SearchView
    final SearchView search = (SearchView) menu.findItem(R.id.search_menu).getActionView();
    search.setSuggestionsAdapter(new ResultAdapter(this, cursor, items));
}
```

### 参考资料：
1. Android官方文档，https://developer.android.com/guide/topics/search/search-dialog.html
2. Stack Overflow回答，https://stackoverflow.com/questions/21585326/implementing-searchview-in-action-bar
3. 官方视频，https://www.youtube.com/watch?v=9OWmnYPX1uc









