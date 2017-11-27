# uiUpdate
子线程真的不能刷新ui吗

1.为什么我的子线程更新了 UI 没报错？
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    title = (TextView) findViewById(R.id.title_tips);
    doGet("http;//www.baidu.com", new Callback() {
        @Override
        public void onFailure(Request request, IOException e) {

        }
        @Override
        public void onResponse(Response response) throws IOException {
            title.setText(response.body().string()); // 这里在子线程更新了 text
        }
    });
}

private void doGet(String url,Callback callback) {
    OkHttpClient client = new OkHttpClient();

    Request.Builder builder = new Request.Builder();
    Request request = builder.url(url).get().build();

    client.newCall(request).enqueue(callback);
}

在看到他发给我的代码，onCreate 里面的部分，一切已经明了，这也是我之前面试几年经验的人设过的坑。下面我直接讲原因，源码分析那些你们自己去看吧，你应该去看。

子线程不能更新 UI 的限制是 viewRootImpl.java 内部限制了
  void checkThread() {
  // 该方法是 viewRootImpl.java 内部代码
  if (mThread != Thread.currentThread()) {
      throw new CalledFromWrongThreadException(
              "Only the original thread that created a view hierarchy can touch its views.");
  }
}

对组件 Activity 而言，viewRootImpl 的初始化在 onCreate 之后，onResume 之后。
如果你的子线程更新代码在满足下面的条件下，那么它可以顺利运行:
修改应用层的 viewRootImpl.java 源码，解除限制
把你更新代码写在 onResume 之前，例如 onCreate 里面，且，更新之际要赶在 viewRootImpl 初始化之前。

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    title = (TextView) findViewById(R.id.title_tips);
    new Thread(
            new Runnable() {
                @Override
                public void run() {
                    try {
                        // 等待 onResume 执行完，让 viewRootImpl 初始化完成
                        Thread.sleep(3000); // ---------- 这里，看这里
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    title.setText("我执行不了");
                }
            }
    ).start();
}

。
