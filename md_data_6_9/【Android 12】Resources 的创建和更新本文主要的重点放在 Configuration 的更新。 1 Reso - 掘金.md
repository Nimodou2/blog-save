> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7153220288899973133)

> 本文主要的重点放在 Configuration 的更新。 1 Resources 的创建 根据一个 Resources 对象在创建的时候是否与一个 Activity 有关联，可以将 Resources 分为 Activi

本文主要的重点放在 Configuration 的更新。

根据一个 Resources 对象在创建的时候是否与一个 Activity 有关联，可以将 Resources 分为 Activity 类型 Resources 和非 Activity 类型 Resources。

1.1 Activity 类型 Resources 的创建
-----------------------------

创建 Activity 类型 Resources 的地方在 ResourcesManager#createResourcesForActivityLocked：

```
    @NonNull
    private Resources createResourcesForActivityLocked(@NonNull IBinder activityToken,
            @NonNull Configuration initialOverrideConfig, @Nullable Integer overrideDisplayId,
            @NonNull ClassLoader classLoader, @NonNull ResourcesImpl impl,
            @NonNull CompatibilityInfo compatInfo) {
        final ActivityResources activityResources = getOrCreateActivityResourcesStructLocked(
                activityToken);
        cleanupReferences(activityResources.activityResources,
                activityResources.activityResourcesQueue,
                (r) -> r.resources);
        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
                : new Resources(classLoader);
        resources.setImpl(impl);
        resources.setCallbacks(mUpdateCallbacks);
        ActivityResource activityResource = new ActivityResource();
        activityResource.resources = new WeakReference<>(resources,
                activityResources.activityResourcesQueue);
        activityResource.overrideConfig.setTo(initialOverrideConfig);
        activityResource.overrideDisplayId = overrideDisplayId;
        activityResources.activityResources.add(activityResource);
        if (DEBUG) {
            Slog.d(TAG, "- creating new ref=" + resources);
            Slog.d(TAG, "- setting ref=" + resources + " with impl=" + impl);
        }
        return resources;
    }


```

首先这里传入了一个 IBinder 类型的 activityToken，这个是 ActivityClientRecord 的 token 对象，代表了当前 Activity。

### 1.1.1 ActivityResources 类的概念

```
        final ActivityResources activityResources = getOrCreateActivityResourcesStructLocked(
                activityToken);


```

接着通过 ResourcesManager#getOrCreateActivityResourcesStructLocked 获取一个 ActivityResources 对象，碰到了第一个新的类，ActivityResources。

ActivityResources 类定义在 ResourcesManager 中：

```
    
    private static class ActivityResources {
        
        public final Configuration overrideConfig = new Configuration();
        
        public int overrideDisplayId;
        
        public final ArrayList<ActivityResource> activityResources = new ArrayList<>();
        ......
    }


```

包含了与一个 Activity 或者 WindowContext 相关联的基础 Configuration 复写和 Resources 集合。

这也就是说，每一个 Activity 或者 WindowContext，都可能对应多个 Resources。在创建一个 Activity 或者 WindowContext 的时候，系统都会为其创建一个 ActivityResources 对象，这个 ActivityResources 的作用之一就是为 Activity 或者 WindowContext 保存与之联系的所有 Resources 对象，通过 ActivityResources 的成员变量 activityResources。在下面会看到，这些 Resources 对象在创建的时候，会被封装为一个 ActivityResource 对象，然后添加到 ActivityResources.activityResources 中。

另外，ActivityResources 还有一个 overrideConfig 成员变量，它会被应用到 ActivityResources.activityResources 队列中的所有 Resources 的 Configuration，后面会看到它的具体作用。

### 1.1.2 ActivityResource 类的概念

ActivityResource 类也定义在 ResourcesManager 中：

```
    
    
    private static class ActivityResource {
        
        public final Configuration overrideConfig = new Configuration();
        
        @Nullable
        public Integer overrideDisplayId;
        @Nullable
        public WeakReference<Resources> resources;
        private ActivityResource() {}
    }


```

ActivityResources 就是对 Resources 的一层封装，这里有两个值得关注的成员变量：

1）、resources，即当前 ActivityResource 对象保存的那个 Resources 对象的弱饮用。

2）、overrideConfig，保存了一个 Activity 或者 WindowContext 对象创建的时候的初始 Configuration，这里我们看到 ResourcesManager#createResourcesForActivityLocked 方法还有一个参数：

```
@NonNull Configuration initialOverrideConfig


```

也就是说在创建 Activity 或者 WindowContext 的时候，也可以指定一个初始 Configuration，就保存在 ActivityResource.overrideConfig 中。这个初始 Configuration 保存的是 Activity 或者 WindowContext 在创建的同时伴随的那一份独特的信息。后面会看到，这个 Configuration 是很特殊的，在 Configuration 更新的时候不会被新传入的 Configuration 覆盖掉。

### 1.1.3 ResourcesManager#getOrCreateActivityResourcesStructLocked

有了一些基础知识后再去看 ResourcesManager#getOrCreateActivityResourcesStructLocked 方法：

```
    private ActivityResources getOrCreateActivityResourcesStructLocked(
            @NonNull IBinder activityToken) {
        ActivityResources activityResources = mActivityResourceReferences.get(activityToken);
        if (activityResources == null) {
            activityResources = new ActivityResources();
            mActivityResourceReferences.put(activityToken, activityResources);
        }
        return activityResources;
    }


```

ResourcesManager#getOrCreateActivityResourcesStructLocked 尝试从 ResourcesManager 的成员变量 mActivityResourceReferences 中根据 activityToken 去取相应的 ActivityResources 对象，如果找不到，就说明这是第一次为这个 Activity 创建相关联的 Resources，那么新创建一个 ActivityResources 对象，并且将键值对 <activityToken, activityResources> 保存在 ResourcesManager 的成员变量 mActivityResourceReferences 中，mActivityResourceReferences 定义是：

```
    
    @UnsupportedAppUsage
    private final WeakHashMap<IBinder, ActivityResources> mActivityResourceReferences =
            new WeakHashMap<>();


```

可以看到，ResourcesManager 通过 mActivityResourceReferences 记录了每一个 Activity 对应的所有 ActivityResources 对象。

接下来其实就是上面所讲内容的具体实践：

```
        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
                : new Resources(classLoader);
        resources.setImpl(impl);
        resources.setCallbacks(mUpdateCallbacks);
        ActivityResource activityResource = new ActivityResource();
        activityResource.resources = new WeakReference<>(resources,
                activityResources.activityResourcesQueue);
        activityResource.overrideConfig.setTo(initialOverrideConfig);
        activityResource.overrideDisplayId = overrideDisplayId;
        activityResources.activityResources.add(activityResource);


```

1)、创建了一个 Resources 对象。

2)、调用 Resources#setImpl 将传入的 ResourcesImpl 对象设置为这个新创建的 Resources 对象的 mImpl 成员变量。

3)、每一次为 Activity 创建 Resources，都需要同时封装一层 ActivityResource 对象，并且将 ActivityResource 对象添加到 activityResources.activityResources 队列中。另外，这里还将 ActivityResource 对象的成员 overrideConfig 赋值为传入的 initialOverrideConfig，这是 ActivityResource.overrideConfig 唯一赋值的地方。

1.2 非 Activity 类型 Resources 的创建
-------------------------------

```
    private @NonNull Resources createResourcesLocked(@NonNull ClassLoader classLoader,
            @NonNull ResourcesImpl impl, @NonNull CompatibilityInfo compatInfo) {
        cleanupReferences(mResourceReferences, mResourcesReferencesQueue);
        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
                : new Resources(classLoader);
        resources.setImpl(impl);
        resources.setCallbacks(mUpdateCallbacks);
        mResourceReferences.add(new WeakReference<>(resources, mResourcesReferencesQueue));
        if (DEBUG) {
            Slog.d(TAG, "- creating new ref=" + resources);
            Slog.d(TAG, "- setting ref=" + resources + " with impl=" + impl);
        }
        return resources;
    }


```

相较于 Activity 类型 Resources 创建的过程，非 Activity 类型 Resources 的创建就显得比较简单，这里看到 ResourcesManager 会把新创建的非 Activity 类型 Resources 添加到成员变量 mResourceReferences：

```
    
    @UnsupportedAppUsage
    private final ArrayList<WeakReference<Resources>> mResourceReferences = new ArrayList<>();


```

就像把 mActivityResourceReferences 记录的是 Activity 类型 Resources 一样，mResourceReferences 记录的是所有非 Activity 类型 Resources。

