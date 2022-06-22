# Android 学习
- [Android 学习](#android-学习)
  - [通知](#通知)
    - [发送通知](#发送通知)
    - [取消通知](#取消通知)
    - [关于通知的其他连缀方法](#关于通知的其他连缀方法)
  - [调用相机照相&从相册中选择图片](#调用相机照相从相册中选择图片)
    - [调用摄像头拍照并显示](#调用摄像头拍照并显示)
    - [从相册中选择照片](#从相册中选择照片)
  - [播放多媒体文件](#播放多媒体文件)
    - [播放音频](#播放音频)
    - [播放视频](#播放视频)
  - [Android网络编程](#android网络编程)
    - [使用HttpURLConnection发送请求](#使用httpurlconnection发送请求)
      - [GET](#get)
      - [POST](#post)
    - [使用OkHttp发送请求](#使用okhttp发送请求)
    - [解析xml文件](#解析xml文件)
      - [PULL解析](#pull解析)
      - [SAX解析](#sax解析)
    - [解析JSON格式数据](#解析json格式数据)
      - [JSONObject方式](#jsonobject方式)
      - [GSON方式](#gson方式)
    - [对okhttp3访问方式进行封装](#对okhttp3访问方式进行封装)
  - [服务](#服务)
    - [Android多线程编程](#android多线程编程)
## 通知
### 发送通知
Android 8.0以上的时候需要先声明channel才能注册通知事件，因此若要发送一个通知事件，基本写法应该为：<br>
```
// 在某个函数中注册，一般是onCreate
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    String channel_1_id = "chat";
    String channel_1_name = "聊天信息";
    int channel_1_importance = NotificationManager.IMPORTANCE_HIGH;
    createNotificationChannel(channel_1_id,channel_1_name,channel_1_importance);
}

...
// 函数之外，活动之内
@TargetApi(Build.VERSION_CODES.O)
private void createNotificationChannel(String channelId, String channelName, int importance) {
    NotificationChannel channel = new NotificationChannel(channelId, channelName, importance);
    channel.setShowBadge(true); // 设置角标功能，对应修改.setNumber(k)
    NotificationManager notificationManager = (NotificationManager) getSystemService(
                NOTIFICATION_SERVICE);
    notificationManager.createNotificationChannel(channel);
}
...

// 某个事件要发送通知的时候，比如onClick
Intent intentPi = new Intent(this, NotificationActivity.class);  // 第二个参数为目标活动
PendingIntent pi = PendingIntent.getActivity(this, 0, intentPi, 0);]
// 上面两行和点击通知跳转事件有关，对应.setContentIntent(pi)
NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    NotificationChannel channel = manager.getNotificationChannel("chat");
    if (channel.getImportance() == NotificationManager.IMPORTANCE_NONE) {
        Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
        intent.putExtra(Settings.EXTRA_APP_PACKAGE, getPackageName());
        intent.putExtra(Settings.EXTRA_CHANNEL_ID, channel.getId());
        startActivity(intent);
        Toast.makeText(this, "聊天消息通知权限已被关闭，请手动将通知打开", Toast.LENGTH_LONG).show();
    }
}
Notification notification = new NotificationCompat.Builder(this, "chat")
                        .setContentTitle("This is content title")
                        .setContentText("This is content text")
                        .setWhen(System.currentTimeMillis())
                        .setSmallIcon(R.mipmap.ic_launcher)
                        .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
                        .setContentIntent(pi)
                        .setAutoCancel(true)
                        .setNumber(1)
                        .build();
manager.notify(1, notification);
```
### 取消通知
取消通知有两种方法：<br>
1. 在new NotificationCompat.Builder(this, channel_id)的连缀方法中缀入.setAutoCancel(true)
2. 在通知事件结束后加入如下代码：
   ```
   NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);  
   // 如果之前定义过manager的话就不用再重复定义了
   manager.cancel(1)
   // 之前发送通知时使用的是manager.notify(1, notification)，因此取消用的id是1
   ```
### 关于通知的其他连缀方法
1. 发出通知的同时播放音频
   ```
   .setSound(Uri.fromFile(new File("/system/media/audio/ringtones/luna.ogg")))
   // 在/system/media/audio/ringtones目录下有很多音频文件，可以自行选择
   ```
2. 发出通知时震动
   ```
   .setVibrate(new long[] {0, 1000, 1000, 1000})
   // 立刻振动1s，随后静止1s，再振动1s
   // 需要在AndroidManifest.xml中声明震动权限：
   <uses-permission android:name="android.permission.VIBRATE" />
   ```
3. 有LED灯的手机发送通知时闪烁LED灯
    ```
    .setLights(Color.GREEN, 1000, 1000)
    // 绿色，亮1s，灭1s
    ```
4. 默认通知效果
    ```
    .setDefaults(NotificationCompat.DEFAULT_ALL)
    ```
5. 通知内容为长文本
   ```
   .setStyle(new NotificationCompat.BigTextStyle().bigText("This is a long ... long text."))
   ```
6. 通知中发送大图片
   ```
   .setStyle(new NotificationCompat.BigPictureStyle().bigPicture(BitmapFactory.decodeResource(getResources(), R.drawable.your_big_image)))
   // your_big_image是你提前放在drawable文件夹内的图片
   ```
7. 设置通知的重要程度
   ```
   .setPriority(NotificationCompat.PRIORITY_MAX)
   // 参数由高到低可选PRIORITY_MAX、PRIORITY_HIGH、PRIORITY_DEFAULT、PRIORITY_LOW、PRIORITY_MIN
   // 感觉构造的时候设置过的话，这里就不用再设置了吧？
   ```
## 调用相机照相&从相册中选择图片
### 调用摄像头拍照并显示
假设在`activity_main.xml`中已经写好了一个点击拍照的Button和一个回显照片的ImageView。对MainActivity写入如下内容：<br>
```
public class MainActivity extends AppCompatActivity {

    public static final int TAKE_PHOTO = 1;
    private ImageView picture;
    private Uri imageUri;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button takePhoto = (Button) findViewById(R.id.take_photo);
        picture = (ImageView) findViewById(R.id.picture);
        takePhoto.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                File outputImage = new File(getExternalCacheDir(), "output_image.jpg");
                // 这里获取了应用关联缓存目录，具体而言就是/sdcard/Android/data/<package name>/cache (未求证)
                try {
                    if (outputImage.exists()) {
                        outputImage.delete();
                    }
                    outputImage.createNewFile();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                // Intent能传递的数据空间有限，需要转化成Uri
                if (Build.VERSION.SDK_INT >= 24) {
                    // targetSdkVersion 24中对文件安全做了限制
                    imageUri = FileProvider.getUriForFile(MainActivity.this,
                            "com.example.cameraalbumtest.fileprovider", outputImage);
                    // FileProvider.getUriForFile的第二个参数为认证字符串，自定义，但要与后面保持一致
                } else {
                    // 系统版本 < Android 7.0
                    imageUri = Uri.fromFile(outputImage);
                }
                // 启动相机程序
                Intent intent = new Intent("android.media.action.IMAGE_CAPTURE");
                intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
                startActivityForResult(intent, TAKE_PHOTO);
            }
        });
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        switch (requestCode) {
            case TAKE_PHOTO:
                if (resultCode == RESULT_OK) {
                    try {
                        // 显示拍摄的照片
                        Bitmap bitmap = BitmapFactory.decodeStream(getContentResolver().openInputStream(imageUri));
                        picture.setImageBitmap(bitmap);
                    } catch (FileNotFoundException e) {
                        e.printStackTrace();
                    }
                }
                break;
            default:
                break;
        }
    }
}
```
上面用到的FileProvider是一种内容提供器，所以要在`AndroidManifest.xml`中对其进行注册：<br>
```
<provider
    android:authorities="com.example.cameraalbumtest.fileprovider"
    android:name="androidx.core.content.FileProvider"
    android:exported="false"
    android:grantUriPermissions="true" >

    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />

</provider>
```
这里的`meta-data`标签中提到的`@xml/file_paths`是带目录的新建文件`/main/res/xml/file_paths.xml`，内容如下：<br>
```
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path
        name="my_images"
        path="." />
</paths>
```
其中`external-path`表情指定Uri共享的路径，具体为`path`属性。同时为保证兼容性，需要在`AndroidManifest.xml`中声明对SD卡的访问权限：<br>
```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```
### 从相册中选择照片
读取相册照片需要申请读写权限，所以首先要在`AndroidManifest.xml`中声明读取权限：<br>
```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```
其次，为了让用户更好地控制自己的文件，同时也为了对文件混乱的情况加以限制，Android Q修改了APP访问外部存储中文件的方法，使得用户在申请并获得了上述权限之后，仍然被限制访问外部存储。一个可用的解决方法是在`AndroidManifest.xml`的`application`标签头部加入如下属性（在PACM00 Android 10上测试可行）：<br>
```
android:requestLegacyExternalStorage="true"
```
假设在`activity_main.xml`中已经写好了一个从相册中选择照片的Button。对MainActivity写入如下内容：<br>
```
public class MainActivity extends AppCompatActivity {
    ...
    public static final int CHOOSE_PHOTO = 2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        Button chooseFromAlbum = (Button) findViewById(R.id.choose_from_album);
        chooseFromAlbum.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 动态申请权限
                if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.permission.READ_EXTERNAL_STORAGE)
                        != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(MainActivity.this,
                            new String[] {Manifest.permission.READ_EXTERNAL_STORAGE}, 1);
                } else {
                    openAlbum();
                }
            }
        });
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        switch (requestCode) {
            case 1:
                if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    openAlbum();
                } else {
                    Toast.makeText(this, "请给予读取相册的权限！", Toast.LENGTH_LONG).show();
                }
                break;
            default:
                break;
        }
    }

    private void openAlbum() {
        Intent intent = new Intent("android.intent.action.GET_CONTENT");
        intent.setType("image/*");
        startActivityForResult(intent, CHOOSE_PHOTO);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        switch (requestCode) {
            ...
            case CHOOSE_PHOTO:
                if (resultCode == RESULT_OK) {
                    if (Build.VERSION.SDK_INT >= 19) {
                        // Android 4.4以上返回的是封装过的图片Uri
                        handleIImageOnKitKat(data);
                    } else {
                        handleImageBeforeKitKat(data);
                    }
                }
                break;
            default:
                break;
        }
    }

    private void handleIImageOnKitKat(Intent data) {
        String imagePath = null;
        Uri uri = data.getData();
        if (DocumentsContract.isDocumentUri(this, uri)) {
            String docId = DocumentsContract.getDocumentId(uri);
            if ("com.android.providers.media.documents".equals(uri.getAuthority())) {
                String id = docId.split(":")[1];
                String selection = MediaStore.Images.Media._ID + "=" + id;
                imagePath = getImagePath(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, selection);
            } else if ("com.android.providers.downloads.documents".equals(uri.getAuthority())) {
                Uri contentUri = ContentUris.withAppendedId(Uri.parse("content://downloads/public_downloads"), Long.valueOf(docId));
                imagePath = getImagePath(contentUri, null);
            }
        } else if ("content".equalsIgnoreCase(uri.getScheme())) {
            imagePath = getImagePath(uri, null);
        } else if ("file".equalsIgnoreCase(uri.getScheme())) {
            imagePath = uri.getPath();
        }
        displayImage(imagePath);
    }

    private void handleImageBeforeKitKat(Intent data) {
        Uri uri = data.getData();
        String imagePath = getImagePath(uri, null);
        displayImage(imagePath);
    }

    private String getImagePath(Uri uri, String selection) {
        String path = null;
        // 通过Uri和selection获取真实的图片路径
        Cursor cursor = getContentResolver().query(uri, null, selection, null, null);
        if (cursor != null) {
            if (cursor.moveToFirst()) {
                path = cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.DATA));
            }
            cursor.close();
        }
        return path;
    }

    private void displayImage(String imagePath) {
        if (imagePath != null) {
            Bitmap bitmap = BitmapFactory.decodeFile(imagePath);
            picture.setImageBitmap(bitmap);
        } else {
            Toast.makeText(this, "无图片", Toast.LENGTH_LONG).show();
        }
    }
}
```
## 播放多媒体文件
### 播放音频
也是先在`AndroidManifest.xml`中声明读权限：<br>
```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```
播放音频文件一般用`MediaPlayer`类。假设在`activity_main.xml`中已经写好了3个Button，分别对应播放、暂停、终止功能。对MainActivity写入如下内容：<br>
```
public class MainActivity extends AppCompatActivity implements View.OnClickListener{

    private MediaPlayer mediaPlayer = new MediaPlayer();
    private static final int READ_PERMISSION = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button play = (Button) findViewById(R.id.play);
        Button pause = (Button) findViewById(R.id.pause);
        Button stop = (Button) findViewById(R.id.stop);
        play.setOnClickListener(this);
        pause.setOnClickListener(this);
        stop.setOnClickListener(this);
        if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.permission.READ_EXTERNAL_STORAGE)
                != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(MainActivity.this, new String[] {
                    Manifest.permission.READ_EXTERNAL_STORAGE}, READ_PERMISSION);
        } else {
            initMediaPlayer();
        }
    }

    private void initMediaPlayer() {
        try {
            File file = new File(Environment.getExternalStorageDirectory(), "Music/Recordings/Standard Recordings/标准录音 1.mp3");
            // 第一个参数获取到根目录，并与第二个参数指示的子路径进行拼接
            mediaPlayer.setDataSource(file.getPath());
            mediaPlayer.prepare();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        switch (requestCode) {
            case READ_PERMISSION:
                if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    initMediaPlayer();
                } else {
                    Toast.makeText(this, "请授予文件读取权限", Toast.LENGTH_LONG).show();
                    finish();
                }
                break;
            default:
        }
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.play:
                if (!mediaPlayer.isPlaying()) {
                    mediaPlayer.start();
                }
                break;
            case R.id.pause:
                if (mediaPlayer.isPlaying()) {
                    mediaPlayer.pause();
                }
                break;
            case R.id.stop:
                if (mediaPlayer.isPlaying()) {
                    mediaPlayer.reset();
                    initMediaPlayer();
                }
                break;
            default:
                break;
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mediaPlayer != null) {
            mediaPlayer.stop();
            mediaPlayer.release();
        }
    }
}
```
`MediaPlayer`在使用`setDataSource()`读取文件并获取播放对象之后、执行`start()`方法之前必须先执行`prepare()`方法。其他常用方法及功能：
<table>
<tr>
<td>seekTo()</td>
<td>从指定位置开始播放音频</td>
</tr>
<tr>
<td>getDuration()</td>
<td>获取载入的音频文件的时长</td>
</tr>
</table>

### 播放视频
先在`AndroidManifest.xml`中声明读取权限：<br>
```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```
同时在`AndroidManifest.xml`的`application`标签头部添加属性：<br>
```
android:requestLegacyExternalStorage="true"
```
假设已经在界面布局了3个Button分别对应播放、暂停、重播功能；同时布局了一个`VideoView`用于播放视频。对MainActivity写入如下内容：<br>
```
public class MainActivity extends AppCompatActivity implements View.OnClickListener{

    private VideoView videoView;
    private static final int PERMISSION_READ = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button play = (Button) findViewById(R.id.play);
        Button pause = (Button) findViewById(R.id.pause);
        Button replay = (Button) findViewById(R.id.replay);
        play.setOnClickListener(this);
        pause.setOnClickListener(this);
        replay.setOnClickListener(this);
        videoView = (VideoView) findViewById(R.id.video_view);
        if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.permission.READ_EXTERNAL_STORAGE)
        != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(MainActivity.this, new String[]
                    {Manifest.permission.READ_EXTERNAL_STORAGE}, PERMISSION_READ);
        } else {
            initVideoPath();
        }
    }

    private void initVideoPath() {
        File file = new File(Environment.getExternalStorageDirectory(), "DCIM/Camera/test.mp4");
        videoView.setVideoPath(file.getPath());
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        switch (requestCode) {
            case PERMISSION_READ:
                if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    initVideoPath();
                } else {
                    Toast.makeText(this, "请给予文件读取权限以播放视频！", Toast.LENGTH_LONG).show();
                    finish();
                }
                break;
            default:
                break;
        }
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.play:
                if (!videoView.isPlaying()) {
                    videoView.start();
                }
                break;
            case R.id.pause:
                if (videoView.isPlaying()) {
                    videoView.pause();
                }
            case R.id.replay:
                if (!videoView.isPlaying()) {
                    videoView.start();
                }
                videoView.resume();
            default:
                break;
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (videoView != null) {
            videoView.suspend();
        }
    }
}
```
其他常用方法及功能：<br>
<table>
<tr>
<td>seekTo()</td>
<td>从指定位置开始播放视频</td>
</tr>
<tr>
<td>getDuration()</td>
<td>获取载入的视频文件的时长</td>
</tr>
</table>

## Android网络编程
访问网络需要申请网络权限。在`AndroidManifest.xml`中声明网络权限：<br>
```
<uses-permission android:name="android.permission.INTERNET" />
```
展示网页内容一般使用`WebView`控件。假设已经在`activity_main.xml`中已经写好了一个WebView则在MainActivity写入如下内容即可展示bing的首页：<br>
```
WebView webView = (WebView) findViewById(R.id.web_view);
webView.getSettings().setJavaScriptEnabled(true);
webView.setWebViewClient(new WebViewClient());
webView.loadUrl("https://cn.bing.com");
```
### 使用HttpURLConnection发送请求
Android 9之后默认不允许使用 HTTP, 只能使用 HTTPS。如果要使用 HTTP , 必须在 `AndroidManifest.xml` 文件的`application`标签头部添加如下属性：<br>
```
android:usesCleartextTraffic="true"
```
#### GET
HTTP发送GET请求的一般流程是：<br>根据网址构造URL对象<br>-> 使用该对象的openConnection()方法得到返回值，并将返回值转为HttpURLConnection对象，记为connection<br>-> 对connection进行设置，如请求方法（GET）、超时时限等<br>-> 使用connection的getInputStream()方法得到服务器返回的流，并据此构造BufferedReader以读取结果<br>
假设我们把通过GET访问`http://cn.bing.com`的功能封装在函数sendRequestWithHttpURLConnection()中，并在TextView控件中展示请求结果：
```
private void sendRequestWithHttpURLConnection() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            HttpURLConnection connection = null;
            BufferedReader reader = null;
            try {
                URL url = new URL("http://cn.bing.com");
                connection = (HttpURLConnection) url.openConnection();
                connection.setRequestMethod("GET");
                connection.setConnectTimeout(8000);
                connection.setReadTimeout(8000);
                InputStream in = connection.getInputStream();
                reader = new BufferedReader(new InputStreamReader(in));
                StringBuilder response = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) {
                    response.append(line);
                }
                showResponse(response.toString());
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if(reader != null) {
                    try {
                        reader.close();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                if (connection != null) {
                    connection.disconnect();
                }
            }
        }
    }).start();
}

private void showResponse(final String response) {
    runOnUiThread(new Runnable() {
        @Override
        public void run() {
            responseText.setText(response);
        }
    });
}
```
#### POST
POST请求和GET请求稍有不同之处，除了更改connection的协议参数外，还要把附带数据用键值对写入后才能调用getInputStream()方法获取服务器返回的流。假设要向服务器提交账号密码：<br>
```
...
URL url = new URL("http://127.0.0.1");
connection = (HttpURLConnection) url.openConnection();
connection.setRequestMethod("POST");
connection.setRequestMethod("POST");
DataOutputStream out = new DataOutputStream(connection.getOutputStream());
out.writeBytes("Username=admin&password=123456");
connection.setConnectTimeout(8000);
connection.setReadTimeout(8000);
InputStream in = connection.getInputStream();
...
```
### 使用OkHttp发送请求
OkHttp也逐渐在安卓开发中得到普及。如果使用OkHttp，则需要在`app/build.gradle`的`dependencies`闭包中添加如下依赖：<br>
```
implementation 'com.squareup.okhttp3:okhttp:3.4.1'
```
添加前可以去OkHttp项目主页查看有无更新，目前最新的版本是4.10.0，但是使用4.10.0会出现一些奇怪的问题，因此选择使用3.4.1。以下是一个GET请求示例：<br>
```
private void sendRequestWithOkHttp() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                OkHttpClient client = new OkHttpClient();
                Request request = new Request.Builder()
                            .url("http://cn.bing.com")
                            .build();
                Response response = client.newCall(request).execute();
                String responseData = response.body().string();
                showResponse(responseData);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }).start();
}
```
首先构建`Resquest`对象，然后通过`newCall()`方法创建`Call`对象，调用`execute()`方法获取服务器返回的数据。OkHttp可以通过连缀的方式在`Builder()`与`build()`之间添加参数。<br>
发送POST请求之前需要先构造一个`requestBody`对象用于存放待提交的参数，随后将其传入post()方法即可：<br>
```
private void sendRequestWithOkHttp() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                OkHttpClient client = new OkHttpClient();
                RequestBody requestBody = new FormBody.Builder()
                                                .add("username", "admin")
                                                .add("password", "123456")
                                                .build();
                Request request = new Request.Builder()
                            .url("http://www.baidu.com")
                            .post(requestBody)
                            .build();
                Response response = client.newCall(request).execute();
                String responseData = response.body().string();
                showResponse(responseData);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }).start();
}
```
### 解析xml文件
假设在某服务器上有一xml文件`get_data.xml`，内容如下所示：<br>
```
<apps>
	<app>
		<id>1</id>
		<name>Google Maps</name>
		<version>1.0</version>
	</app>
	<app>
		<id>2</id>
		<name>Edge</name>
		<version>2.1</version>
	</app>
	<app>
		<id>3</id>
		<name>Apache Server</name>
		<version>2.4.45</version>
	</app>
</apps>
```
在请求得到返回之后对xml文件进行解析，可以用PULL解析或者SAX解析。
#### PULL解析
PULL解析的代码如下所示：
```
private void parseXMLWithPull(String xmlData) {
    try {
        XmlPullParserFactory factory = XmlPullParserFactory.newInstance();
        XmlPullParser xmlPullParser = factory.newPullParser();
        xmlPullParser.setInput(new StringReader(xmlData));
        int eventType = xmlPullParser.getEventType();
        String id = "";
        String name = "";
        String version = "";
        while (eventType != XmlPullParser.END_DOCUMENT) {
            String nodeName = xmlPullParser.getName();
            switch (eventType) {
                case XmlPullParser.START_TAG: {
                    if ("id".equals(nodeName)) {
                        id = xmlPullParser.nextText();
                    } else if ("name".equals(nodeName)) {
                        name = xmlPullParser.nextText();
                    } else if ("version".equals(nodeName)) {
                        version = xmlPullParser.nextText();
                    }
                    break;
                }
                case  XmlPullParser.END_TAG: {
                    if ("app".equals(nodeName)) {
                        Log.d("TAG", "id is "+id);
                        Log.d("TAG", "name is " + name);
                        Log.d("TAG", "version is "+version);
                    }
                    break;
                }
                default:
                    break;
            }
            eventType = xmlPullParser.next();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
#### SAX解析
使用SAX解析方式需要新建一个`ContentHandler`类继承自`DefaultHandler`并重写父类的5个方法：
```
public class ContentHandler extends DefaultHandler {

    private String nodeName;
    private StringBuilder id;
    private StringBuilder name;
    private StringBuilder version;

    @Override
    public void startDocument() throws SAXException {
        id = new StringBuilder();
        name = new StringBuilder();
        version = new StringBuilder();
    }

    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
        // 记录当前节点名
        nodeName = localName;
    }

    @Override
    public void characters(char[] ch, int start, int length) throws SAXException {
        if ("id".equals(nodeName)) {
            id.append(ch, start, length);
        } else if ("name".equals(nodeName)) {
            name.append(ch, start, length);
        } else  if ("version".equals(nodeName)) {
            version.append(ch, start, length);
        }
    }

    @Override
    public void endElement(String uri, String localName, String qName) throws SAXException {
        if ("app".equals(localName)) {
            Log.d("ContentHandler", "id is "+id.toString().trim());
            Log.d("ContentHandler", "name is "+name.toString().trim());
            Log.d("ContentHandler", "version is "+version.toString().trim());
            // 最后要将StringBuilder清空
            id.setLength(0);
            name.setLength(0);
            version.setLength(0);
        }
    }

    @Override
    public void endDocument() throws SAXException {
        super.endDocument();
    }
}
```
随后在`MainActivity`中加入如下函数：
```
private void parseXMLWithSAX(String xmlData) {
    try {
        SAXParserFactory factory = SAXParserFactory.newInstance();
        XMLReader xmlReader = factory.newSAXParser().getXMLReader();
        ContentHandler handler = new ContentHandler();
        xmlReader.setContentHandler(handler);
        xmlReader.parse(new InputSource(new StringReader(xmlData)));
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
### 解析JSON格式数据
假设在某服务器上有一xml文件`get_data.json`，内容如下所示：<br>
```
[{"id":"5", "version":"5.5","name":"Clash of Clans"},
{"id":"6", "version":"7.0","name":"Boom Beach"},
{"id":"7", "version":"3.5","name":"Clash Royale"}]
```
在请求得到返回之后对json文件进行解析，可以用JSONObject或者GSON等方式解析。
#### JSONObject方式
JSONObject方式的代码如下：<br>
```
private void parseJSONWithJSONObject(String jsonData) {
    try {
        JSONArray jsonArray = new JSONArray(jsonData);
        for (int i = 0; i < jsonArray.length(); i++) {
            JSONObject jsonObject = jsonArray.getJSONObject(i);
            String id = jsonObject.getString("id");
            String name = jsonObject.getString("name");
            String version = jsonObject.getString("version");
            Log.d("TAG", "id is "+id);
            Log.d("TAG", "name is " + name);
            Log.d("TAG", "version is "+version);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
#### GSON方式
使用GSON方式需要在`app/build.gradle`的`dependencies`闭包中添加依赖：<br>
```
implementation 'com.google.code.gson:gson:2.8.5'
```
GSON可以将一段JSON格式的字符串自动映射成一个对象，因此需要根据JSON内容先定义一个类。例如针对本例可以新增一个App类：<br>
```
public class App {

    private String id;
    private String name;
    private String version;

    public void setId(String id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setVersion(String version) {
        this.version = version;
    }

    public String getName() {
        return name;
    }

    public String getId() {
        return id;
    }

    public String getVersion() {
        return version;
    }
}
```
当JSON数组结构稍微麻烦些时，一般要借助GSON中的`TypeToken`方法将期望解析成的数据类型传入`fromJson()`方法中。因此整个过程代码如下所示：<br>
```
private void parseJSONWithGSON(String jsonData) {
    Gson gson = new Gson();
    List<App> appList = gson.fromJson(jsonData, new TypeToken<List<App>>(){}.getType());
    for (App app : appList) {
        Log.d("TAG", "id is "+app.getId());
        Log.d("TAG", "name is " + app.getName());
        Log.d("TAG", "version is "+app.getVersion());
    }
}
```
### 对okhttp3访问方式进行封装
新建一个`HttpUtil`公共类，对其写入请求方式，则想要发起网络请求时只需要调用其即可：<br>
```
public class HttpUtil {

    public static void okHttpGet(String address, okhttp3.Callback callback) {
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
                .url(address)
                .build();
        client.newCall(request).enqueue(callback);
    }

    public static void okHttpPost(String address, RequestBody requestBody, okhttp3.Callback callback) {
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
                .url(address)
                .post(requestBody)
                .build();
        client.newCall(request).enqueue(callback);
    }
}
```
其中`enqueue()`方法内部已经开好了子线程，这里不能执行任何UI操作，除非借助`runOnUiThread()`方法进行线程转换。需要GET请求时，如下：<br>
```
String url = "www.baidu.com";
HttpUtil.okHttpGet(url, new okhttp3.Callback() {
    @Override
    public void onResponse(Call call, Response response) throws IOException {
        // 得到服务器返回的内容
        String responseData = response.body().string();
    }

    @Override
    public void onFailure(Call call, IOException e) {
        // 对异常情况处理
    }
});
``` 
需要POST请求时如下：<br>
```
RequestBody requestBody = new FormBody.Builder()
                            .add("username", "admin")
                            .add("password", "123456")
                            .build();
HttpUtil.okHttpPost(url, requestBody, new okhttp3.Callback() {
    @Override
    public void onResponse(Call call, Response response) throws IOException {
        // 得到服务器返回的内容
        String responseData = response.body().string();
    }

    @Override
    public void onFailure(Call call, IOException e) {
        // 对异常情况处理
    }
});
```
## 服务
需要在服务的内部创建子线程并执行具体的任务。<br>
### Android多线程编程


<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>








