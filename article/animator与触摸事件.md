```
    //in ViewGroup.java
    protected boolean isTransformedTouchPointInView(float x, float y, View child,
            PointF outLocalPoint) {
        final float[] point = getTempPoint();
        point[0] = x;
        point[1] = y;
        transformPointToViewLocal(point, child);
        final boolean isInView = child.pointInView(point[0], point[1]);
        if (isInView && outLocalPoint != null) {
            outLocalPoint.set(point[0], point[1]);
        }
        return isInView;
    }
    
    public void transformPointToViewLocal(float[] point, View child) {
        point[0] += mScrollX - child.mLeft;
        point[1] += mScrollY - child.mTop;
        if (!child.hasIdentityMatrix()) {
            child.getInverseMatrix().mapPoints(point);
        }
    }
    //in View.java
     public boolean pointInView(float localX, float localY, float slop) {
        return localX >= -slop && localY >= -slop && localX < ((mRight - mLeft) + slop) &&
                localY < ((mBottom - mTop) + slop);
    }
```
上面两个方法是ViewGroup里面判断child是否能够接收触摸事件的决议方法，为什么写这两个方法呢，因为最近清闲，学习了一下绘图体系，学到Draw方法的时候，发现google通过独立的scroll参数通过影响canvas坐标来实现内容平移，但view的layout坐标left等保持不变，google这样做的原因未知，也有可能是遵循惯例:)，但google实现这种方案的时候却对一个问题考虑不周，那就是事件决议，就像前面两个方法展示的，当ViewGroup进行事件分发判断子view是否在点击区域内的时候，加入了ViewGroup的scroll偏移，也就是说当ViewGroup发生滑动的时候，事件依然能够正确定位子View，但如果子View滑动，在子view的pointInView方法内却没有考虑scroll偏移，这导致子view滑动后无法正确定位点击事件，为了弥补这个问题，google在加入硬件加速时，顺便利用RenderNode重新实现了另外一套所谓的属性动画，属性动画虽然叫属性动画，其实也并不是直接改变view的属性，仍然是在view的属性上加一个偏移量，只不过这次是通过RenderNode直接对Frame Buffer中的内容进行矩阵变换，那么属性如果没变，触摸事件决议的时候怎么办呢，关键就在于child.hasIdentityMatrix()，原来RenderNode中保存了所有的矩阵变换，触摸事件决议的时候，就像ViewGroup考虑scroll那样，把这个矩阵取出来，也考虑进去就可以了