1.3 小结
------

1)、Activity 类型 Resources 通过 ResourcesManager#createResourcesForActivityLocked 创建，非 Activity 类型 Resources 通过 ResourcesManager#createResourcesLocked 创建。

Activity 类型 Resources 创建流程有两个，非 Activity 类型 Resources 创建流程只有一个。

调用堆栈总结为：

```
Activity类型Resources创建流程之一：
...
    -> ContextImpl#createActivityContext
    或
    -> ContextImpl#createWindowContextResources
        -> ResourcesManager#createBaseTokenResources
            -> ResourcesManager#createResourcesForActivity
                -> ResourcesManager#createResourcesForActivityLocked
            
Activity类型Resources和非Activity类型Resources共同的创建流程：
...
    -> ResourcesManager#getResources
    （根据此处传入的activityToken是否为null来决定是调用ResourcesManager#createResourcesForActivity还是ResourcesManager#createResources）
        -> ResourcesManager#createResourcesForActivity（Activity类型Resources）
            -> ResourcesManager#createResourcesForActivityLocked
        -> ResourcesManager#createResources（非Activity类型Resources）
            -> ResourcesManager#createResourcesLocked


```

2)、ResourcesManager 为每一个 Activity 都创建了一个 ActivityResources 对象，用来保存所有和这个 Activity 有联系的 Resources，将每一个 Resource 封装为了 ActivityResource 对象。

3)、ActivityResource 的成员变量 overrideConfig 保存了每一个 Activity 类型 Resource 在创建时期的初始 Configuration，它代表了每一个 Activity = 类型 Resource 的独特部分，在 Activity 类型 Resources 更新的时候有着重要作用。

4）、ResourcesManager 成员变量 mActivityResourceReferences 保存的是所有 Activity 类型的 Resources，成员变量 mResourceReferences 保存了所有非 Activity 类型 Resources。

2.1 所有 Resources 的更新
--------------------

所有 Resources 的更新在 ResourcesManager#applyConfigurationToResources。

调用堆栈第一处：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/827c5e6613814d69b8a78eff3f213cf7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这一处涉及到了 ConfigurationChangeItem，ConfigurationChangeItem 使用的地方在 WindowProcessController：

```
ActivityTaskManagerService
    -> WindowProcessController
        -> WindowProcessController
            -> WindowProcessController

ActivityRecord
	-> WindowProcessController


```

第二处：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9c9d3ab81b34fbab552fd1f9eff3cd2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

主要的还是第一处。

接下来分析具体代码。

```
    
    public final boolean applyConfigurationToResources(@NonNull Configuration config,
            @Nullable CompatibilityInfo compat, @Nullable DisplayAdjustments adjustments) {
        synchronized (mLock) {
            try {
                Trace.traceBegin(Trace.TRACE_TAG_RESOURCES,
                        "ResourcesManager#applyConfigurationToResources");
                if (!mResConfiguration.isOtherSeqNewer(config) && compat == null) {
                    if (DEBUG || DEBUG_CONFIGURATION) {
                        Slog.v(TAG, "Skipping new config: curSeq="
                                + mResConfiguration.seq + ", newSeq=" + config.seq);
                    }
                    return false;
                }
                int changes = mResConfiguration.updateFrom(config);
                
                Configuration tmpConfig = new Configuration();
                for (int i = mResourceImpls.size() - 1; i >= 0; i--) {
                    ResourcesKey key = mResourceImpls.keyAt(i);
                    WeakReference<ResourcesImpl> weakImplRef = mResourceImpls.valueAt(i);
                    ResourcesImpl r = weakImplRef != null ? weakImplRef.get() : null;
                    if (r != null) {
                        applyConfigurationToResourcesLocked(config, compat, tmpConfig, key, r);
                    } else {
                        mResourceImpls.removeAt(i);
                    }
                }
                return changes != 0;
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
            }
        }
    }


```

将全局 configuration 的更新应用到受 ResourcesManager 管理的 Resources 对象上。

挑几处重要的地方分析。

### 2.1.1 更新 ResourcesManager.mResConfiguration

```
                if (!mResConfiguration.isOtherSeqNewer(config) && compat == null) {
                    if (DEBUG || DEBUG_CONFIGURATION) {
                        Slog.v(TAG, "Skipping new config: curSeq="
                                + mResConfiguration.seq + ", newSeq=" + config.seq);
                    }
                    return false;
                }
                int changes = mResConfiguration.updateFrom(config);


```

这里 ResourcesManager.mResConfiguration 储存了每一次传入的 Configuration 的值，如果这一次传入的 Configuration 并不比 ResourcesManager.mResConfiguration 新，那么跳过这次更新。ResourcesManager.mResConfiguration 的定义是：

```
    
    @UnsupportedAppUsage
    private final Configuration mResConfiguration = new Configuration();


```

ResourcesManager.mResConfiguration 作为全局 Configuration 来使用，所有 Resources 对象基于它来生成。这个具体怎么理解，可以看后续的分析。

另外这也是 ResourcesManager.mResConfiguration 更新的唯一地方。

### 2.1.2 ResourcesImpl 类和 ResourcesManager 成员变量 mResourceImpls

```
                for (int i = mResourceImpls.size() - 1; i >= 0; i--) {
                    ResourcesKey key = mResourceImpls.keyAt(i);
                    WeakReference<ResourcesImpl> weakImplRef = mResourceImpls.valueAt(i);
                    ResourcesImpl r = weakImplRef != null ? weakImplRef.get() : null;
                    if (r != null) {
                        applyConfigurationToResourcesLocked(config, compat, tmpConfig, key, r);
                    } else {
                        mResourceImpls.removeAt(i);
                    }
                }


```

接下来的内容就是遍历 ResourcesManager.mResourceImpls 中的每一个 ResourcesImpl 对象进行更新操作。

在分析 applyConfigurationToResourcesLocked 方法是如何更新 ResourcesManager.mResourceImpls 中的每一个 ResourcesImpl 对象之前，需要对 ResourcesImpl 类和 ResourcesManager 的成员变量 mResourceImpls 有一定的了解。

mResourceImpls，定义是：

```
    
    @UnsupportedAppUsage
    private final ArrayMap<ResourcesKey, WeakReference<ResourcesImpl>> mResourceImpls =
            new ArrayMap<>();


```

注释说，这是 ResourcesImpl 对象到他们的 Configuration 的映射。

ResourcesKey 对象从名字上来看，就是为了作为 Map 的主键存在的。ResourcesKey 对象中有一个成员变量，mOverrdieConfiguration，也是保存了一个 ResourcesKey 独有的 Configuration。

作为 value 的 ResourcesImpl 类，从名字上也可以看出，是 Resources 的实现类，真正持有 Configuration 和其他信息那个角色。

### 2.1.3 ResourcesManager.mResourceImpls 内容的填充

跟踪一下 ResourcesImpl 是如何添加到 ResourcesManager.mResourceImpls 中的。

ResourcesManager.mResourceImpls 添加数据是在 ResourcesManager#findOrCreateResourcesImplForKeyLocked 方法中：

```
    
    private @Nullable ResourcesImpl findOrCreateResourcesImplForKeyLocked(
            @NonNull ResourcesKey key, @Nullable ApkAssetsSupplier apkSupplier) {
        ResourcesImpl impl = findResourcesImplForKeyLocked(key);
        if (impl == null) {
            impl = createResourcesImpl(key, apkSupplier);
            if (impl != null) {
                mResourceImpls.put(key, new WeakReference<>(impl));
            }
        }
        return impl;
    }


```

每次想获得一个 ResourcesImpl 对象的时候，先从 ResourcesManager.mResourceImpls 中去寻找与传入的 ResourcesKey 对象对应的那个 ResourcesImpl 对象，如果找不到，那么就会调用 ResourcesImpl#createResourcesImpl 去为当前 ResourcesKey 对象创建一个 ResourcesImpl 对象，然后把这个键值对添加到 ResourcesManager.mResourceImpls 中。这里也说明了，ResourcesKey 和 ResourcesImpl 是一一对应的。

在 ResourcesManager#findOrCreateResourcesImplForKeyLocked 方法的调用情况的时候，看到以下两种调用地点：

1）、ResourcesManager#createResourcesForActivity

