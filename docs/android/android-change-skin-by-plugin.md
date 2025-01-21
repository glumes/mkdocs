---
title: "Android 插件换肤原理及源码分析"
date: 2017-12-22T15:51:44+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
slug: "android-change-skin-by-plugin"
 
---


在学习安卓插件化开发的路上，有一处风景是肯定要观赏的，那就是基于插件的应用换肤了。

<!--more-->

## 插件换肤原理概述

基于 插件进行应用换肤 的技术大致可以分为两个方面：

*	如何加载插件包中各式各样的资源，如 drawable、color 等。
*	如何定位到需要换肤的控件，并优雅地更改样式，如 无须重启换肤 等。

针对第一个问题，相关的研究已经比较多了，通过研究 `Resource`类 的源码，在其构造函数中有个`AssetManager`类参数，而最终获取资源都是通过`AssetManager`来获取的。

于是，通过构造`AssetManager`并生成插件的`Resource`类，就可以加载插件包中的资源。

针对第二个问题，首先是定位需要换肤的控件，大多数是通过在控件的 XML 布局中添加 ***标识***，标识那些需要换肤的控件及需要改变的属性。然后再通过控件的`set`方法改变属性即可。

在改变控件的属性时，若每次都通过遍历页面所有 View 来换肤则性能开销太大，通过 `LayoutInflater.Factory` 接口在加载布局文件时便先处理所有 View 的属性，只保存那些需要换肤的控件，则会优化性能。

当然还是有其他问题待解决的，例如：`Resource`类加载的资源 ID 冲突， 插件`Resource`不同安卓版本的兼容性，使用`LayoutInflater.Factory`是一种侵入式编程，会干涉系统构造 View 的过程，如何无侵入的换肤，动态加载的控件如何进行换肤 等等问题......

看到很多换肤的框架都参考了该工程，也来分析一下其原理。

再了解插件换肤的大致原理后，再去分析换肤框架的源码就变得简单多了，无非就是要解决上述的问题，下面就对 Android-Skin-Loader 源码进行分析。

## 动态加载插件资源

在 `SkinManager`的`load`方法中，加载了插件包，并且得到了插件的资源`Resource`。
``` java
						PackageManager mPm = context.getPackageManager();
						PackageInfo mInfo = mPm.getPackageArchiveInfo(skinPkgPath, PackageManager.GET_ACTIVITIES);
						// 得到插件包名，根据包名和资源 ID 得到资源
						skinPackageName = mInfo.packageName;

						// 通过反射构造 AssetManager 类
						AssetManager assetManager = AssetManager.class.newInstance();
						Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
						// 反射调用 addAssetPath 方法
						addAssetPath.invoke(assetManager, skinPkgPath);
						// 得到皮肤插件的 Resource
						Resources superRes = context.getResources();
						Resources skinResource = new Resources(assetManager,superRes.getDisplayMetrics(),superRes.getConfiguration());
						
						// 保存皮肤包路径
						SkinConfig.saveSkinPath(context, skinPkgPath);
```

有了插件资源`Resource`，就可以去得到想要的资源了。


## 换肤控件及属性的标识

Android-Skin-Loader 框架自定义了一个 `enable`的属性，用在 XML 文件中来标识哪些控件需要进行换肤。

并且 Android-Skin-Loader 在需要继承的基类 `BaseActivity`、`BaseFragment`、`BaseFragmentActivity`中都设置了`LayoutInflater.Factory`，以便在布局加载之前进行预操作，也就是保存那些 需要换肤的控件 和识别 需要换肤的属性，这里 *换肤控件* 和 *换肤属性* 两个东西要区别开，它们所要进行的操作是不一样的，要先找到 *换肤控件*，然后再去找它的 *换肤属性* 。

