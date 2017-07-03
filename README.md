# CustomView
View的工作原理


View的三大流程：测量流程，布局流程，绘制流程。


一.初识ViewRoot和DecorView

1.ViewRoot对应ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。

在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立
关联。

View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure、layout、和draw三个过程才能最终将一个View绘制出来，performTraversals
会依次调用performMeasure、performLayout、performDraw三个方法，这三个方法分别完成顶级View的measure、layout、draw这三大流程，其中在performMeasure
中会调用measure方法，在measure方法中又会调用onMeasure方法，在onMeasure方法中则会对所有的子元素进行measure过程，这个时候measure过程就从父容器
传递到子元素中了，这样就完成了一次measure过程。如此反复，就完成了整个View树的遍历，同理，performLayout和performDraw的传递流程和performMeasure
是类似的，唯一不同的是，performDraw的传递过程是在draw方法中通过dispatchDraw来实现的，不过并没有什么本质区别。


measure过程决定了View的宽高，Measure完成以后，可以通过getMeasureWidth和getMeasureHeight方法来获取到View测量后的宽高，在几乎所有情况下，它都等于
View最终的宽高。Layout过程决定了View的四个顶点的坐标和实际的View宽高，完成以后，可以通过，getTop、getBottom、getLeft、getRight来拿到View的
四个顶点的位置，并且可以通过getWidth，getHeight来拿到View的最终宽高。Draw过程则决定了View的显示，只有draw方法完成以后View的内容才能呈现在屏幕上。

DecorView作为顶级View，一般情况下它内部会包含一个竖直方向的LinearLayout，这个LinearLayout里面有上下两个部分，上面是标题栏,具体情况和Android
版本和主题有关，下面是内容栏，就是是我们setContentView的View。




二.理解MeasureSpec


MeasureSpec在很大程度上决定了一个View的尺寸规格，之所以说很大程度上，是因为这个过程还受父容器的影响，在测量过程中，系统会将View的LayoutParams
根据父容器所施加的规则转换成对应的MeasureSpec，然后再根据这个MeasureSpec来测量出View的宽高。


1.MeasureSpec

MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，SpecMode和SpecSize也是一个int值。

SpecMode有三类：

（1）UNSPECIFIED

父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量的状态。


（2）EXACTLY
父容器已经检测出View所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值。它对应于LayoutParams中的match_parent和具体数值这两种模式


（3）AT_MOST
父容器指定了一个可用的大小，即SpecSize，View的大小不能大于这个值，具体是什么值，要看不同View的实现。它对应于LayoutParams中的wrap_content。



2.MeasureSpec和LayoutParams的对应关系

MeasureSpec不是唯一有LayoutParams决定的，LayoutParams需要和父容器一起才能决定View的MeasureSpec，从而进一步决定View的宽高。MeasureSpec
一旦确定后，onMeasure中就可以确定View的测量宽高。

（1）对于DecorView来说，其MeasureSpec是由窗口的尺寸或者说屏幕的尺寸和其自身的LayoutParams来共同确定的。

（2）对于普通View来说，View的measure过程由ViewGroup传递而来，先来看一下ViewGroup的measureChildWithMargins方法：


     protected void measureChildWithMargins(View child,
              int parentWidthMeasureSpec, int widthUsed,
              int parentHeightMeasureSpec, int heightUsed) {
          final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

          final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                  mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                          + widthUsed, lp.width);
          final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                  mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                          + heightUsed, lp.height);

          child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
      }