```
    @Nullable
    private Resources createResourcesForActivity(@NonNull IBinder activityToken,
            @NonNull ResourcesKey key, @NonNull Configuration initialOverrideConfig,
            @Nullable Integer overrideDisplayId, @NonNull ClassLoader classLoader,
            @Nullable ApkAssetsSupplier apkSupplier) {
        synchronized (mLock) {
            if (DEBUG) {
                Throwable here = new Throwable();
                here.fillInStackTrace();
                Slog.w(TAG, "!! Get resources for activity=" + activityToken + " key=" + key, here);
            }

            ResourcesImpl resourcesImpl = findOrCreateResourcesImplForKeyLocked(key, apkSupplier);
            if (resourcesImpl == null) {
                return null;
            }

            return createResourcesForActivityLocked(activityToken, initialOverrideConfig,
                    overrideDisplayId, classLoader, resourcesImpl, key.mCompatInfo);
        }
    }


```

这个正好对应的 Activity 类型 Resources 的创建流程，提前通过 ResourcesManager#findOrCreateResourcesImplForKeyLocked 方法获取一个 ResourcesImpl 对象，然后在 ResourcesManager#createResourcesForActivityLocked 方法中，再通过 Resources#setImpl 让新创建的 Activity 类型 Resources 保存这个 ResourcesImpl 对象。

2)、ResourcesManager#createResources

```
    
    @Nullable
    private Resources createResources(@NonNull ResourcesKey key, @NonNull ClassLoader classLoader,
            @Nullable ApkAssetsSupplier apkSupplier) {
        synchronized (mLock) {
            if (DEBUG) {
                Throwable here = new Throwable();
                here.fillInStackTrace();
                Slog.w(TAG, "!! Create resources for key=" + key, here);
            }

            ResourcesImpl resourcesImpl = findOrCreateResourcesImplForKeyLocked(key, apkSupplier);
            if (resourcesImpl == null) {
                return null;
            }

            return createResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
        }
    }


```

这个正好对应的非 Activity 类型 Resources 的创建流程，提前通过 ResourcesManager#findOrCreateResourcesImplForKeyLocked 方法获取一个 ResourcesImpl 对象，然后在 ResourcesManager#createResourcesLocked 方法中，再通过 Resources#setImpl 让新创建的非 Activity 类型 Resources 保存这个 ResourcesImpl 对象。

从以上内容可以获取几点重要信息：

*   ResourcesImpl 和 Resources 是一对多的关系，多个 Resources 内部的 mResourcesImpl 成员变量可能是同一个。
*   不管是 Activity 类型 Resources，还是非 Activity 类型 Resources，它们内部的 mResourcesImpl 成员变量都被记录在 ResourcesManager 的成员变量 mResourceImpls 中。那么更新 mResourceImpls 表中记录的所有 ResourcesImpl 对象中保存的 Configuration，其实就是更新所有 Resources 中保存的 Configuration。

### 2.1.4 ResourcesImpl 对象的创建

再看下创建 ResourcesImpl 对象的 createResourcesImpl 方法：

```
    private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key,
            @Nullable ApkAssetsSupplier apkSupplier) {
        final AssetManager assets = createAssetManager(key, apkSupplier);
        if (assets == null) {
            return null;
        }
        final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
        daj.setCompatibilityInfo(key.mCompatInfo);
        final Configuration config = generateConfig(key);
        final DisplayMetrics displayMetrics = getDisplayMetrics(generateDisplayId(key), daj);
        final ResourcesImpl impl = new ResourcesImpl(assets, displayMetrics, config, daj);
        if (DEBUG) {
            Slog.d(TAG, "- creating impl=" + impl + " with key: " + key);
        }
        return impl;
    }


```

这里有一点需要注意的是，这里创建 ResourcesImpl 用的 Configuration 是通过 ResourcesManager#generateConfig 生成的，上面提到的 ResourcesManager 的成员变量 mResConfiguration 的作用就是在这里，mResConfiguration 可以通过 ResourcesManager#getConfiguration 方法得到：

```
    public Configuration getConfiguration() {
        synchronized (mLock) {
            return mResConfiguration;
        }
    }


```

通过上面的分析可知，ResourcesManager 每一次创建 Resources 对象时，都需要提前从 mResourceImpls 中获取一个 ResourcesImpl 对象，而 mResourceImpls 中的所有 ResourcesImpl 对象都是通过 ResourcesManager#createResourcesImpl 添加的。ResourcesManager#createResourcesImpl 方法在创建 ResourcesImpl 对象的时候，需要向 ResourcesImpl 的构造方法中传入一个 Configuration，这个 Configuration 对象是通过 ResourcesManager#generateConfig 方法生成的，而 ResourcesManager#generateConfig 返回的 Configuration 对象，则是基于 ResourcesManager.mResConfiguration 创建的：

```
    private Configuration generateConfig(@NonNull ResourcesKey key) {
        Configuration config;
        final boolean hasOverrideConfig = key.hasOverrideConfiguration();
        if (hasOverrideConfig) {
            config = new Configuration(getConfiguration());
            config.updateFrom(key.mOverrideConfiguration);
            if (DEBUG) Slog.v(TAG, "Applied overrideConfig=" + key.mOverrideConfiguration);
        } else {
            config = getConfiguration();
        }
        return config;
    }


```

以 ResourcesManager.mResConfiguration 为 base，update 上 ResourcesKey.mOverrideConfiguration（如果 Resourceskey 不为 null 的话）。

这段逻辑展示了 mResConfiguration 的作用，就如 mResConfiguration 的注释所说的那样，所有的 Resources 的创建都是基于 ResourcesManager.mResConfiguration 的。

### 2.1.5 更新 ResourcesImpl 对象

可以继续往下分析了：

```
                for (int i = mResourceImpls.size() - 1; i >= 0; i--) {
                    ResourcesKey key = mResourceImpls.keyAt(i);
                    WeakReference<ResourcesImpl> weakImplRef = mResourceImpls.valueAt(i);
                    ResourcesImpl r = weakImplRef != null ? weakImplRef.get() : null;
                    if (r != null) {
                        applyConfigurationToResourcesLocked(config, compat, tmpConfig, key, r);
                    } else {
                        mResourceImpls.removeAt(i);
                    }
                }


```

这里遍历 ResourcesManager.mResourceImpls，然后对于每一个 ResourcesImpl 对象，调用 applyConfigurationToResourcesLocked 去更新。

```
    private void applyConfigurationToResourcesLocked(@NonNull Configuration config,
            @Nullable CompatibilityInfo compat, Configuration tmpConfig,
            ResourcesKey key, ResourcesImpl resourcesImpl) {
        if (DEBUG || DEBUG_CONFIGURATION) {
            Slog.v(TAG, "Changing resources "
                    + resourcesImpl + " config to: " + config);
        }

        tmpConfig.setTo(config);
        if (key.hasOverrideConfiguration()) {
            tmpConfig.updateFrom(key.mOverrideConfiguration);
        }

        
        
        
        DisplayAdjustments daj = resourcesImpl.getDisplayAdjustments();
        if (compat != null) {
            daj = new DisplayAdjustments(daj);
            daj.setCompatibilityInfo(compat);
        }
        daj.setConfiguration(tmpConfig);
        DisplayMetrics dm = getDisplayMetrics(generateDisplayId(key), daj);

        resourcesImpl.updateConfiguration(tmpConfig, dm, compat);
    }


```

重点看下 Configuration 是怎么更新的。

将本次传入的 Configuration 作为 base，然后再根据 ResourcesKey.mOverrideConfiguration 进行 update，得到最终的 Configuration，然后再去调用 ResourcesImpl#updateConfiguration 去更新 ResourcesImpl，最终在 ResourcesImpl#calConfigChanges 中，ResourcesImpl.mConfiguraton 得到了更新。

这里有一个重要的点，如果当前 ResourcesImp 对象对应的 ResourcesKey 中保存的 Configuration 不是 Configuration.EMPTY，那么最终得到的 tmpConfig 是经过 ResourcesKey.mOverrideConfiguration 的 update 的，不仅仅只是本次更新 Configuration 的传参 config 的值。

想象这样一种情况，比如这里传入的 config.screenWidthDp 为 500dp，但是 key.mOverrideConfiguration.screenWidthDp 为 600dp，那么最终 tmpConfig 中的 screenWidthDp 还是 600dp，导致 ResourcesImpl.mConfiguration.screenWidthDp 还是 600dp，没有更新为传入的 500dp。也就是说，如果当前 ResourcesImpl 对应了一个 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY 的 ResourcesKey，那么很可能经过这一次 ResourcesManager.applyConfigurationToResourcesLocked，当前 ResourcesImpl 最终没有得到更新。

### 2.1.6 小结