``` java
@Override
	public View onCreateView(String name, Context context, AttributeSet attrs) {
		// 从 AttributeSet 中得到换肤属性，判断是否需要进行换肤
		boolean isSkinEnable = attrs.getAttributeBooleanValue(SkinConfig.NAMESPACE, SkinConfig.ATTR_SKIN_ENABLE, false);
        if (!isSkinEnable){
        		return null; // 不需要换肤的，则返回 null，由系统构造
        }
		// 构造需要换肤的 View
		View view = createView(context, name, attrs);
		if (view == null){
			return null;
		}
		// 解析需要换肤 View 的属性
		parseSkinAttr(context, attrs, view);
		return view;
	}
```
以上代码就是我们所说的侵入式编程，干扰了系统构造 View 的过程，所做的工作就是找出需要换肤的 View 并交由下一步进行解析。

``` java
private void parseSkinAttr(Context context, AttributeSet attrs, View view) {
		List<SkinAttr> viewAttrs = new ArrayList<SkinAttr>();
		
		for (int i = 0; i < attrs.getAttributeCount(); i++){
			String attrName = attrs.getAttributeName(i);
			String attrValue = attrs.getAttributeValue(i);
			
			if(!AttrFactory.isSupportedAttr(attrName)){
				continue; // AttrFactory 定义了哪些属性支持换肤，若该属性不支持换肤就跳过继续
			}
		    if(attrValue.startsWith("@")){ // 表明是引用类型，例如 @color/red
					int id = Integer.parseInt(attrValue.substring(1));
					String typeName = context.getResources().getResourceTypeName(id); // 类型名
					String entryName = context.getResources().getResourceEntryName(id); // 入口名
					SkinAttr mSkinAttr = AttrFactory.get(attrName, id, entryName, typeName);
					if (mSkinAttr != null) {
						viewAttrs.add(mSkinAttr);
					}
		    }
		    }
		if(!ListUtils.isEmpty(viewAttrs)){
			SkinItem skinItem = new SkinItem();
			skinItem.view = view;
			skinItem.attrs = viewAttrs;
			// 将需要 换肤的控件 和 换肤的属性 这两个东西进行保存
			mSkinItems.add(skinItem);
			if(SkinManager.getInstance().isExternalSkin()){
				skinItem.apply(); // 如果是外部的皮肤，则要 apply 一下，防止换肤不及时
			}
		}
	}
```
上述代码的作用就是从需要换肤的控件中，找到那些需要更改的属性，并将它保存在`mSkinItems`全局变量中。

Android-Skin-Loader 框架中有一个抽象基类`SkinAttr`表示需要更改的属性，而具体需要更改的属性都是继承自`SkinAttr`，所以如果想要更改更多的属性，就必须自己添加对应的`SkinAttr`类了。

解析完属性后，换肤控件及属性都被保存在了`mSkinItems`全局变量中，这样就完成了加载布局界面的预操作。

显然，Android-Skin-Loader 框架对于解决找出待更改的属性这一问题，并不是那么的方便，并且干预了系统构造 View 的过程。

