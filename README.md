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











