经过 ResourcesManager#applyConfigurationToResources，最终的结果是，所有 ResourcesImpl 对象中保存的 Configuration 对象得到更新，当然由于 ResourcesKey 中保存的 Configuration 不为 Configuration.EMPTY，最终某些 ResourcesImpl 的 Configuration 也可能没有得到更新。

2.2 Activity 类型 Resources 的更新
-----------------------------

Activity Resources 的更新在 ResourcesManager#updateResourcesForActivity。

调用堆栈是：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/779a0bc2edfe480e846ca90d2584d533~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

看到是通过 ActivityConfigurationChangeItem 传递信息的，ActivityConfigurationChangeItem 在服务端用的地方在 ActivityRecord#scheduleConfigurationChanged：

```
    private void scheduleConfigurationChanged(Configuration config) {
        if (!attachedToProcess()) {
            ProtoLog.w(WM_DEBUG_CONFIGURATION, "Can't report activity configuration "
                    + "update - client not running, activityRecord=%s", this);
            return;
        }
        try {
            ProtoLog.v(WM_DEBUG_CONFIGURATION, "Sending new config to %s, "
                    + "config: %s", this, config);
            mAtmService.getLifecycleManager().scheduleTransaction(app.getThread(), appToken,
                    ActivityConfigurationChangeItem.obtain(config));
        } catch (RemoteException e) {
            
        }
    }


```

接下来看具体代码。

```
    
    public void updateResourcesForActivity(@NonNull IBinder activityToken,
            @Nullable Configuration overrideConfig, int displayId) {
        try {
            
            synchronized (mLock) {
                final ActivityResources activityResources =
                        getOrCreateActivityResourcesStructLocked(activityToken);

                boolean movedToDifferentDisplay = activityResources.overrideDisplayId != displayId;
                if (Objects.equals(activityResources.overrideConfig, overrideConfig)
                        && !movedToDifferentDisplay) {
                    
                    return;
                }

                
                
                final Configuration oldConfig = new Configuration(activityResources.overrideConfig);

                
                if (overrideConfig != null) {
                    activityResources.overrideConfig.setTo(overrideConfig);
                } else {
                    activityResources.overrideConfig.unset();
                }

                

                
                final int refCount = activityResources.activityResources.size();
                for (int i = 0; i < refCount; i++) {
                    final ActivityResource activityResource =
                            activityResources.activityResources.get(i);

                    final Resources resources = activityResource.resources.get();
                    if (resources == null) {
                        continue;
                    }

                    final ResourcesKey newKey = rebaseActivityOverrideConfig(activityResource,
                            overrideConfig, displayId);
                    if (newKey == null) {
                        continue;
                    }

                    
                    
                    final ResourcesImpl resourcesImpl =
                            findOrCreateResourcesImplForKeyLocked(newKey);
                    if (resourcesImpl != null && resourcesImpl != resources.getImpl()) {
                        
                        
                        resources.setImpl(resourcesImpl);
                    }
                }
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
        }
    }


```

挑几点重要的分析。

### 2.2.1 遍历前的准备

```
                final ActivityResources activityResources =
                        getOrCreateActivityResourcesStructLocked(activityToken);

                boolean movedToDifferentDisplay = activityResources.overrideDisplayId != displayId;
                if (Objects.equals(activityResources.overrideConfig, overrideConfig)
                        && !movedToDifferentDisplay) {
                    
                    return;
                }

                
                
                final Configuration oldConfig = new Configuration(activityResources.overrideConfig);

                
                if (overrideConfig != null) {
                    activityResources.overrideConfig.setTo(overrideConfig);
                } else {
                    activityResources.overrideConfig.unset();
                }


```

1)、判断本次 ResourcesManager#updateResourcesForActivity 传入的新的 Configuration 和上一次传入的是否一致，如果一致那本次就不用更新了。这里是 ActivityResources.overideConfig 唯一更新的地方。

2)、获取到 Activity 的 activityResources.overrideConfig 的一份拷贝，因为现在我们要将 activityResources.overrideConfig 更新为传入的 overrideConfig，但是后续还要用到更新前的 activityResources.overrideConfig。

3)、将 activityResources.overrideConfig 更新为传入的 overrideConfig。

### 2.2.2 遍历当前 Activity 对应的 ActivityResources 对象的 activityResources 成员

```
                
                final int refCount = activityResources.activityResources.size();
                for (int i = 0; i < refCount; i++) {
                    final ActivityResource activityResource =
                            activityResources.activityResources.get(i);
                    final Resources resources = activityResource.resources.get();
                    if (resources == null) {
                        continue;
                    }
                    final ResourcesKey newKey = rebaseActivityOverrideConfig(activityResource,
                            overrideConfig, displayId);
                    if (newKey == null) {
                        continue;
                    }
                    
                    
                    final ResourcesImpl resourcesImpl =
                            findOrCreateResourcesImplForKeyLocked(newKey);
                    if (resourcesImpl != null && resourcesImpl != resources.getImpl()) {
                        
                        
                        resources.setImpl(resourcesImpl);
                    }
                }


```

这里可以看到和 ResourcesManager#applyConfigurationToResources 的差别，ResourcesManager#applyConfigurationToResources 是遍历 ResourcesManager.mResourceImpls，而这里只遍历和当前 Activity 相关联的 Resources 对象，也就是保存在 ActivityResources.activityResources 队列中的所有 Resources。

这里先通过 ResourcesManager#rebaseActivityOverrideConfig 生成一个新 ResourcesKey，这个 ResourcesKey.mOverrideConfiguration 就是基于传入的新 Configuration 和其他因素综合生成的最终的 Configuration，然后再跟据这个 ResourcesKey 从 ResourcesManager.mResourceImpls 寻找一个对应的 ResourcesImpl 对象，找不到就创建一个，然后把新的 ResourcesImpl 对象通过 Resources#setImpl 赋值给 Resources.mResourcesImpl 成员变量，从而完成对 Resources 对象的更新。

重点看一下 ResourcesManager#rebaseActivityOverrideConfig 方法。

### 2.2.3 ResourcesManager#rebaseActivityOverrideConfig

```
    
    @Nullable
    private ResourcesKey rebaseActivityOverrideConfig(@NonNull ActivityResource activityResource,
            @Nullable Configuration newOverrideConfig, int displayId) {
        final Resources resources = activityResource.resources.get();
        if (resources == null) {
            return null;
        }

        
        
        final ResourcesKey oldKey = findKeyForResourceImplLocked(resources.getImpl());
        if (oldKey == null) {
            Slog.e(TAG, "can't find ResourcesKey for resources impl="
                    + resources.getImpl());
            return null;
        }

        
        final Configuration rebasedOverrideConfig = new Configuration();
        if (newOverrideConfig != null) {
            rebasedOverrideConfig.setTo(newOverrideConfig);
        }

        

        final boolean hasOverrideConfig =
                !activityResource.overrideConfig.equals(Configuration.EMPTY);
        if (hasOverrideConfig) {
            rebasedOverrideConfig.updateFrom(activityResource.overrideConfig);
        }

        

        
        final ResourcesKey newKey = new ResourcesKey(oldKey.mResDir,
                oldKey.mSplitResDirs, oldKey.mOverlayPaths, oldKey.mLibDirs,
                displayId, rebasedOverrideConfig, oldKey.mCompatInfo, oldKey.mLoaders);

        if (DEBUG) {
            Slog.d(TAG, "rebasing ref=" + resources + " from oldKey=" + oldKey
                    + " to newKey=" + newKey + ", displayId=" + displayId);
        }

        return newKey;
    }


```

1)、首先调用 ResourcesManager#findKeyForResourceImplLocked 在 ResourcesManager.mResourcesImpl 中查找当前 ResourcesImpl 有没有对应的 ResourcesKey，没有直接返回。

2)、接下来为此次的要创建的新的 ResourcesKey 生成新的 Configuration：rebasedOverrideConfig，它把传入的 newOverrideConfig 作为 base，然后 update 上 ActivityResource.overrideConfig 得到终值。

3)、最后再根据最终的 rebasedOverrideConfig 去创建新的 ResourcesKey。

这里和 ResourcesManager#applyConfigurationForResources 中更新 Resources 的思想是一样的，ActivityResource.overrideConfig 唯一赋值的时间点是在 Activity 类型 Resources 创建的时候，那么 ActivityResource.overrideConfig 就保存了当前 Activity 类型 Resouces 创建的时候的初始 Configuration，抽象点说就是这个 Configuration 是这个 Resources 与生俱来的，是这个 Resources 不同于其他 Resources 的重要特点，任何时候都不应该被丢弃，因此需要放在最后的时候再 update 上去，这样通过这个 Resources 获取的 Configuration 总是会包含这个 Resources 的特点部分。