上述方法会对子元素进行measure，在调用子元素的measure方法之前会先通过getChildMeasureSpec方法来得到子元素的MeasureSpec，从下面的源码来看，很显然
子元素的MeasureSpec的创建与父容器的MeasureSpec和子元素本身的LayoutParams有关，此外还和View的margin，padding有关：

     public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
            int specMode = MeasureSpec.getMode(spec);
            int specSize = MeasureSpec.getSize(spec);

            int size = Math.max(0, specSize - padding);

            int resultSize = 0;
            int resultMode = 0;

            switch (specMode) {
            // Parent has imposed an exact size on us
            case MeasureSpec.EXACTLY:
                if (childDimension >= 0) {
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size. So be it.
                    resultSize = size;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size. It can't be
                    // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;

            // Parent has imposed a maximum size on us
            case MeasureSpec.AT_MOST:
                if (childDimension >= 0) {
                    // Child wants a specific size... so be it
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size, but our size is not fixed.
                    // Constrain child to not be bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size. It can't be
                    // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;

            // Parent asked to see how big we want to be
            case MeasureSpec.UNSPECIFIED:
                if (childDimension >= 0) {
                    // Child wants a specific size... let him have it
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size... find out how big it should
                    // be
                    resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                    resultMode = MeasureSpec.UNSPECIFIED;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size.... find out how
                    // big it should be
                    resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                    resultMode = MeasureSpec.UNSPECIFIED;
                }
                break;
            }
            return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
        }

上面的代码充分说明了MeasureSpec是如何受父容器的MeasureSpec和自身的LayoutParams的影响的。注意，其中parentSize是指父容器中目前可使用的大小。
这里再简单说明一下：

（1）当View采用固定宽高的时候，不管父容器的MeasureSpec是什么，View的MeasureSpec都是精确模式并且其大小遵循LayoutParams中的大小。

（2）当View的宽高是match_parent时，如果父容器的模式是精确模式，那么View也是精确模式，大小是父容器可使用的剩余空间；如果父容器是最大模式，
那么View也是最大模式，并且其大小不会超过父容器的剩余空间。

（3）当View的宽高是wrap——content时，不管父容器的模式是精确还是最大化，View的模式总是最大化，并且不会超过父容器的剩余空间。

UNSPECIFIED模式主要用于系统内部多次Measure的情形，一般来说，我们不需要关注此模式。

结论：只要提供父容器的MeasureSpec和子元素的LayoutParams，就能可以快速的确定出子元素的MeasureSpec了，有了MeasureSpec就可以进一步确定出子元素
测量后的大小了。





三.View的工作流程

View的工作流程主要指measure，layout，draw这三大流程。

1.measure过程

如果只是一个原始的View，那么通过measure方法就完成了其测量过程，如果是一个ViewGroup，那么除了完成自己的测量过程外，还会遍历去调用所有子元素的
measure方法，各个子元素在递归执行这个流程，下面针对这两种情况分别讨论。

（1）View的measure过程：

View的measure过程由其measure方法来完成，measure方法是一个final类型的方法，不可以被重写，在View的measure方法中回去调用View的onMeasure方法，
因此只需要看onMeasure方法即可，源码如下所示：

     protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
     }

setMeasuredDimension方法会设置View的宽高的测量值，因此我们再来看getDefaultSize这个方法：

       public static int getDefaultSize(int size, int measureSpec) {
             int result = size;
             int specMode = MeasureSpec.getMode(measureSpec);
             int specSize = MeasureSpec.getSize(measureSpec);

             switch (specMode) {
             case MeasureSpec.UNSPECIFIED:
                 result = size;
                 break;
             case MeasureSpec.AT_MOST:
             case MeasureSpec.EXACTLY:
                 result = specSize;
                 break;
             }
             return result;
         }

这里我们主要看AT_MOST和EXACTLY这两种情况就好了，简单的理解，getDefaultSize方法就是返回父容器的MeasureSpec中的SpecSize，而这个SpecSize就是
View测量后的大小。

UNSPECIFIED这种情况一般用于系统内部的测量过程，在这种情况下，View的大小为getDefaultSize的第一个参数size，即宽高为getSuggestedMinimumWidth()
这个方法的返回值，我们看一下它的源码：

     protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

也就是说，如果View设置了背景，那么返回android：minWidth这个属性所指定的值，这个值可以为0，如果View设置了背景，则返回android：minWidth和背景
最小宽度这两者中的最大值。这就是View在UNSPECIFIED情况下测量的宽高。


下面讨论另外两种情况，直接继承View的自定义控件，需要重写onMeasure方法方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content就
相当于使用match_parent，从上述代码中我们知道，如果View布局中使用wrap_content，那么它的SpecMode就是AT_MOST模式，在这种情况下，它的宽高等于
SpecSize,通过对前面MeasureSpec受自身LayoutParams和父容器MeasureSpec的影响分析可知，这时候View的SpecSize是parentSize，就是父容器中可使用
的剩余空间的大小，很显然，View的宽高就等于父容器当前剩余空间的大小，这种效果和在布局中使用match_parent的效果是一样的，如何解决这个问题呢，我们
需要重写onMeasure方法，代码如下所示：

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
        //重写wrap_content
        if(widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(mWidth,mHeight);
        }else if(widthMeasureSpec == MeasureSpec.AT_MOST){
            setMeasuredDimension(mWidth,heightSpecSize);
        }else if(heightSpecMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(widthSpecSize,mHeight);
        }
    }