下面研究 hongyang 大神的解决思路：[AndroidChangeSkin](https://github.com/hongyangAndroid/AndroidChangeSkin) 代码。


AndroidChangeSkin 框架并没有使用 `LayoutInflater.Factory`方案了，采用了一种无侵入的方案。

对于标识需要换肤的控件这一问题，AndroidChangeSkin 并没有再添加自定义属性，而是使用 View 自带的 `tag`属性。并在在`tag`属性的字符串值中，传递了要换肤的标识、要换肤的属性、要换肤的属性名。通过解析这三者来完成标识的任务。这样就不必要对每个属性都进行操作了。

``` java
	//传入activity，找到content元素，递归遍历所有的子View，根据tag命名，记录需要换肤的View
	public static List<SkinView> getSkinViews(Activity activity)
    {
        List<SkinView> skinViews = new ArrayList<SkinView>();
        ViewGroup content = (ViewGroup) activity.findViewById(android.R.id.content);
        addSkinViews(content, skinViews); // 找到需要换肤的 View 放到 skinViews 里面
        return skinViews;
    }
    
	 /**
     * 得到换肤的 View ，如果 tag 为 null 或者 tag 不是字符串，则返回 null
     * 解析需要换肤的 View ，得到所有需要更改的属性 SkinAttr
     */
    public static SkinView getSkinView(View view)
    {
        Object tag = view.getTag(R.id.skin_tag_id);
        if (tag == null)
        {
            tag = view.getTag();
        }
        if (tag == null) return null;
        if (!(tag instanceof String)) return null;
        String tagStr = (String) tag;

        List<SkinAttr> skinAttrs = parseTag(tagStr);
        if (!skinAttrs.isEmpty())
        {
            changeViewTag(view);
            return new SkinView(view, skinAttrs);
        }
        return null;
    }
```
AndroidChangeSkin 完成查找*换肤控件*和*换肤属性*两大任务，之所以说是无侵入性，就是因为它是从 Activity 布局的顶层开始遍历的，是在布局文件加载完成之后。

## 换肤操作及响应回调

完成了 View 的标识及查找任务之后，剩下就是最终的换肤操作了。

要做到无须重启应用和 Activity 完成换肤，Android-Skin-Loader 和 AndroidChangeSkin 都是基于*观察者模式*来处理的，也就是通过回调方法。

收到进行换肤的指令时，在页面中响应回调方法，通过皮肤插件的`Resource`加载对应的资源完成替换。

在此之前，我们找到了需要换肤的 View 和需要更改的属性，那么最终的换肤操作也就是由这些 View 来设置它的新属性，插件资源的加载也就是发生在这里了。也只有在这个时候，才会去加载皮肤插件中的资源，而之前的第一步只是构造插件的`Resource`并没有加载资源。


Android-Skin-Loader 是为每一个需要更改的属性定义了一个类，并在此类中去加载资源。
``` java
public class TextColorAttr extends SkinAttr {

	@Override
	public void apply(View view) {
		if(view instanceof TextView){
			TextView tv = (TextView)view;
			if(RES_TYPE_NAME_COLOR.equals(attrValueTypeName)){
		 tv.setTextColor(SkinManager.getInstance().convertToColorStateList(attrValueRefId));
```

而 androidChangeSkin 则没有编写那么多类，采用了枚举类型来更改属性，同样也是在属性中加载资源。
``` java
public enum SkinAttrType
{
    BACKGROUND("background"){
                @Override
                public void apply(View view, String resName)
                {
                    Drawable drawable = getResourceManager().getDrawableByName(resName);
                    if (drawable != null)
                    {
                        view.setBackgroundDrawable(drawable);
                    } else{
					try{
	                     int color = getResourceManager().getColor(resName);
	                     view.setBackgroundColor(color);
					},
```

至于，基于观察者模式来响应换肤操作就比较简单了，看过代码很容易知道是怎么一个编程方式了。

## 动态添加 View 的换肤

以上只是分析了从布局文件中加载的换肤，对于运行时动态添加的 View 同样可以换肤，只不过不能再 XML 文件中添加属性了。

毕竟即使是在运行时添加的 View 也是要先确定好需求，编写对应代码的。只不过少了标识该 View 是否需要换肤这一步，直接找到需要换肤的属性就好了，在收到换肤指令时，也是加载插件资源，直接更改属性即可。


## 总结

在总结了基于插件换肤的原理和相关代码之后，发现其实应用换肤也不是那么难嘛....


## 参考
1. http://blog.zhaiyifan.cn/2015/09/10/Android%E6%8D%A2%E8%82%A4%E6%8A%80%E6%9C%AF%E6%80%BB%E7%BB%93/
2. http://blog.csdn.net/zhi184816/article/details/53436761
3. http://www.jianshu.com/p/af7c0585dd5b