### 2.2.4 小结

经过 ResourcesManager#updateResourcesForActivity，最终的结果为，所有的 Activity 类型的 Resources 的 mResourcesImpl 成员变量被替换，这个过程中可能伴随着新的 ResourcesImpl 对象的创建，但是 ResourcesImpl 对象中保存的 Configuration 没有更新。

这里我们通过 Demo App 的 log 来验证一下之前的纸上谈兵谈的正确不正确。

3.1 在 Launcher 界面点击 Demo App 图标启动 App
-------------------------------------

```
12-17 11:26:35.375  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@8a62ae
12-17 11:26:35.384  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@a9dfbba
12-17 11:26:35.558  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@2c79e0
12-17 11:26:35.587  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@256d33f
12-17 11:26:35.646  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@763a4f8
12-17 11:26:35.693  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@df10a2f
12-17 11:26:35.957  7133  7133 I ukynho_res: ContextImpl#constructor ---- context = android.app.ContextImpl@d7e7b3
12-17 11:26:35.535  7133  7133 I ukynho_res: Resources#constructor ---- this = android.content.res.Resources@82d2a74
12-17 11:26:35.672  7133  7133 I ukynho_res: Resources#constructor ---- this = android.content.res.Resources@c3ad9c2
12-17 11:26:35.735  7133  7133 I ukynho_res: Resources#constructor ---- this = android.content.res.Resources@ca21027
12-17 11:26:35.840  7133  7133 I ukynho_res: Resources#constructor ---- this = android.content.res.Resources@19dc86c
12-17 11:26:35.967  7133  7133 I ukynho_res: Resources#constructor ---- this = android.content.res.Resources@8df4c0f
    
12-17 11:26:35.523  7133  7133 I ukynho_res: ResourcesImpl#constructor ---- resourcesImpl = android.content.res.ResourcesImpl@765ba61
12-17 11:26:35.658  7133  7133 I ukynho_res: ResourcesImpl#constructor ---- resourcesImpl = android.content.res.ResourcesImpl@ab5eb37
12-17 11:26:35.724  7133  7133 I ukynho_res: ResourcesImpl#constructor ---- resourcesImpl = android.content.res.ResourcesImpl@7863c28


```

首先我们看到总共创建了 7 个 Context，但是只创建了 5 个 Resources 对象，ResourcesImpl 更少，只有 3 个。

对应关系为：

```
12-17 11:26:35.377  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@8a62ae  resources = android.content.res.Resources@dfa5116
12-17 11:26:35.540  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@a9dfbba     resources = android.content.res.Resources@82d2a74
12-17 11:26:35.556  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@2c79e0  resources = android.content.res.Resources@82d2a74
12-17 11:26:35.589  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@256d33f     resources = android.content.res.Resources@82d2a74
12-17 11:26:35.675  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@763a4f8     resources = android.content.res.Resources@c3ad9c2
12-17 11:26:35.742  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@df10a2f     resources = android.content.res.Resources@ca21027
12-17 11:26:35.955  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@d7e7b3  resources = android.content.res.Resources@82d2a74
12-17 11:26:35.971  7133  7133 I ukynho_res: ContextImpl#setResources ---- context = android.app.ContextImpl@d7e7b3  resources = android.content.res.Resources@8df4c0f
12-17 11:26:35.537  7133  7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@82d2a74    mResourcesImpl = null   impl = android.content.res.ResourcesImpl@765ba61
12-17 11:26:35.673  7133  7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@c3ad9c2    mResourcesImpl = null   impl = android.content.res.ResourcesImpl@ab5eb37
12-17 11:26:35.737  7133  7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@ca21027    mResourcesImpl = null   impl = android.content.res.ResourcesImpl@7863c28
12-17 11:26:35.842  7133  7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@19dc86c    mResourcesImpl = null   impl = android.content.res.ResourcesImpl@765ba61
12-17 11:26:35.969  7133  7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@8df4c0f    mResourcesImpl = null   impl = android.content.res.ResourcesImpl@765ba61
12-17 11:26:35.515  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{ mHash=593d1817 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut ?orien ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}}
12-17 11:26:35.515  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = null
12-17 11:26:35.533  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- new resourcesIml = android.content.res.ResourcesImpl@765ba61
12-17 11:26:35.653  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{ mHash=2a5c3093 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir sw360dp w360dp h722dp 480dpi nrml long ?ldr ?wideColorGamut port ?uimode ?night -touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}}
12-17 11:26:35.653  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = null
12-17 11:26:35.670  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- new resourcesIml = android.content.res.ResourcesImpl@ab5eb37
12-17 11:26:35.710  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{ mHash=7d7a14ea mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}}
12-17 11:26:35.710  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = null
12-17 11:26:35.732  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- new resourcesIml = android.content.res.ResourcesImpl@7863c28
12-17 11:26:35.838  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{ mHash=593d1817 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut ?orien ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}}
12-17 11:26:35.839  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = android.content.res.ResourcesImpl@765ba61
12-17 11:26:35.963  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{ mHash=593d1817 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut ?orien ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}}
12-17 11:26:35.964  7133  7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = android.content.res.ResourcesImpl@765ba61


```

整理成表格：

<table><thead><tr><th>Context</th><th>Resources</th><th>ResourcesImpl</th><th>ResourcesKey</th></tr></thead><tbody><tr><td>ContextImpl@8a62ae</td><td>Resources@dfa5116</td><td></td><td></td></tr><tr><td>ContextImpl@a9dfbba</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@2c79e0</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@256d33f</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@763a4f8</td><td>Resources@c3ad9c2</td><td>ResourcesImpl@ab5eb37</td><td>ResourcesKey{mHash=2a5c3093......}</td></tr><tr><td>ContextImpl@df10a2f</td><td>Resources@ca21027</td><td>ResourcesImpl@7863c28</td><td>ResourcesKey{mHash=7d7a14ea......}</td></tr><tr><td>ContextImpl@d7e7b3</td><td>Resources@8df4c0f</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td></td><td>Resources@19dc86c</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr></tbody></table>

有一些 Log 可能没有打印出来，导致表格不是很全，但是无伤大雅。

### 3.1.1 Context 和 Resources 的对应关系

Resources 对于 Context 来说是一对多的关系，多个 Context 可能对应同一个 Resources，在为 Context 创建 Resources 的时候，会先去寻找已经存在 Resources 中是否有合适的可以复用：

```
    @Nullable
    private Resources findResourcesForActivityLocked(@NonNull IBinder targetActivityToken,
            @NonNull ResourcesKey targetKey, @NonNull ClassLoader targetClassLoader) {
        ActivityResources activityResources = getOrCreateActivityResourcesStructLocked(
                targetActivityToken);
        final int size = activityResources.activityResources.size();
        for (int index = 0; index < size; index++) {
            ActivityResource activityResource = activityResources.activityResources.get(index);
            Resources resources = activityResource.resources.get();
            ResourcesKey key = resources == null ? null : findKeyForResourceImplLocked(
                    resources.getImpl());
            if (key != null
                    && Objects.equals(resources.getClassLoader(), targetClassLoader)
                    && Objects.equals(key, targetKey)) {
                return resources;
            }
        }
        return null;
    }


```

遍历当前 Activity 中所有的 ResourcesImpl 对象，看这些 ResourcesImpl 对应的 ResourcesKey 对象和传入的 ResourcesKey 对象 targetKey 是否一致。如果一致，说明此时对于当前传入的 ResourcesKey 对象，已经有一个 ResourcesImp 对象与其对应了，返回这个 Resources 对象即可。

### 3.1.2 Resources 和 ResourcesImpl 的对应关系

ResourcesImpl 对于 Resources 来说也是一对多的关系，多个 Resources 可能对应同一个 ResourcesImpl，在为 Resources 创建 ResourcesImpl 的时候，会先去寻找已经存在 ResourcesImpl 中是否有合适的可以复用：

```
    
    private @Nullable ResourcesImpl findOrCreateResourcesImplForKeyLocked(
            @NonNull ResourcesKey key, @Nullable ApkAssetsSupplier apkSupplier) {
        ResourcesImpl impl = findResourcesImplForKeyLocked(key);
        if (impl == null) {
            impl = createResourcesImpl(key, apkSupplier);
            if (impl != null) {
                mResourceImpls.put(key, new WeakReference<>(impl));
            }
        }
        return impl;
    }


```

