关于Activity token的说明： Token 是一个ibinder对象，可以在进程间传递。其中包含name。 那是哪个类的静态内部类？ 答案： 是 ActivityRecord 的静态内部类。
持有 ActivityRecord的弱引用。保持跨进程的联系。
```
 static class Token extends IApplicationToken.Stub {
        private final WeakReference<ActivityRecord> weakActivity;
        private final String name;

        Token(ActivityRecord activity, Intent intent) {
            weakActivity = new WeakReference<>(activity);
            name = intent.getComponent().flattenToShortString();
        }
```












