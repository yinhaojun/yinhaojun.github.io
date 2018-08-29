---
title: how to use ImageCardView
date: 2018-08-29 21:00:20
tags:
- Android
- Leanback
---

# 前言

上篇文章，我们介绍了**GridItemPresenter**，关联的类主要有：

- Presenter：**GridItemPresenter**
- ViewHolder's view：**TextView**
- CardInfo/Item：**String**

本篇文章我们主要讲解另外一种Presenter，

- Presenter：**CardPresenter**
- ViewHolder's view：**ImageCardView**
- CardInfo/Item：**Movie**

个人认为Presenter还是很好理解的，就是定义Item的布局以及内容的显示。因此，真正需要核心内容主要还是Item如何布局与显示。本文我们主要讲解***ImageCardView***，该类是由Leanback提供，主要是包含了图片、标题和内容的卡片式布局。

<!-- more -->

# ImageCardView基础

ImageCardView是BaseCardView的子类，所以阅读BaseCardView的源码是个不错的选择。以下是对BaseCardView的解释：

> android.support.v17.leanback.widget  public class BaseCardView  extends android.widget.FrameLayout A card style layout that responds to certain state changes. It arranges its children in a vertical column, with different regions becoming  visible at different times.  A BaseCardView will draw its children  based on its type, the region visibilities of the child types, and the  state of the widget. A child may be marked as belonging to one of three  regions: main, info, or extra. The main region is always visible, while  the info and extra regions can be set to display based on the activated  or selected state of the View. The card states are set by calling  setActivated and setSelected.

BaseCardView本身并不提供特定的布局实现，因此你可以通过定义继承BaseCardView的子类来实现特定的布局控件。ImageCardView是SDK中唯一提供的继承自BaseCardView的子类，现在让我们具体来学习下如何使用它吧！！！

本例主要就是简单的显示电影信息，包括电影海报、标题和简介；主要新建的类有：

| 序号 | 名称          | 备注                     |
| ---- | ------------- | ------------------------ |
| 1    | CardPresenter | 决定Item的布局和内容显示 |
| 2    | Movie         | 电影信息                 |
| 3    | Utils         | 工具类                   |



## 添加CardPresenter

选中项目包名，然后右键->New->class->CardPresenter。

该类继承自Presenter，通过继承父类的Presenter.ViewHolder实现自己的ViewHolder，该ViewHolder持有一个ImageCardView的对象，用来显示具体的电影信息。

```
public class CardPresenter extends Presenter {
 
    private static final String TAG = CardPresenter.class.getSimpleName();
 
    private static Context mContext;
    private static int CARD_WIDTH = 313;
    private static int CARD_HEIGHT = 176;
 
    static class ViewHolder extends Presenter.ViewHolder {
        private Movie mMovie;
        private ImageCardView mCardView;
        private Drawable mDefaultCardImage;
 
        public ViewHolder(View view) {
            super(view);
            mCardView = (ImageCardView) view;
            //movie是默认的图片，需要用户自己添加到drawable中
            mDefaultCardImage = mContext.getResources().getDrawable(R.drawable.movie);
        }
 
        public void setMovie(Movie m) {
            mMovie = m;
        }
 
        public Movie getMovie() {
            return mMovie;
        }
 
        public ImageCardView getCardView() {
            return mCardView;
        }
 
        public Drawable getDefaultCardImage() {
            return mDefaultCardImage;
        }
 
    }
 
    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent) {
        Log.d(TAG, "onCreateViewHolder");
        mContext = parent.getContext();
 
        ImageCardView cardView = new ImageCardView(mContext);
        cardView.setFocusable(true);
        cardView.setFocusableInTouchMode(true);
        cardView.setBackgroundColor(mContext.getResources().getColor(R.color.fastlane_background));
        return new ViewHolder(cardView);
    }
 
    @Override
    public void onBindViewHolder(Presenter.ViewHolder viewHolder, Object item) {
        Movie movie = (Movie) item;
        ((ViewHolder) viewHolder).setMovie(movie);
 
        Log.d(TAG, "onBindViewHolder");
        ((ViewHolder) viewHolder).mCardView.setTitleText(movie.getTitle());
        ((ViewHolder) viewHolder).mCardView.setContentText(movie.getStudio());
        ((ViewHolder) viewHolder).mCardView.setMainImageDimensions(CARD_WIDTH, CARD_HEIGHT);
        ((ViewHolder) viewHolder).mCardView.setMainImage(((ViewHolder) viewHolder).getDefaultCardImage());
    }
 
    @Override
    public void onUnbindViewHolder(Presenter.ViewHolder viewHolder) {
        Log.d(TAG, "onUnbindViewHolder");
    }
 
    @Override
    public void onViewAttachedToWindow(Presenter.ViewHolder viewHolder) {
        // TO DO
    }
}
```

