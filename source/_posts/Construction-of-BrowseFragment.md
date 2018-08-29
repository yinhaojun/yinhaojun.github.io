---
title: Leanback实现TV开发教程之BrowseFragment
date: 2018-08-20 22:26:58
tags:
- Android
- Leanback
---

上一章我们介绍了在Android Studio上如何使用Leanback进行TV开发应用的一些介绍和基础知识，本章我们实现如何将Header和可选的对象集合（卡片）进行关联。在实现该功能之前，我们先来了解下BrowseFragment的结构和使用方法。

这里我们使用Android TV的示例程序进行介绍，当你启动应用的时候，内容是以网格的结构对齐的。左侧的每个header标题都有一个内容行，他们是1对1的关系，这种组合我们称之为“ListRow”。BrowseFragment主体就是一组ListRow的集合，本文我们将使用RowsAdapter来表示。

<!-- more -->

# 概述

下图中，圆角蓝色框所包含的内容就是前文提到的ListRow，而蓝色方框包裹了一组ListRow，我们称之为RowsAdapter。

![RowsAdapter1](RowsAdapter1-800x451.png)

我们接下来具体看看ListRow，这种结构由ArrayObjectAdapter（本文我们称之为RowAdapter）进行关联，维护了一组对象（本文我们称之为CardInfo或者Item）。

这个CardInfo可以是任意的对象，我们稍后详细解释CardInfo并介绍如何利用Presenter来显示CardInfo。

![ListRow1](ListRow1-800x441.png)

总结一下：

```
ArrayObjectAdapter (RowsAdapter) ← A set of ListRow
 ListRow = HeaderItem + ArrayObjectAdapter (RowAdapter)
 ArrayObjectAdapter (RowAdapter) ← A set of Object (CardInfo/Item) 
```

# 具体介绍

## Presenter

Presenter定义Card的设计与如何显示，由于它是一个抽象类，因此为了要为程序设计合适的UI，你必须继承该类。

当你继承Presenter的时候，你至少要重写以下三个方法：

```
1. onCreateViewHolder(ViewGroup parent)
2. onBindViewHolder(ViewHolder viewHolder, Object cardInfo/item)
3. onUnbindViewHolder(ViewHolder viewHolder)
```

如果想了解更多关于这些方法的信息，建议你参考Presenter的源代码。ViewHolder是Presenter的内部类，该类绑定了View的引用，你可以在特定方法中通过ViewHolder访问View。

*注意：其实是对Recyclerview进行了封装，这么说是不是有种豁然开朗的感觉？*

## HeadersFragment、RowsFragment(GridItemPresenter)

在这个示例中，Object(CardInfo/item)是String字符串类型，ViewHolder绑定TextView来显示该字符串。onCreateViewHolder()方法定义view的布局；onBindViewHolder()方法的参数包含了onCreateViewHolder方法中创建的viewHolder和对应的卡片信息，在本例中就是String字符串。

```
   private class GridItemPresenter extends Presenter {
        @Override
        public ViewHolder onCreateViewHolder(ViewGroup parent) {
            TextView view = new TextView(parent.getContext());
            view.setLayoutParams(new ViewGroup.LayoutParams(GRID_ITEM_WIDTH, GRID_ITEM_HEIGHT));
            view.setFocusable(true);
            view.setFocusableInTouchMode(true);
            view.setBackgroundColor(getResources().getColor(R.color.default_background));
            view.setTextColor(Color.WHITE);
            view.setGravity(Gravity.CENTER);
            return new ViewHolder(view);
        }

        @Override
        public void onBindViewHolder(ViewHolder viewHolder, Object item) {
            ((TextView) viewHolder.view).setText((String) item);
        }

        @Override
        public void onUnbindViewHolder(ViewHolder viewHolder) {

        }
    }
}
```

定义完GridItemPresenter之后，你只需要在activity开始的时候设置RowsAdapter，你也可以在MainFragment的onActivityCreated()方法中去设置RowsAdapter。

