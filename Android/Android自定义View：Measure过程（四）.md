# 1.作用

测量`View`的宽 / 高

> 在某些情况下，需要多次测量`（measure）`才能确定`View`最终的宽/高；
>
> 该情况下，`measure`过程后得到的宽 / 高可能不准确；
>
> 此处建议：在`layout`过程中`onLayout()`去获取最终的宽 / 高



# 2.储备知识

### 1.ViewGroup.LayoutParams

* 作用：指定视图`View` 的高度`（height）` 和 宽度`（width）`等布局参数。
* 具体使用

| 参数         |                             解释                             |
| ------------ | :----------------------------------------------------------: |
| 具体值       |                           dp / px                            |
| fill_parent  |  强制性使子视图的大小扩展至与父视图大小相等（不含 padding )  |
| match_parent |        与fill_parent相同，用于Android 2.3 & 之后版本         |
| wrap_content | 自适应大小，强制性地使视图扩展以便显示其全部内容(含 padding ) |



### 2.MeasureSpec

* 定义：测量View大小的依据
* 作用：决定一个视图View的大小
* 组成：测量规格`（MeasureSpec）` = 测量模式`（mode）` + 测量大小`（size）`。mode占高2位，size占低30位。
* 测量模式`（Mode）`的类型有3种：UNSPECIFIED、EXACTLY 和AT_MOST。

> 该措施的目的 = 减少对象内存分配

```java
		// 1. 获取测量模式（Mode）
    int specMode = MeasureSpec.getMode(measureSpec)

    // 2. 获取测量大小（Size）
    int specSize = MeasureSpec.getSize(measureSpec)
```

* MeasureSpec值的计算

```java
/**
  * 源码分析：getChildMeasureSpec（）
  * 作用：根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
  * 注：子view的大小由父view的MeasureSpec值 和 子view的LayoutParams属性 共同决定
  **/

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  

         //参数说明
         * @param spec 父view的详细测量值(MeasureSpec) 
         * @param padding view当前尺寸的的内边距和外边距(padding,margin) 
         * @param childDimension 子视图的布局参数（宽/高）

            //父view的测量模式
            int specMode = MeasureSpec.getMode(spec);     

            //父view的大小
            int specSize = MeasureSpec.getSize(spec);     
          
            //通过父view计算出的子view = 父大小-边距（父要求的大小，但子view不一定用这个值）   
            int size = Math.max(0, specSize - padding);  
          
            //子view想要的实际大小和模式（需要计算）  
            int resultSize = 0;  
            int resultMode = 0;  
          
            //通过父view的MeasureSpec和子view的LayoutParams确定子view的大小  


            // 当父view的模式为EXACITY时，父view强加给子view确切的值
           //一般是父view设置为match_parent或者固定值的ViewGroup 
            switch (specMode) {  
            case MeasureSpec.EXACTLY:  
                // 当子view的LayoutParams>0，即有确切的值  
                if (childDimension >= 0) {  
                    //子view大小为子自身所赋的值，模式大小为EXACTLY  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  

                // 当子view的LayoutParams为MATCH_PARENT时(-1)  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    //子view大小为父view大小，模式为EXACTLY  
                    resultSize = size;  
                    resultMode = MeasureSpec.EXACTLY;  

                // 当子view的LayoutParams为WRAP_CONTENT时(-2)      
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    //子view决定自己的大小，但最大不能超过父view，模式为AT_MOST  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                }  
                break;  
          
            // 当父view的模式为AT_MOST时，父view强加给子view一个最大的值。（一般是父view设置为wrap_content）  
            case MeasureSpec.AT_MOST:  
                // 道理同上  
                if (childDimension >= 0) {  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                }  
                break;  
          
            // 当父view的模式为UNSPECIFIED时，父容器不对view有任何限制，要多大给多大
            // 多见于ListView、GridView  
            case MeasureSpec.UNSPECIFIED:  
                if (childDimension >= 0) {  
                    // 子view大小为子自身所赋的值  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    // 因为父view为UNSPECIFIED，所以MATCH_PARENT的话子类大小为0  
                    resultSize = 0;  
                    resultMode = MeasureSpec.UNSPECIFIED;  
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    // 因为父view为UNSPECIFIED，所以WRAP_CONTENT的话子类大小为0  
                    resultSize = 0;  
                    resultMode = MeasureSpec.UNSPECIFIED;  
                }  
                break;  
            }  
            return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
        }  
```

规律总结：

![总结](img/measure_spec_detail.png?raw=true)



# 3.Measer过程详解

### 1.单一View的measure过程

* 具体使用：继承自`View`、`SurfaceView` 或 其他`View`；不包含子`View`
* 具体流程：