## 添加Movie

**Movie**类其实就是具体的数据源，目前主要包含电影的标题、海报和内容三个属性，后续还会进行扩展。

```

```

准备好数据模型类（Movie）和Presenter（CardPresenter）后，我们就可以显示Movie信息了。

```
private void loadRows() {
        ArrayObjectAdapter mRowsAdapter = new ArrayObjectAdapter(new ListRowPresenter()); 
        /* CardPresenter */
        HeaderItem cardPresenterHeader = new HeaderItem(0, "CardPresenter");
        CardPresenter cardPresenter = new CardPresenter();
        ArrayObjectAdapter cardRowAdapter = new ArrayObjectAdapter(cardPresenter);
 
        for(int i=0; i<10; i++) {
            Movie movie = new Movie();
            movie.setTitle("title" + i);
            movie.setStudio("studio" + i);
            cardRowAdapter.add(movie);
        }
        mRowsAdapter.add(new ListRow(cardPresenterHeader, cardRowAdapter));
 		setAdapter(mRowsAdapter)
    }
```

## 查看效果

编译运行，结果如下：

![first.png](first.png)

![second.png](second.png)

# Picasso显示电影海报

我们会发现上面的结果没有显示电影海报信息，而是显示了一张默认图片。一般来说电影信息都是从后台数据取出来的，包括海报、标题、描述等等，这一节主要介绍如何使用Picasso库来在线加载电影海报。

**picasso**是一个在线加载图片的开源库，类似的还有Glide等，具体的介绍可以参考这里：