如果已经有一个现有的 ResourcesImpl 对象与传入的 ResourcesKey 对应，那么就不需要再创建一个新的 ResourcesImpl 对象了。

### 3.1.3 比较的关键：ResourcesKey

这里看到，以上两种情况，决定是否复用的逻辑就是比较 Resourceskey。

从之前的介绍中，我们知道创建 Resources 有两个入口，ResourcesManager#getResources 和 ResourcesManager#createBaseTokenResources，这两个方法有一个共同点：

```
    public @Nullable Resources createBaseTokenResources(@NonNull IBinder token,
            @Nullable String resDir,
            @Nullable String[] splitResDirs,
            @Nullable String[] legacyOverlayDirs,
            @Nullable String[] overlayPaths,
            @Nullable String[] libDirs,
            int displayId,
            @Nullable Configuration overrideConfig,
            @NonNull CompatibilityInfo compatInfo,
            @Nullable ClassLoader classLoader,
            @Nullable List<ResourcesLoader> loaders) {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_RESOURCES,
                    "ResourcesManager#createBaseActivityResources");
            final ResourcesKey key = new ResourcesKey(
                    resDir,
                    splitResDirs,
                    combinedOverlayPaths(legacyOverlayDirs, overlayPaths),
                    libDirs,
                    displayId,
                    overrideConfig,
                    compatInfo,
                    loaders == null ? null : loaders.toArray(new ResourcesLoader[0]));
            ......
            }
        ......
    }

    public Resources getResources(
            @Nullable IBinder activityToken,
            @Nullable String resDir,
            @Nullable String[] splitResDirs,
            @Nullable String[] legacyOverlayDirs,
            @Nullable String[] overlayPaths,
            @Nullable String[] libDirs,
            @Nullable Integer overrideDisplayId,
            @Nullable Configuration overrideConfig,
            @NonNull CompatibilityInfo compatInfo,
            @Nullable ClassLoader classLoader,
            @Nullable List<ResourcesLoader> loaders) {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "ResourcesManager#getResources");
            final ResourcesKey key = new ResourcesKey(
                    resDir,
                    splitResDirs,
                    combinedOverlayPaths(legacyOverlayDirs, overlayPaths),
                    libDirs,
                    overrideDisplayId != null ? overrideDisplayId : INVALID_DISPLAY,
                    overrideConfig,
                    compatInfo,
                    loaders == null ? null : loaders.toArray(new ResourcesLoader[0]));
            ......
            }
        ......
    }


```

都传入了一个 Configuration 类型的 overrideConfig 对象，并且基于这个 overrideConfig 对象生成了一个 ResourcesKey 对象，上面说了两种情况，比较的也就是这个 ResourcesKey 对象和 ResourcesManager.mResourceImpls 中保存的 ResourcesKey 对象。这说明了创建 Resources 的时候，会基于传入的 overrideConfig 生成一个 ResourcesKey，这个 ResourcesKey 有独特性，在一定程度上体现了这个 Resources 的特点，至于具体 ResourcesKey 是怎么比较的，先挖个坑。

### 3.1.4 重点关注的两个 Resources 和 ResourcesKey

在 Demo App 启动的过程中，总共创建了 3 个 ResourcesKey，其中有一个的 mOverrideConfiguration 成员是 Configuration.EMPTY，剩下两个分别是：

<table><thead><tr><th>Context</th><th>Resources</th><th>ResourcesImpl</th><th>ResourcesKey</th></tr></thead><tbody><tr><td>ContextImpl@8a62ae</td><td>Resources@dfa5116</td><td></td><td></td></tr><tr><td>ContextImpl@a9dfbba</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@2c79e0</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@256d33f</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td><strong>ContextImpl@763a4f8</strong></td><td><strong>Resources@c3ad9c2</strong></td><td><strong>ResourcesImpl@ab5eb37</strong></td><td><strong>ResourcesKey{mHash=2a5c3093......}</strong></td></tr><tr><td><strong>ContextImpl@df10a2f</strong></td><td><strong>Resources@ca21027</strong></td><td><strong>ResourcesImpl@7863c28</strong></td><td><strong>ResourcesKey{mHash=7d7a14ea......}</strong></td></tr><tr><td>ContextImpl@d7e7b3</td><td>Resources@8df4c0f</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td></td><td>Resources@19dc86c</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr></tbody></table>

ResourcesKey{mHash=2a5c3093 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir sw360dp w360dp h722dp 480dpi nrml long ?ldr ?wideColorGamut port ?uimode ?night -touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}}

对应的 ContextImpl@763a4f8 通过 ContextImpl#createSystemUiContext 创建，对应的 Resources@c3ad9c2 通过 ResourcesManager#createResourcesLocked 得到，这就是我个人定义的非 Activity 类型 Resources。

ResourcesKey{mHash=7d7a14ea mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}}

对应的 ContextImpl@df10a2f 通过 ContextImpl#createActivityContext 创建，对应的 Resources@ca21027 通过 ResourcesManager#createResourcesForActivityLocked 得到，这就是我个人定义的 Activity 类型 Resources。

3.2 从竖屏转到横屏
-----------

分别有一次 ResourcesManager#applyConfigurationToResources 和 ResourcesManager#updateResourcesForActivity。

### 3.2.1 ResourcesManager#applyConfigurationToResources

这里遍历的是 ResourcesManager.mResourceImpls，此时 ResourcesManager.mResourceImpls 有 3 个，我们这里只关注 orientation 这一个属性。

1)、先看 ResourcesImpl@7863c28，它对应了 Activity 类型 Resources。

key = ResourcesKey{mHash=7d7a14ea mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}} resourcesImpl = android.content.res.ResourcesImpl@7863c28

对应的 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY。

12-17 11:27:22.179 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_270} s.206 fontWeightAdjustment=0?ecid}

这个是传入的 newConfig，方向为横屏。

12-17 11:27:22.180 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- after update, final config = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}

但是由于 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY，update 上 ResourcesKey.mOverrideConfiguration 后方向又变为竖屏。

12-17 11:27:22.181 7133 7133 I ukynho_res: ResourcesImpl#updateConfiguration ---- resourcesImpl = android.content.res.ResourcesImpl@7863c28 newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid} oldConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}

最终传给 ResourcesImpl#updateConfiguration 的 Configuration 显示竖屏，说明由于 ResourcesKey 的存在，ResourcesImpl@7863c2 在此次 ResourcesManager#applyConfigurationToResources 没有更新成功。

2)、i = 1, ResourcesImpl@765ba61。

12-17 11:27:22.183 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResources ---- i = 1 key = ResourcesKey{ mHash=593d1817 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut **?orien** ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}} resourcesImpl = android.content.res.ResourcesImpl@765ba61

对应的 ResourcesKey.mOverrideConfiguration 为 Configuration.EMPTY。

12-17 11:27:22.184 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_270} s.206 fontWeightAdjustment=0?ecid}

这个是传入的 newConfig，方向为横屏。

12-17 11:27:22.186 7133 7133 I ukynho_res: ResourcesImpl#updateConfiguration ---- resourcesImpl = android.content.res.ResourcesImpl@765ba61 newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_270} s.206 fontWeightAdjustment=0?ecid} oldConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.80 fontWeightAdjustment=0?ecid}

由于它的 ResourcesKey 的 mOverrideConfiguration 是 Configuration.EMPTY，那么直接更新成功，看到最终传给 ResourcesImpl#updateConfiguration 的 Configuration 为横屏，更新成功。

3)、i = 0, ResourcesImpl@ab5eb37。

12-17 11:27:22.188 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResources ---- i = 0 key = ResourcesKey{ mHash=2a5c3093 mOverrideConfig={0.0 ?mcc?mnc ?localeList ?layoutDir sw360dp w360dp h722dp 480dpi nrml long ?ldr ?wideColorGamut **port** ?uimode ?night -touch ?keyb/?/? ?nav/? winConfig={mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}} resourcesImpl = android.content.res.ResourcesImpl@ab5eb37

对应的 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY。

12-17 11:27:22.189 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_270} s.206 fontWeightAdjustment=0?ecid}

这个是传入的 newConfig，方向为横屏。

12-17 11:27:22.189 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- after update, final config = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** -touch -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_270} s.206 fontWeightAdjustment=0?ecid}

但是由于 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY，update 上 ResourcesKey.mOverrideConfiguration 后方向又变为竖屏。

12-17 11:27:22.191 7133 7133 I ukynho_res: ResourcesImpl#updateConfiguration ----

