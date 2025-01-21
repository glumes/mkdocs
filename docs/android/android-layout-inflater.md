---
title: "Android 布局加载之 LayoutInflater"
date: 2017-12-22T15:55:08+08:00
categories: ["Android"]
tags: ["Android","Inflater"]
toc: true
original: true
addwechat: true
 
slug: "android-layout-inflater"
---


Activity 在界面创建时需要将 XML 布局文件中的内容加载进来，正如我们在 ListView 或者 RecyclerView 中需要将 Item 的布局加载进来一样，都是使用 LayoutInflater 来进行操作的。

LayoutInflater 实例的获取有多种方式，但最终是通过`(LayoutInflater)context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)`来得到的，也就是说加载布局的 `LayoutInflater` 是来自于系统服务的。

<!--more-->

由于 Android 系统源码中关于 Content 部分采用的是装饰模式，Context 的具体功能都是由 `ContextImpl` 来实现的。通过在 ContextImpl 中找到`getSystemService`的代码，一路跟进，得知最后返回的实例是`PhoneLayoutInflater`。

``` java
        registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
```
LayoutInflater 只是一个抽象类，而 PhoneLayoutInflater 才是具体的实现类。

## inflate 方法加载 View

使用 LayoutInflater 时常用方法就是`inflate`方法了，将一个布局文件 ID 传入并最后解析成一个 View 。

LayoutInflater 加载布局的 inflate 方法也有多种重载形式：
``` java
View inflate(@LayoutRes int resource, @Nullable ViewGroup root)
View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
```

而这两者的差别就在于是否要将 `resource` 布局文件加载到 `root`布局中去。

不过有点需要注意的地方，若 `root`为 null，则在 xml 布局中为 `resource`设置的属性会失效，只是单纯的加载布局。
``` java
				  // temp 是 xml 布局中的顶层 View
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    if (root != null) { // root 
	                    // root 不为 null 才会生成 layoutParams
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
							//  如果不添加到 root 中，则直接把布局参数设置给 temp
                            temp.setLayoutParams(params);
                        }
                    }
                    // 加载子 View 
					rInflateChildren(parser, temp, attrs, true);
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);//添加到布局中，则布局参数用到 addView 中去
                    }
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
```

跟进`createViewFromTag`方法查看 View 是如何创建出来的。
``` java
			View view; // 最后要返回的 View
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs); // 是否设置了 Factory2 
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs); // 是否设置了 Factory
            } else {
                view = null;
            }
            if (view == null && mPrivateFactory != null) { // 是否设置了 PrivateFactory
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {  // 如果的 Factory 都没有设置过，最后在生成 View
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) { // 系统控件 
                        view = onCreateView(parent, name, attrs);
                    } else { // 非系统控件，自定义的 View 
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }
```

如果设置过 Factory 接口，那么将由 Factory 中的 onCreateView 方法来生成 View 。

关于 `LayoutInflater.Factory` 的作用，就是用来在加载布局时可以自行去创建 View，抢在系统创建 View 之前去创建。

关于 `LayoutInflater.Factory` 的使用场景，现在比较多的就是应用的换肤了。

若没有设置过 Factory 接口，则是判断是否为自定义控件或者系统控件，不管是 onCreateView 方法还是 createView 方法，内部最终都是调用到了 createView 方法，通过它来生成 View 。

``` java
// 通过反射生成 View 的参数，分别是 Context 和 AttributeSet 类
static final Class<?>[] mConstructorSignature = new Class[] {
            Context.class, AttributeSet.class};
            
public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

		if (constructor == null) { // 从缓存中得到 View 的构造器，没有则调用 getConstructor
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {  // 过滤，是否允许生成该 View
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) :                  name).asSubclass(View.class);
                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs); // 不允许生成该 View
                    }
                }
            }
        Object[] args = mConstructorArgs;
        args[1] = attrs;
        final View view = constructor.newInstance(args); // 通过反射生成 View
		return view;
```

在 createView 方法内部，首先从 View 的构造器缓存中查找是否有对应的缓存，若没有则生成构造器并且放到缓存中去，若有构造器则看能否通过过滤，是否允许该 View 生成。

最后都满足条件的则是通过 View 的构造器反射生成了 View 。


在生成 View 时采用 `Constructor.newInstance`调用构造函数，而参数所需要的变量就是`mConstructorSignature`变量所定义的，分别是 `Context` 和 `AttributeSet`。可以看到，在最后生成 View 时也传入了对应的参数。

采用 `Constructor.newInstance`的形式反射生成 View ，是为了解耦，只需要有了类名，就可以加载出来。

由此可见，LayoutInflater 加载布局仍然是需要传递 `Context`的，不光是为了得到 LayoutInflater ，在反射生成 View 时同样会用到。



## 深度遍历加载布局

如果需要加载的布局只有一个控件，那么 LayoutInflater 返回那个 View 工作也就结束了。

若布局文件中有多个需要加载的 View ，则通过`rInflateChildren`方法继续加载顶层 View 下的 View ，最后通过`rInflate`方法来加载。

``` java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
        final int depth = parser.getDepth();
        int type;
		// 若 while 条件不成立，则加载结束了
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }
            final String name = parser.getName(); // 从 XmlPullParser 中得到 name 出来解析
            
            if (TAG_REQUEST_FOCUS.equals(name)) { // name 各种情况下的解析
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true); // 继续遍历
                viewGroup.addView(view, params); // 顶层 View 添加 子 View
            }
        }

        if (finishInflate) { // 遍历解析
            parent.onFinishInflate();
        }
    }
```
`rInflate`方法首先判断是否解析结束了，若没有，则从 XmlPullParser 中加载出下一个 View 进行处理，中间还会对不同的类型进行处理，比如`TAG_REQUEST_FOCUS`、`TAG_TAG`、`TAG_INCLUDE`、`TAG_MERGE`等等。

最后仍然还是通过`createViewFromTag`来生成 View ，并以这个生成的 View 为父节点，开始深度遍历，继续调用`rInflateChildren`方法加载布局，并把这个 View 加入到它的父 View 中去。

至于为什么生成 View 的方法名字`createViewFromTag`从字面上来看是来自于 `Tag`标签，想必是和 `XmlPullParser`解析布局生成的内容有关。



## 参考

1. http://www.sunnyang.com/661.html?utm_source=tuicool&utm_medium=referral
2. http://blog.csdn.net/lmj623565791/article/details/51503977
3. https://segmentfault.com/a/1190000003813755
4. http://blog.csdn.net/panda1234lee/article/details/9009719


