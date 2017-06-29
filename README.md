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


2.














































