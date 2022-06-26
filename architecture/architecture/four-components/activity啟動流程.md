关于Activity token的说明： Token 是一个ibinder对象，可以在进程间传递。其中包含name。

```
 static class Token extends IApplicationToken.Stub {
        private final WeakReference<ActivityRecord> weakActivity;
        private final String name;

        Token(ActivityRecord activity, Intent intent) {
            weakActivity = new WeakReference<>(activity);
            name = intent.getComponent().flattenToShortString();
        }
```