在上面的代码中，我们只需要给View指定一个默认的宽高即可，并在MeasureSpec中的SpecMode为AT_MOST时设置。




（2）ViewGroup的measure过程：

ViewGroup除了要完成自己的measure过程外，还要遍历所有子元素，调用子元素的measure方法各个子元素再去递归执行这个过程，和View不同的是，ViewGroup
是一个抽象类，因此它没有重写View的onMeasure方法，但是它提供了一个measureChildre方法，如下所示：

     protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
     }

从上述代码来看，ViewGroup在measure时，会对每一个子元素进行measure，measureChild这个方法也很好理解，它的源码如下：

     protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
      }

measureChild的思想就是取出子元素的LayoutParams，然后再通过getChildMeasureSpec方法来创建子元素的MeasureSpec，接着再将MeasureSpec
直接传递给View的measure方法来进行测量。getChildMeasureSpec已经在前面分析过了，就是子元素的MeasureSpec如何受自身LayoutParams和父容器
的MeasureSpec影响的。


View的measure过程是三大流程中最复杂的一个，measure完成后，就可以通过getMeasuredWidth/Height方法来获取View的测量宽高了，一个比较好的习惯就是
在onLayout方法中获取View的最终宽高。



现在考虑一种情况：比如我们需要在Activity启动的时候就做一件任务，但是这一任务需要获取某个View的宽高，实际上在Activity的onCreate，onStart，
onResume中均无法正确获取到某个View的宽高信息，因为View的measure过程和Activity的生命周期方法不是同步的，因此无法保证Activity在执行了
onCreate，onStart，onResume时，某个View已经测量完毕了，如果没有测量完毕，那么View的宽高就是0，这里给出四个方法解决此问题：


1.Activity/View#onWindowFocusChanged

onWindowFocusChanged这个方法的含义是：View已经初始化完毕了，宽高已经准备好了，这是时候去获取宽高是没有问题的。需要注意的是
onWindowFocusChanged会被调用多次，当Activity的窗口得到和失去焦点的时候均会被调用一次。具体来说，当Activity继续执行和暂停执行时，
onWindowFocusChanged均会被调用，如果频繁的执行onResume和onPause，那么onWindowFocusChanged也会被频繁的调用。典型代码如下：

     public void onWindowFocusChanged（boolean hasFocus{
          super.onWindowFocusChanged（hasFocus);
          if(hasFocus){
               int width = view.getMeasuredWidth();
               int height = view.getMeasuredHeight();
          }
     }


2.View.post(runnable)

通过post可以将一个runnable投递到消息队列的尾部，然后等待Looper调用此runnable的时候View也已经初始化好了，典型代码如下：

     peotected void onStart(){
          super.onStart();
          view.post(new Runnable(){

               public void run(){
                    int width = view.getMeasuredWidth();
                    int height = view.getMeasuredHeight(); 
               }
          });
     }

(3)ViewTreeObserver

使用ViewTreeObserver的众多回调可以完成这个功能，比如使用onGlobalLayoutListener接口，当View树的状态发生改变或者View树内部的View的
可见性发生改变时，onGlobalLayout方法将被回调，因此这是一个获取View的宽高的很好的时机，需要注意的是，随着View树的状态改变等，onGlobalLayout
会被调用多次，典型代码如下：


    peotected void onStart(){
          super.onStart();
          
          ViewTreeObserver observer = view.getViewTreeObserver();
          observer.addOnGlobalLayoutListener(new OnGlobalLayoutListener(){
               
               public void onGlobalLayout(){
               
                    view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
                    
                    int width = view.getMeasuredWidth();
                    int height = view.getMeasuredHeight(); 
               }
          })
      }


4.通过手动对View进行measure来得到View的宽高。这种情况比较复杂，要分情况处理，根据View的LayoutParams来分：