resourcesImpl = android.content.res.ResourcesImpl@ab5eb37 newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** -touch -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_270} s.206 fontWeightAdjustment=0?ecid} oldConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** -touch -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.80 fontWeightAdjustment=0?ecid}

最终传给 ResourcesImpl#updateConfiguration 的 Configuration 显示竖屏，说明由于 ResourcesKey 的存在，ResourcesImpl@ab5eb37 在此次 ResourcesManager#applyConfigurationToResources 没有更新成功。

那么最终只有 i = 1, ResourcesImpl@765ba61 的情况更新成功。

### 3.2.2 ResourcesManager#updateResourcesForActivity

这里 Activity 类型 Resources 只有一个，即上一步中没有更新成功的 ResourcesImpl@7863c28 对应的 Resources@ca21027。

12-17 11:27:22.212 7133 7133 I ukynho_res: ResourcesManager#updateResourcesForActivity ---- activity = android.os.BinderProxy@4f70d2d overrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid} activityResources = android.app.ResourcesManager$ActivityResources@b9789c5 activityResources.overrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}

12-17 11:27:22.213 7133 7133 I ukynho_res: ResourcesManager#updateResourcesForActivity ---- i = 0 resources = android.content.res.Resources@ca21027

此时需要更新 Resources 对象的只有一个，Resources@ca21027。

12-17 11:27:22.214 7133 7133 I ukynho_res: ResourcesManager#rebaseActivityOverrideConfig ---- resources = android.content.res.Resources@ca21027 newOverrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid} oldKey = ResourcesKey{ mHash=7d7a14ea mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1 fontWeightAdjustment=0?ecid}}

12-17 11:27:22.215 7133 7133 I ukynho_res: ResourcesManager#rebaseActivityOverrideConfig ---- activityResource.overrideConfig = {0.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut **?orien** ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid} rebasedOverrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}

12-17 11:27:22.216 7133 7133 I ukynho_res: ResourcesManager#rebaseActivityOverrideConfig ---- newKey = ResourcesKey{mHash=3f55d5d mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}} rebasedOverrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}

这里 Resources@ca21027 对应的 ActivityResource.overrideConfig 为 Configuration.EMPTY，那么 newKey 的创建就完全使用了 ResourcesManager#rebaseActivityOverrideConfig 方法传入的 newOverrideConfig。

12-17 11:27:22.217 7133 7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{mHash=3f55d5d mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}}

12-17 11:27:22.217 7133 7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = null

12-17 11:27:22.221 7133 7133 I ukynho_res: ResourcesImpl#constructor ---- resourcesImpl = android.content.res.ResourcesImpl@ff5ba1 config = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}

12-17 11:27:22.223 7133 7133 I ukynho_res: ResourcesImpl#updateConfiguration ---- resourcesImpl = android.content.res.ResourcesImpl@ff5ba1 newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid} oldConfig = {1.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut ?orien ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid}

12-17 11:27:22.226 7133 7133 D ResourcesManager: - creating impl=android.content.res.ResourcesImpl@ff5ba1 with key: ResourcesKey{mHash=3f55d5d mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}}

12-17 11:27:22.226 7133 7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- new resourcesIml = android.content.res.ResourcesImpl@ff5ba1

12-17 11:27:22.226 7133 7133 I ukynho_res: ResourcesManager#updateResourcesForActivity ---- i = 0 resourcesImpl = android.content.res.ResourcesImpl@ff5ba1 resources.getImpl() = android.content.res.ResourcesImpl@7863c28

12-17 11:27:22.227 7133 7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@ca21027 mResourcesImpl = android.content.res.ResourcesImpl@7863c28 impl = android.content.res.ResourcesImpl@ff5ba1

最终创建了一个新的 ResourcesKey{mHash=3f55d5d ......}，并且创建了一个新的 ResourcesImpl 对象，ResouresImpl@ff5ba1，与之对应，最后通过 Resources#setImpl 方法将 Resources@ca21027 中的 mResourcesImpl 成员赋值为 ResouresImpl@ff5ba1，完成了此次的 Configuration 更新。

虽然此时 Resources@ca2102 中的成员变量 mResourcesImpl 从 ResourcesImpl@7863c28 被替换为了 ResouresImpl@ff5ba1，但是 ResourcesImpl@7863c28 仍然保存在 ResourcesManager 的 mResourceImpls 成员变量中，这为后续 ResourcesImpl@7863c28 的复用提供了支持。

<table><thead><tr><th>Context</th><th>Resources</th><th>ResourcesImpl</th><th>ResourcesKey</th></tr></thead><tbody><tr><td>ContextImpl@8a62ae</td><td>Resources@dfa5116</td><td></td><td></td></tr><tr><td>ContextImpl@a9dfbba</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@2c79e0</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@256d33f</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@763a4f8</td><td>Resources@c3ad9c2</td><td>ResourcesImpl@ab5eb37</td><td>ResourcesKey{mHash=2a5c3093......}</td></tr><tr><td><strong>ContextImpl@df10a2f</strong></td><td><strong>Resources@ca21027</strong></td><td><strong>ResouresImpl@ff5ba1</strong></td><td><strong>ResourcesKey{mHash=3f55d5d ......}</strong></td></tr><tr><td>ContextImpl@d7e7b3</td><td>Resources@8df4c0f</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td></td><td>Resources@19dc86c</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td></td><td></td><td><strong>ResourcesImpl@7863c28</strong></td><td><strong>ResourcesKey{mHash=7d7a14ea......}</strong></td></tr></tbody></table>

3.3 从横屏转到竖屏
-----------

分别有一次 ResourcesManager#applyConfigurationToResources 和 ResourcesManager#updateResourcesForActivity。

### 3.3.1 ResourcesManager#applyConfigurationToResources

这里遍历的是 ResourcesManager.mResourceImpls，此时 ResourcesManager.mResourceImpls 有 4 个，因为上一步新增了一个 ResourcesImpl#ff5ba1，我们只看这个 ResourcesImpl 对象如何更新，同样我们这里只关注方向这一个属性。

1)、i = 3，ResourcesImpl@7863c28。

2)、i = 2, ResourcesImpl@765ba61。

3)、i = 1, ResourcesImpl@ab5eb37。

4)、i = 0, ResourcesImpl@ff5ba1。

12-17 11:27:34.325 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResources ---- i = 0 key = ResourcesKey{mHash=3f55d5d mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long* ***land*** *finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}} resourcesImpl = android.content.res.ResourcesImpl@ff5ba1

对应的 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY。

12-17 11:27:34.326 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long **port** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.265 fontWeightAdjustment=0?ecid}

这个是传入的 newConfig，方向为竖屏。

12-17 11:27:34.326 7133 7133 I ukynho_res: ResourcesManager#applyConfigurationToResourcesLocked ---- after update, final config = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}

但是由于 ResourcesKey.mOverrideConfiguration 不为 Configuration.EMPTY，update 上 ResourcesKey.mOverrideConfiguration 后方向又变为横屏。

12-17 11:27:34.328 7133 7133 I ukynho_res: ResourcesImpl#updateConfiguration ---- resourcesImpl = android.content.res.ResourcesImpl@ff5ba1 newConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long **land** finger -keyb/v/h -nav/h winConfig={mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid} oldConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen

最终传给 ResourcesImpl#updateConfiguration 的 Configuration 显示横屏，说明由于 ResourcesKey 的存在，ResourcesImpl@7863c2 在此次 ResourcesManager#applyConfigurationToResources 没有更新成功。

### 3.3.2 ResourcesManager#updateResourcesForActivity

这里 Activity Resources 只有一个，即上一步中没有更新成功的 ResourcesImpl@ff5ba1 对应的 Resources@ca21027。

12-17 11:27:34.367 7133 7133 I ukynho_res: ResourcesManager#updateResourcesForActivity ---- activity = android.os.BinderProxy@4f70d2d overrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.3 fontWeightAdjustment=0?ecid} activityResources = android.app.ResourcesManager$ActivityResources@b9789c5 activityResources.overrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}

12-17 11:27:34.368 7133 7133 I ukynho_res: ResourcesManager#updateResourcesForActivity ---- i = 0 resources = android.content.res.Resources@ca21027

此时需要更新的只有 Resources@ca21027。