- [Picasso](http://square.github.io/picasso/)
- [Android SDK: Working with Picasso](http://code.tutsplus.com/tutorials/android-sdk-working-with-picasso--cms-22149)
- [How to Use Picasso Image Loader Library in Android](http://javatechig.com/android/how-to-use-picasso-library-in-android)

## 添加picasso

在build.gradle中添加picasso库，代码如下：

```

```

## 在Movie添加海报字段

```
private String cardImageUrl;
 
public String getCardImageUrl() {
    return cardImageUrl;
}
 
public void setCardImageUrl(String cardImageUrl) {
    this.cardImageUrl = cardImageUrl;
}
```

我们也可以定义个getImageURI 的方法，将Url转成URI格式。

```

```



## 新增PicassoImageCardViewTarget 

PicassoImageCardViewTarget继承自Picasso的Target类，传入ImageCardView，Picasso会将加载结果设置到该ImageCardView中。

```
public static class PicassoImageCardViewTarget implements Target {
        private ImageCardView mImageCardView;
 
        public PicassoImageCardViewTarget(ImageCardView imageCardView) {
            mImageCardView = imageCardView;
        }
 
        @Override
        public void onBitmapLoaded(Bitmap bitmap, Picasso.LoadedFrom loadedFrom) {
            Drawable bitmapDrawable = new BitmapDrawable(mContext.getResources(), bitmap);
            mImageCardView.setMainImage(bitmapDrawable);
        }
 
        @Override
    	public void onBitmapFailed(Exception e, Drawable errorDrawable) {
        	mImageCardView.setMainImage(errorDrawable);
    	}
 
        @Override
        public void onPrepareLoad(Drawable drawable) {
            // Do nothing, default_background manager has its own transitions
        }
    }
```

## 修改ViewHolder

```
 public ViewHolder(View view) {
            super(view);
            mCardView = (ImageCardView) view;
            mImageCardViewTarget = new PicassoImageCardViewTarget(mCardView);
            mDefaultCardImage = mContext.getResources().getDrawable(R.drawable.movie);
        }
 
           ...
 
        protected void updateCardViewImage(URI uri) {
            Picasso.with(mContext)
                    .load(uri.toString())
                    .resize(Utils.convertDpToPixel(mContext, CARD_WIDTH),
                            Utils.convertDpToPixel(mContext, CARD_HEIGHT))
                    .error(mDefaultCardImage)
                    .into(mImageCardViewTarget);
        }
    }
 
 
    ...
 
    @Override
    public void onBindViewHolder(Presenter.ViewHolder viewHolder, Object item) {
        Movie movie = (Movie) item;
        ((ViewHolder) viewHolder).setMovie(movie);
 
        Log.d(TAG, "onBindViewHolder");
        if (movie.getCardImageUrl() != null) {
            ((ViewHolder) viewHolder).mCardView.setTitleText(movie.getTitle());
            ((ViewHolder) viewHolder).mCardView.setContentText(movie.getStudio());
            ((ViewHolder) viewHolder).mCardView.setMainImageDimensions(CARD_WIDTH, CARD_HEIGHT);
            ((ViewHolder) viewHolder).updateCardViewImage(movie.getCardImageURI());
            //((ViewHolder) viewHolder).mCardView.setMainImage(((ViewHolder) viewHolder).getDefaultCardImage());
        }
    }
```

Movie设置海报地址

```
ArrayObjectAdapter mRowsAdapter = new ArrayObjectAdapter(new ListRowPresenter());
        /* CardPresenter */
        HeaderItem cardPresenterHeader = new HeaderItem(0, "本地背景");
        ArrayObjectAdapter cardRowAdapter = new ArrayObjectAdapter(new CardPresenter());
        String[] imgs = {
        	   "https://extraimage.net/images/2018/08/27/57861ec7718febe65615debfdc2524b9.jpg",
                "https://extraimage.net/images/2018/08/28/33ba4d8e91c9f9891a3bffce06960919.jpg",
                "https://extraimage.net/images/2018/08/26/72b140200fc0622e9df270fca7f8d83b.jpg",
                "https://extraimage.net/images/2018/08/19/fd471762a6dc439334f9ca47fda75190.jpg",
                "http://images2.imagebam.com/c0/45/05/765e68930277054.jpg",
                "http://images2.imagebam.com/96/af/53/a18b21937430324.jpg",
                "https://www.z4a.net/images/2018/07/19/065f3550a6973cd7a.jpg",
                "http://images2.imagebam.com/2d/64/d3/b69a83936061624.jpg",
                "http://images2.imagebam.com/f0/38/d1/2fa611932895844.jpg",
                "http://www.imageto.org/images/tUAZ1.jpg"};
        for (int i = 0; i < 10; i++) {
            Movie movie = new Movie();
            movie.setTitle("title" + i);
            movie.setStudio("studio" + i);
            movie.setCardImageUrl(imgs[i]);
            cardRowAdapter.add(movie);
        }
        mRowsAdapter.add(new ListRow(cardPresenterHeader, cardRowAdapter));
```

## 添加网络权限

因为是加载网络图片，所以务必在AndroidManifest.xml中添加网络权限。

```
 <uses-permission android:name="android.permission.INTERNET" />
```

## 查看效果

编译运行，效果如下：

![third.png](third.png)