（1）match_parent
直接放弃，原因很简单，因为根据View的measure过程，如前面分析的，构造此种MeasureSpec需要知道parentSize，即父容器的剩余空间，而这个时候我们无法知道
parentSize的大小，理论上不可能测量出View的大小。


（2）具体的数值，比如宽高都是100，如下measure：

     int widthMeasurePec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
     int heightMeasurePec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
     view.measure(widthMeasurePec,heightMeasurePec);
     int width = view.getMeasuredWidth();
     int height = view.getMeasuredHeight(); 

(3)wrap_content，如下measure：

     
     int widthMeasurePec = MeasureSpec.makeMeasureSpec(（1<<30）-1,MeasureSpec.AT_MOST);
     int heightMeasurePec = MeasureSpec.makeMeasureSpec(（1<<30）-1,MeasureSpec.AT_MOST);
     view.measure(widthMeasurePec,heightMeasurePec);
     int width = view.getMeasuredWidth();
     int height = view.getMeasuredHeight(); 

通过分析MeasureSpec的实现可以知道，View的尺寸使用30位二进制表示，也就是最大是30个1，即2^30-1，也就是（1<<30）-1，在最大化模式下，我们用View
理论上能支持的最大值去构造MeasureSpec是合理的。





2.Layout过程


Layout的作用是ViewGroup用来确定子元素的位置，Layout方法确定View本身的位置，而onLayout方法则会确定所有子元素的位置，先看View的Layout方法：

public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);----------------------确定View的四个顶点的位置

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);-------------------------------父容器确定子元素的位置
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
    
layout方法的大致流程如下:

(1)首先会通过setFrame方法来设定View的四个顶点的位置，即初始化mLeft，mRight，mTop，mBottom这四个值，View的四个顶点一旦确定，那么View在父容器
中的位置也就确定了。

（2）接着会调用onLayout方法，这个方法的用途是父容器确定子元素的位置，和onMeasure方法类型，onLayout的具体实现同样和具体的布局有关，所有View
和ViewGroup均没有真正实现onLayout方法。


注意：View的getMeasureWidth和getWidth这两个方法的区别：
（1）getMeasureWidth/Height形成于View的measure过程，getWidth/Height形成于View的layout过程。



3.draw过程：


View的绘制过程遵循如下几步：

（1）绘制背景backgroud.draw（canvas）

(2)绘制自己（onDraw）

（3）绘制children（dispatchDraw）

（4）绘制装饰（onDrawScrollBars）