```
 @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        Log.i(TAG, "onActivityCreated");
        super.onActivityCreated(savedInstanceState);

        setupUIElements();

        loadRows();
    }

    ...

    private void loadRows() {
        mRowsAdapter = new ArrayObjectAdapter(new ListRowPresenter());

        /* GridItemPresenter */
        HeaderItem gridItemPresenterHeader = new HeaderItem(0, "GridItemPresenter");

        GridItemPresenter mGridPresenter = new GridItemPresenter();
        ArrayObjectAdapter gridRowAdapter = new ArrayObjectAdapter(mGridPresenter);
        gridRowAdapter.add("ITEM 1");
        gridRowAdapter.add("ITEM 2");
        gridRowAdapter.add("ITEM 3");
        mRowsAdapter.add(new ListRow(gridItemPresenterHeader, gridRowAdapter));

        /* set */
        setAdapter(mRowsAdapter);
    }
```

以下是MainFragment的完整代码：

```
package com.corochann.androidtvapptutorial;

import android.graphics.Color;
import android.os.Bundle;
import android.support.v17.leanback.app.BrowseFragment;
import android.support.v17.leanback.widget.ArrayObjectAdapter;
import android.support.v17.leanback.widget.HeaderItem;
import android.support.v17.leanback.widget.ListRow;
import android.support.v17.leanback.widget.ListRowPresenter;
import android.support.v17.leanback.widget.Presenter;
import android.util.Log;
import android.view.Gravity;
import android.view.ViewGroup;
import android.widget.TextView;

/**
 * Created by corochann on 2015/06/28.
 */
public class MainFragment extends BrowseFragment {
    private static final String TAG = MainFragment.class.getSimpleName();

    private ArrayObjectAdapter mRowsAdapter;
    private static final int GRID_ITEM_WIDTH = 300;
    private static final int GRID_ITEM_HEIGHT = 200;

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        Log.i(TAG, "onActivityCreated");
        super.onActivityCreated(savedInstanceState);

        setupUIElements();

        loadRows();
    }

    private void setupUIElements() {
        // setBadgeDrawable(getActivity().getResources().getDrawable(R.drawable.videos_by_google_banner));
        setTitle("Hello Android TV!"); // Badge, when set, takes precedent
        // over title
        setHeadersState(HEADERS_ENABLED);
        setHeadersTransitionOnBackEnabled(true);

        // set fastLane (or headers) background color
        setBrandColor(getResources().getColor(R.color.fastlane_background));
        // set search icon color
        setSearchAffordanceColor(getResources().getColor(R.color.search_opaque));
    }

    private void loadRows() {
        mRowsAdapter = new ArrayObjectAdapter(new ListRowPresenter());

        /* GridItemPresenter */
        HeaderItem gridItemPresenterHeader = new HeaderItem(0, "GridItemPresenter");

        GridItemPresenter mGridPresenter = new GridItemPresenter();
        ArrayObjectAdapter gridRowAdapter = new ArrayObjectAdapter(mGridPresenter);
        gridRowAdapter.add("ITEM 1");
        gridRowAdapter.add("ITEM 2");
        gridRowAdapter.add("ITEM 3");
        mRowsAdapter.add(new ListRow(gridItemPresenterHeader, gridRowAdapter));

        /* set */
        setAdapter(mRowsAdapter);
    }

    private class GridItemPresenter extends Presenter {
        @Override
        public ViewHolder onCreateViewHolder(ViewGroup parent) {
            TextView view = new TextView(parent.getContext());
            view.setLayoutParams(new ViewGroup.LayoutParams(GRID_ITEM_WIDTH, GRID_ITEM_HEIGHT));
            view.setFocusable(true);
            view.setFocusableInTouchMode(true);
            view.setBackgroundColor(getResources().getColor(R.color.default_background));
            view.setTextColor(Color.WHITE);
            view.setGravity(Gravity.CENTER);
            return new ViewHolder(view);
        }

        @Override
        public void onBindViewHolder(ViewHolder viewHolder, Object item) {
            ((TextView) viewHolder.view).setText((String) item);
        }

        @Override
        public void onUnbindViewHolder(ViewHolder viewHolder) {

        }
    }
}
```

在colors.xml中添加缺失的颜色：

```
    <color name="default_background">#3d3d3d</color>
```



## 编译并运行

这样我们就实现了Header和Contents的关联，效果如下：

![](GridItemPresenter1-800x450.png)

![](GridItemPresenter2-800x450.png)

*注意：我们仅仅是定义了Presenter，加载数据。至于其他特性，如item会变大当被选中的时候，这些特性是由Leanbak Sdk已经实现好的。*





















