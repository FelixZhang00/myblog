title: Android ListView头部视差控件
tag: 自定义控件
categotry: Android
---
## 效果展示
<img src="http://7viip0.com1.z0.glb.clouddn.com/2015-04-25-头部视差.gif" />

## 代码实现

### 静态布局，为ListView增加头部的View
```java

	 mListView = (ParallaxListView) findViewById(R.id.listview);
        mHeadView = View.inflate(this, R.layout.head, null);            //异步解析xml中的布局
        mListView.addHeaderView(mHeadView);
```


### 自定义ListView，重写overScrollBy方法

	
	overScrollBy方法会在ListView滑动到顶部和底部时会调用。
	获取头部控件的大小需要在布局解析完成后才能知道，否则得到的将是0，
	通过设置监听器mHeadView.getViewTreeObserver().addOnGlobalLayoutListener，
	当布局文件解析完成后，会调用此监听器中的回调方法，这是就可以将头部控件传入自定义的ListView中了
	
```java

	public class ParallaxListView extends ListView {

    public ParallaxListView(Context context) {
        super(context);
    }

    public ParallaxListView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public ParallaxListView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    private ImageView parallaxImageView;
    private int maxHeight;
    private int originalHeight;

    public void setParallaxImageView(ImageView parallaxImageView) {
        this.parallaxImageView = parallaxImageView;
        originalHeight = parallaxImageView.getHeight();

        //获得imagview上图片的原始高度，即为imageview的最高度
        maxHeight = parallaxImageView.getDrawable().getIntrinsicHeight();

    }


    /**
     * 当ListView被滑动到顶部和底部时会调用此方法
     *
     * @param deltaY         y方向滑动的距离。 正：底部到头;负：顶部到头
     * @param maxOverScrollY 到头后，最大可滚动的范围
     * @param isTouchEvent   是否是触摸滑动。true:手指拖动到头;false:惯性滑动到头。
     * @return
     */
    @Override
    protected boolean overScrollBy(int deltaX, int deltaY, int scrollX, int scrollY, int scrollRangeX, int scrollRangeY, int maxOverScrollX, int maxOverScrollY, boolean isTouchEvent) {
        if (deltaY < 0 && isTouchEvent) {
            int newHeight = parallaxImageView.getHeight() - deltaY / 3;     //新的高度增加与手指拖动的距离不成正比，
            //达到拖动吃力的效果
            if (newHeight > maxHeight) newHeight = maxHeight;
            if (parallaxImageView != null) {
                parallaxImageView.getLayoutParams().height = newHeight;
                parallaxImageView.requestLayout();                              //当完成高度设置后，需要调用重新布局方法
            }
        }

        return super.overScrollBy(deltaX, deltaY, scrollX, scrollY, scrollRangeX, scrollRangeY, maxOverScrollX, maxOverScrollY, isTouchEvent);

    }
    }
    
```

在`MainActivity`中

```java
	final ImageView parallaxImageView = (ImageView) mHeadView.findViewById(R.id.imageView);
        //当从xml中加载完成后，才能知道imageview的长高
        mHeadView.getViewTreeObserver().addOnGlobalLayoutListener(new OnGlobalLayoutListener() 		{
            @SuppressLint("NewApi")
            @Override
            public void onGlobalLayout() {
                mListView.setParallaxImageView(parallaxImageView);
                mHeadView.getViewTreeObserver().removeOnGlobalLayoutListener(this);  //取消当前的观察者
            }
        });
        mAdapter = new MyAdapter(this, R.layout.list_item, lists);
        mListView.setAdapter(mAdapter);
```


### 设置动画
	当手指抬起时，希望头部View可以慢慢地回到原来的大小。
	为达到此目的，可以先自定义Animation	,在构造方法中传入需要动画效果的View，覆写applyTransformation方法，
	该方法会传入interpolatedTime参数，表示当前动画进行的时间百分比，据此可以设置每一帧View的属性，达到动画的效果。
	
	
```java

	public class ResetHeightAnimation extends Animation {

    int mOriginalHeight, mTargetHeight;
    int totalValue;
    View mView;

    public ResetHeightAnimation(View view, int targetHeight) {
        super();
        mTargetHeight = targetHeight;
        mView = view;
        mOriginalHeight = mView.getHeight();
        totalValue = mTargetHeight - mOriginalHeight;

        setDuration(400);
        setInterpolator(new OvershootInterpolator()); //设置加速器
    }

    /**
     * @param interpolatedTime 标示动画执行的精度或者百分比
     *                         范围在0~1，对于每一个进度，我都可以在此方法中为制定的view设置属性，
     *                         达到动画额效果
     */
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        super.applyTransformation(interpolatedTime, t);
        int newHeight = (int) (mOriginalHeight + totalValue * interpolatedTime);
        mView.getLayoutParams().height = newHeight;
        mView.requestLayout();

    }
}

```

### 完整代码
[github](https://github.com/FelixZhang00/My_HeadParallax)