这一点可以通过draw方法的源码可以明显看出来，如下所示：


     public void draw(Canvas canvas) {
             final int privateFlags = mPrivateFlags;
             final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                     (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
             mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

             /*
              * Draw traversal performs several drawing steps which must be executed
              * in the appropriate order:
              *
              *      1. Draw the background
              *      2. If necessary, save the canvas' layers to prepare for fading
              *      3. Draw view's content
              *      4. Draw children
              *      5. If necessary, draw the fading edges and restore layers
              *      6. Draw decorations (scrollbars for instance)
              */

             // Step 1, draw the background, if needed
             int saveCount;

             if (!dirtyOpaque) {
                 drawBackground(canvas);
             }

             // skip step 2 & 5 if possible (common case)
             final int viewFlags = mViewFlags;
             boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
             boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
             if (!verticalEdges && !horizontalEdges) {
                 // Step 3, draw the content
                 if (!dirtyOpaque) onDraw(canvas);

                 // Step 4, draw the children
                 dispatchDraw(canvas);

                 // Overlay is part of the content and draws beneath Foreground
                 if (mOverlay != null && !mOverlay.isEmpty()) {
                     mOverlay.getOverlayView().dispatchDraw(canvas);
                 }

                 // Step 6, draw decorations (foreground, scrollbars)
                 onDrawForeground(canvas);

                 // we're done...
                 return;
             }

             /*
              * Here we do the full fledged routine...
              * (this is an uncommon case where speed matters less,
              * this is why we repeat some of the tests that have been
              * done above)
              */

             boolean drawTop = false;
             boolean drawBottom = false;
             boolean drawLeft = false;
             boolean drawRight = false;

             float topFadeStrength = 0.0f;
             float bottomFadeStrength = 0.0f;
             float leftFadeStrength = 0.0f;
             float rightFadeStrength = 0.0f;

             // Step 2, save the canvas' layers
             int paddingLeft = mPaddingLeft;

             final boolean offsetRequired = isPaddingOffsetRequired();
             if (offsetRequired) {
                 paddingLeft += getLeftPaddingOffset();
             }

             int left = mScrollX + paddingLeft;
             int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
             int top = mScrollY + getFadeTop(offsetRequired);
             int bottom = top + getFadeHeight(offsetRequired);

             if (offsetRequired) {
                 right += getRightPaddingOffset();
                 bottom += getBottomPaddingOffset();
             }

             final ScrollabilityCache scrollabilityCache = mScrollCache;
             final float fadeHeight = scrollabilityCache.fadingEdgeLength;
             int length = (int) fadeHeight;

             // clip the fade length if top and bottom fades overlap
             // overlapping fades produce odd-looking artifacts
             if (verticalEdges && (top + length > bottom - length)) {
                 length = (bottom - top) / 2;
             }

             // also clip horizontal fades if necessary
             if (horizontalEdges && (left + length > right - length)) {
                 length = (right - left) / 2;
             }

             if (verticalEdges) {
                 topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
                 drawTop = topFadeStrength * fadeHeight > 1.0f;
                 bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
                 drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
             }

             if (horizontalEdges) {
                 leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
                 drawLeft = leftFadeStrength * fadeHeight > 1.0f;
                 rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
                 drawRight = rightFadeStrength * fadeHeight > 1.0f;
             }

             saveCount = canvas.getSaveCount();

             int solidColor = getSolidColor();
             if (solidColor == 0) {
                 final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

                 if (drawTop) {
                     canvas.saveLayer(left, top, right, top + length, null, flags);
                 }

                 if (drawBottom) {
                     canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
                 }

                 if (drawLeft) {
                     canvas.saveLayer(left, top, left + length, bottom, null, flags);
                 }

                 if (drawRight) {
                     canvas.saveLayer(right - length, top, right, bottom, null, flags);
                 }
             } else {
                 scrollabilityCache.setFadeColor(solidColor);
             }

             // Step 3, draw the content
             if (!dirtyOpaque) onDraw(canvas);

             // Step 4, draw the children
             dispatchDraw(canvas);

             // Step 5, draw the fade effect and restore layers
             final Paint p = scrollabilityCache.paint;
             final Matrix matrix = scrollabilityCache.matrix;
             final Shader fade = scrollabilityCache.shader;

             if (drawTop) {
                 matrix.setScale(1, fadeHeight * topFadeStrength);
                 matrix.postTranslate(left, top);
                 fade.setLocalMatrix(matrix);
                 p.setShader(fade);
                 canvas.drawRect(left, top, right, top + length, p);
             }

             if (drawBottom) {
                 matrix.setScale(1, fadeHeight * bottomFadeStrength);
                 matrix.postRotate(180);
                 matrix.postTranslate(left, bottom);
                 fade.setLocalMatrix(matrix);
                 p.setShader(fade);
                 canvas.drawRect(left, bottom - length, right, bottom, p);
             }

             if (drawLeft) {
                 matrix.setScale(1, fadeHeight * leftFadeStrength);
                 matrix.postRotate(-90);
                 matrix.postTranslate(left, top);
                 fade.setLocalMatrix(matrix);
                 p.setShader(fade);
                 canvas.drawRect(left, top, left + length, bottom, p);
             }

             if (drawRight) {
                 matrix.setScale(1, fadeHeight * rightFadeStrength);
                 matrix.postRotate(90);
                 matrix.postTranslate(right, top);
                 fade.setLocalMatrix(matrix);
                 p.setShader(fade);
                 canvas.drawRect(right - length, top, right, bottom, p);
             }

             canvas.restoreToCount(saveCount);

             // Overlay is part of the content and draws beneath Foreground
             if (mOverlay != null && !mOverlay.isEmpty()) {
                 mOverlay.getOverlayView().dispatchDraw(canvas);
             }

             // Step 6, draw decorations (foreground, scrollbars)
             onDrawForeground(canvas);
         }

    
    
  View绘制过程的传递是通过dispatchDraw来实现的，dispatchDraw会遍历调用所有子元素的draw方法，如此draw事件就一层层地传递下去。
  
  
  
  View有一个特殊的方法setWillNotDraw，先看一下他的源码：
    
          /**
          * If this view doesn't do any drawing on its own, set this flag to
          * allow further optimizations. By default, this flag is not set on
          * View, but could be set on some View subclasses such as ViewGroup.
          *
          * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
          * you should clear this flag.
          *
          * @param willNotDraw whether or not this View draw on its own
          */
         public void setWillNotDraw(boolean willNotDraw) {
             setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
         }
    
   如果 一个View不需要绘制任何内容，那么设置这个标记位为true后，系统会进行相应的优化。默认情况下，View没有启用这个优化标记位，但是ViewGroup
   会默认启动这个优化标记位，这个标记位对实际开发的意义是：
   
   当我们的自定义View控件继承于ViewGroup并且本身不具备绘制功能时，就可以开启这个标记位从而便于系统进行后续的优化。当然，当明确知道一个
   ViewGroup需要通过onDraw来绘制内容时，我们需要显式地关闭WILL_NOT_DRAW这个标记位。




四.自定义View


1.自定义View须知：

（1）让View支持wrap_content：因为直接继承View或者ViewGroup的控件，如果不在onMeasure中对wrap_content做特殊处理，那么当外界在布局中使用
wrap_content时就无法达到预期的效果，和match_parent一致。

（2）如果有必要，让你的View支持padding：因为直接继承View的控件，如果不在draw方法中处理padding，那么padding效果是无法起作用的，另外，
直接继承ViewGroup的控件需要在onMeasure和onLayout中考虑padding和子元素的margin对其造成的影响，不然将导致padding和子元素的margin失效。


（3）尽量不要在View中使用Handler，没必要：因为View的本身就提供了post系列的方法，完全可以替代Handler的作用。


（4）如果View中有线程或者动画，需要及时停止，参考View#onDetchedFromWindow：如果有线程或者动画需要停止时，那么onDetchedFromWindow是一个
很好的时机，当包含此View的Activity退出或者当前View被remove时，View的onDetchedFromWindow方法会被调用，和此方法对应的是onAttachedToWindow，
当包含此View的Activity启动时，View的onAttachedToWindow方法被调用，



（5）View带有滑动嵌套时，需要处理好滑动的冲突。



2.自定义属性：

（1）在values目录下创建自定义属性的XML，比如attrs.xml。文件内容如下：

     <?xml version="1.0" encoding="utf-8"?>
     <resources>
         <declare-styleable name="CircleView">
             <attr name="circle_color" format="color"/>
         </declare-styleable>
     </resources>
     
 在上面的XML中声明了一个自定义属性集合“CircleView”,在这个集合中有很多自定义属性，这里只定义了一个格式为“color”的属性“circle_color”,这里
 的格式color指的是颜色，除了颜色格式，自定义属性还有其他格式，比如reference是指id，demension是指尺寸，而像string，integer和boolean这种
 是指基本数据类型。
 
 
（2）在View的构造方法中解析自定义属性的值并做相应处理，对于本例来说，我们需要解析circle_color这个属性的值，代码如下所示：     
     
     public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        /**
         * 解析自定义属性
         */
        //加载自定义属性集合CircleView
        TypedArray a = context.obtainStyledAttributes(attrs,R.styleable.CircleView);
        //加载指定属性circle_color属性，如果没有指定，则加载Color.Red
        mColor = a.getColor(R.styleable.CircleView_circle_color,Color.RED);
        a.recycle();
        init();
    } 
     
首先加载自定义属性集合CircleView，接着解析CircleView属性集合中的circle_color属性，它的id为R.styleableCircleView_circle_color.



(3)在布局文件中使用自定义属性：

     <?xml version="1.0" encoding="utf-8"?>
     <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
         xmlns:app="http://schemas.android.com/apk/res-auto"
         android:id="@+id/activity_main"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         >

         <com.example.saber.customviewstest.CircleView
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             app:circle_color="@color/light_green"
              />
     </LinearLayout>
     
首先为了使用自定义属性，必须在布局中添加schemas声明：xmlns:app="http://schemas.android.com/apk/res-auto"。在这个声明中，app是自定义属性的
前缀，当然也可以换其他名字，但是CircleView中的自定义属性前缀必须和这里的一致。
     
     
     
     
     
     
     
     