12-17 11:27:34.369 7133 7133 I ukynho_res: ResourcesManager#rebaseActivityOverrideConfig ---- resources = android.content.res.Resources@ca21027 newOverrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.3 fontWeightAdjustment=0?ecid} oldKey = ResourcesKey{ mHash=3f55d5d mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w722dp h330dp 480dpi nrml long land finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 2400, 1080) mAppBounds=Rect(144, 0 - 2310, 1080) mMaxBounds=Rect(0, 0 - 2400, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_270} s.2 fontWeightAdjustment=0?ecid}}

12-17 11:27:34.369 7133 7133 I ukynho_res: ResourcesManager#rebaseActivityOverrideConfig ---- activityResource.overrideConfig = {0.0 ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut ?orien ?uimode ?night ?touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mMaxBounds=Rect(0, 0 - 0, 0) mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=undefined mAlwaysOnTop=undefined mRotation=undefined} ?fontWeightAdjustment?ecid} rebasedOverrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.3 fontWeightAdjustment=0?ecid}

12-17 11:27:34.370 7133 7133 I ukynho_res: ResourcesManager#rebaseActivityOverrideConfig ---- newKey = ResourcesKey{mHash=7d7a14ea mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.3 fontWeightAdjustment=0?ecid}} rebasedOverrideConfig = {0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.3 fontWeightAdjustment=0?ecid}

这里 Resources@ca21027 对应的 ActivityResource.overrideConfig 为 Configuration.EMPTY，那么 newKey 的创建就完全使用了 ResourcesManager#rebaseActivityOverrideConfig 方法传入的 newOverrideConfig。

12-17 11:27:34.371 7133 7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- key = ResourcesKey{mHash=7d7a14ea mOverrideConfig={0.93 ?mcc?mnc [en_US] ldltr sw360dp w360dp h722dp 480dpi nrml long port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2400) mAppBounds=Rect(0, 90 - 1080, 2256) mMaxBounds=Rect(0, 0 - 1080, 2400) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.3 fontWeightAdjustment=0?ecid}}

12-17 11:27:34.372 7133 7133 I ukynho_res: ResourcesManager#findOrCreateResourcesImplForKeyLocked ---- existing resourcesIml = android.content.res.ResourcesImpl@7863c28

12-17 11:27:34.372 7133 7133 I ukynho_res: ResourcesManager#updateResourcesForActivity ---- i = 0 resourcesImpl = android.content.res.ResourcesImpl@7863c28 resources.getImpl() = android.content.res.ResourcesImpl@ff5ba1

12-17 11:27:34.373 7133 7133 I ukynho_res: Resources#setImpl ---- resources = android.content.res.Resources@ca21027 mResourcesImpl = android.content.res.ResourcesImpl@ff5ba1 impl = android.content.res.ResourcesImpl@7863c28

最终创建了一个新的 ResourcesKey{mHash=3f55d5d ......}，但是这里在 ResourcesImpl.mResourceImpls 中可以找到一个对应的 ResourcesImpl 对象，ResouresImpl@7863c28，最后通过 Resources#setImpl 方法将 Resources@ca21027 中的 mResourcesImpl 成员赋值为 ResouresImpl@7863c28，完成了此次的 Configuration 更新。

<table><thead><tr><th>Context</th><th>Resources</th><th>ResourcesImpl</th><th>ResourcesKey</th></tr></thead><tbody><tr><td>ContextImpl@8a62ae</td><td>Resources@dfa5116</td><td></td><td></td></tr><tr><td>ContextImpl@a9dfbba</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@2c79e0</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@256d33f</td><td>Resources@82d2a74</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td>ContextImpl@763a4f8</td><td>Resources@c3ad9c2</td><td>ResourcesImpl@ab5eb37</td><td>ResourcesKey{mHash=2a5c3093......}</td></tr><tr><td><strong>ContextImpl@df10a2f</strong></td><td><strong>Resources@ca21027</strong></td><td><strong>ResouresImpl@7863c28</strong></td><td><strong>ResourcesKey{mHash=7d7a14ea......}</strong></td></tr><tr><td>ContextImpl@d7e7b3</td><td>Resources@8df4c0f</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td></td><td>Resources@19dc86c</td><td>ResourcesImpl@765ba61</td><td>ResourcesKey{mHash=593d1817 ......}</td></tr><tr><td></td><td></td><td><strong>ResourcesImpl@ff5ba1</strong></td><td><strong>ResourcesKey{mHash=3f55d5d......}</strong></td></tr></tbody></table>

4.1 ResourcesManager#applyConfigurationToResource
-------------------------------------------------

ResourcesManager#applyConfigurationToResources 中，更新的是所有 ResourcesImpl（因为 Resources 只是对 ResourcesImpl 的一层封装，真正持有 Configuration 的是 ResourcesImpl），通过遍历 ResourcesImpl.mResourceImpls，来更新每一个 ResourcesImpl 对象。

```
finalConfig = newConfig.updateFrom(key,mOverrideConfiguration)。


```

其中 newConfig 为从系统服务进程传过来的的 Configuration，key 为当前 ResourcesImpl 对象对应的 ResourcesKey 对象，得到最终的 finalConfig，通过 ResourcesImpl#updateConfiguration 方法，来更新 ResourcesImpl 中的 mConfiguration 对象。

如果一个 ResourcesImpl 对应的 ResourcesKey 不为 Configuration.EMPTY，那么在一次 Configuration 的更新中，finalConfig 的某些属性很有可能被 key.mOverrideConfiguration 覆盖掉。

4.2 ResourcesManager#updateResourcesForActivity
-----------------------------------------------

ResourcesManager#updateResourcesForActivity，更新的是当前 Activity 对应的所有 Resources，那么就需要知道哪些 Resources 是对应当前 Activity 的。根据之前的分析，我们知道每一个 Activity 都对应一个 ActivityResources 对象，ActivityResources.activityResources 队列中的每一个 ActivityResource 都对应一个 Resources 对象，这些 Resources 对象就是与当前 Activity 关联的 Resources 对象。那么我们就可以找到当前 Activity 对应的 ActivityResources 对象，通过遍历 ActivityResoures.activityResources，更新每一个 ActivityResource 对象中保存的 Resources 对象。

更新每一个 Resoures 对象的具体内容是，以本次 ResourcesManager#updateResourcesForActivity 中从系统服务端传入的 newConfig 为 baseConfig，再 update 上创建 Resources 时候传入的初始 Configuration（已经保存在 ActivityResource.overrideConfig 中），生成一个 finalConfig，然后基于这个 finalConfig 生成一个新的 ResourcesKey。

```
finalConfig = newConfig.updateFrom(ActivityResoruce,overrideConfig)。


```

最终有两种情况：

1)、生成的这个 ResourcesKey，在 ResourcesManager.mResourcesImpl 中找不到对应的 ResourcesImpl 对象，那么创建一个新的 ResourcesImpl 对象，并且将键值对 <ResourcesKey, ResourcesImpl> 加入到 ResourcesManager.mResourcesImpl 中，返回这个新创建的 ResourcesImpl 对象。

2)、生成的这个 ResourcesKey，在 ResourcesManager.mResourcesImpl 中可以找到对应的 ResourcesImpl 对象，那么直接返回这个 ResourcesImpl 对象。

最终通过 Resources#setImpl，通过替换 Resources 中的 mResourcesImpl 成员完成当前 Resources 的更新。

4.3 Log 分析
----------

从 Log 分析这一节中，可以看到 ResourcesManager 存在 ResourcesImpl 对象的复用。

我们只看和 Resources@ca21027 的部分：

1)、Activity 创建的时候，生成了 Resources@ca21027，以及 <ResourcesKey{ mHash=7d7a14ea......}, ResouresImpl@7863c28>。

2)、从竖屏转为横屏，生成了新的键值对，<ResourcesKey{ mHash=3f55d5d......}, ResourcesImpl@ff5ba1 >，Resources@ca21027 将成员变量 mResourcesImpl 从 ResouresImpl@7863c28 替换为 ResourcesImpl@ff5ba1。

3)、从横屏转为竖屏，这个时候，没有创建新的键值对，而是复用了第一步的 <ResourcesKey{ mHash=7d7a14ea......}, ResouresImpl@7863c28>，Resources@ca21027 将成员变量 mResourcesImpl 从 ResouresImpl@ff5ba1 替换为 ResourcesImpl@7863c28。

4)、同样，再从竖屏转为横屏，也不会创建新的键值对，会复用第二步生成的 <ResourcesKey{ mHash=3f55d5d......}, ResourcesImpl@ff5ba1>，Resources@ca21027 将成员变量 mResourcesImpl 从 ResouresImpl@7863c28 替换为 ResourcesImpl@ff5ba1。