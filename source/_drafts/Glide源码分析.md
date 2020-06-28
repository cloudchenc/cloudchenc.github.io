---
title: Glide源码分析
tags:
categories:
---

```java
Glide.with(this).load(url).into(imageView);
```

###1. with()

```java
    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }

    public static RequestManager with(FragmentActivity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    public static RequestManager with(android.app.Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }

    public static RequestManager with(Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }
```
每一个with()方法都是先调用RequestManagerRetriever的静态方法get()得到一个RequestManagerRetriever对象，  
然后在调用实例的get()方法，获取到RequestManager对象。

这里with()传入的参数，最终只分两种情况，Application类型参数和非Application类型参数

当在主线程时，Application类型参数会使Glide的生命周期与应用程序的生命周期同步，当应用终止，Glide的加载才会停止。  
            非Application类型参数会以`com.bumptech.glide.manager`为tag创建一个隐藏的fragment(即RequestManagerFragment或SupportRequestManagerFragment)，
            然后向当前的activity中添加，Glide通过监听fragment的生命周期，判断当前继续/停止加载图片。

当在子线程时，非Application类型的参数会被强制转为Application类型来处理


###2. load()

以load图片url举例，
```java
    public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }

    public DrawableTypeRequest<String> fromString() {
        return loadGeneric(String.class);
    }

    private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        if (modelClass != null && streamModelLoader == null && fileDescriptorModelLoader == null) {
            throw new IllegalArgumentException("Unknown type " + modelClass + ". You must provide a Model of a type for"
                    + " which there is a registered ModelLoader, if you are using a custom model, you must first call"
                    + " Glide#register with a ModelLoaderFactory for your custom model class");
        }

        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }
```

###3. into()

参考资料：

https://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247485940&idx=1&sn=0fcb7b8a433392c8ad80579b3b59c2fa&chksm=ebd87966dcaff07014234d702fe39ce3fa7f5c338f56b1db339d999dacb9a490c9a80eee5fee&scene=21#wechat_redirect
https://mp.weixin.qq.com/s?__biz=MzI4NjY4MTU5Nw==&mid=2247486204&idx=2&sn=eabf558413baf96c3474533605c40dbf&chksm=ebd87a6edcaff378d0682133904ad32517348cc797ae1906d0453a631bac0a02968dbea8af2d&mpshare=1&scene=1&srcid=0524IBWDjiBZodSxMnqGqTHq&sharer_sharetime=1590302161316&sharer_shareid=9216e6e62e990610fc0215dd4d76a033#rd
https://juejin.im/post/5e2109e25188254c257c40c6  

https://blog.csdn.net/guolin_blog/article/details/53759439  
https://blog.csdn.net/guolin_blog/article/details/53939176   
https://blog.csdn.net/guolin_blog/article/details/54895665  
https://blog.csdn.net/guolin_blog/article/details/70215985   
