---
title: 使用Retrofit2.0实现Google Drive文件上传进度显示
date: 2016-04-23 19:19:06
categories: Google Drive Android
tags: [Android, Google Drive, Retrofit]
---

在[上一篇文章][1]中，我们完成了Retrofit2.0接口的定义及基本的调用。
不知大家是否关注到`DriveApi.uploadFileMutil()`这个接口方法需要传入两个`MultipartBody.Part`对象，分别对应Metadata part和Media part，如何实现这里的Part对象呢？分为两部分：

## Metadata Part
这部分比较简单，`MultipartBody.Part.create()`方法直接创建即可：
```java
String content = "{\"name\": \"" + fileName + "\"}";
MediaType contentType = MediaType.parse("application/json; charset=UTF-8");
MultipartBody.Part metaPart =
        MultipartBody.Part.create(RequestBody.create(contentType, content));
```

## Media Part
需要我们自己实现一个`RequestBody`对象，并重写里面的`contentType()`和`writeTo()`方法。代码可以参考StackOverflow上的这个回答：[Is it possible to show progress bar when upload image via Retrofit 2][2]
```java
public class ProgressRequestBody extends RequestBody {
    private File mFile;
    private String mPath;
    private UploadCallbacks mListener;

    private static final int DEFAULT_BUFFER_SIZE = 2048;

    public interface UploadCallbacks {
        void onProgressUpdate(int percentage);
        void onError();
        void onFinish();
    }

    public ProgressRequestBody(final File file, final  UploadCallbacks listener) {
        mFile = file;
        mListener = listener;            
    }

    @Override
    public MediaType contentType() {
        // i want to upload only images
        return MediaType.parse("image/*");
    }

    @Override
    public void writeTo(BufferedSink sink) throws IOException {
        long fileLength = mFile.length();
        byte[] buffer = new byte[DEFAULT_BUFFER_SIZE];
        FileInputStream in = new FileInputStream(mFile);
        long uploaded = 0;

        try {
            int read;
            Handler handler = new Handler(Looper.getMainLooper());
            while ((read = in.read(buffer)) != -1) {

                uploaded += read;
                sink.write(buffer, 0, read);
                // update progress on UI thread
                handler.post(new ProgressUpdater(uploaded, fileLength));
            }
        } finally {
            in.close();
        }
    }

    private class ProgressUpdater implements Runnable {
        private long mUploaded;
        private long mTotal;
        public ProgressUpdater(long uploaded, long total) {
            mUploaded = uploaded;
            mTotal = total;
        }

        @Override
        public void run() {
            mListener.onProgressUpdate((int)(100 * mUploaded / mTotal));            
        }
    }
}
```
注意SO这个回答贴出的代码还有点问题，下面这句应该放在while循环的最下面，否则你最后一直只能收到99%。
```java
// update progress on UI thread
handler.post(new ProgressUpdater(uploaded, fileLength));
```

## Callback
然后实现`UploadCallbacks`接口里的相关回调即可：
```java
class MyActivity extends AppCompatActivity implements ProgressRequestBody.UploadCallbacks {
    ProgressBar progressBar;
    ProgressRequestBody mBody;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        progressBar = findViewById(R.id.progressBar);
        mBody = new ProgressRequestBody(mFile, this);
    }

    @Override
    public void onProgressUpdate(int percentage) {
        // set current progress
        progressBar.setProgress(percentage);
    }

    @Override
    public void onError() {
        // do something on error
    }

    @Override
    public void onFinish() {
        // do something on upload finished
        // for example start next uploading at queue
        progressBar.setProgress(100);
    }

}
```
最后同样调用MultipartBody.Part.create()创建即可：
```java
MultipartBody.Part dataPart = MultipartBody.Part.create(mBody);
```

[1]: https://codezjx.github.io/2016/04/23/google-drive-api-with-retrofit2/
[2]: http://stackoverflow.com/questions/33338181/is-it-possible-to-show-progress-bar-when-upload-image-via-retrofit-2/33384551#33384551
