#DigitalCalendar 项目总结
----
[TOC]

###项目难点：
1. 每次点开都显示今日日期页面
	> finish所有Acitivity，结束进程？
2. 系统没有自带的字体非常慢,如何预加载？
	>在Application里面load到一个HashStable里面，在BaseActivity获取。
3. 及时插入通知和延迟插入通知统一，Proxy模式
	


### RecyclerView
将原来的ViewFlipper替换为RecyclerView，改善滑动用户体验。

### 自定义View
Android自定义View的三种实现方式: 
 - 组合控件
	将一些标准控件（如TextView，Button）放置在RelativeLayout等ViewGroup里组合起来形成一个新的控件。比如很多应用中普遍使用的标题栏控件。
 - 自绘控件
 	自绘控件继承View类，在View的onDraw方法中完成绘制。
 - 继承控件
 	继承已有的控件，创建新控件，保留继承的父控件的特性，并且还可以引入新特性。下面就以支持横向滑动删除列表项的自定义ListView的实现来介绍。


### RecyclerView用法
 - 基本用法
![](http://www.jcodecraeer.com/uploads/20141021/16791413904189.png)

1. 先在xml布局文件中创建一个RecyclerView的布局
2. 在MainActivity中获取这个RecyclerView，并声明LayoutManager与Adapter
```java
mRecyclerView = (RecyclerView)findViewById(R.id.my_recycler_view);
//创建默认的线性LayoutManager
mLayoutManager = new LinearLayoutManager(this);
//如果想要一个横向的List只要设置LinearLayoutManager如下就行
//mLayoutManager.setOrientation(LinearLayoutManager.HORIZONTAL);
//如果想要一个Grid布局的列表，只要声明LayoutManager为GridLayoutManager即可
//mLayoutManager = new GridLayoutManager(context,columNum);
mRecyclerView.setLayoutManager(mLayoutManager);
//如果可以确定每个item的高度是固定的，设置这个选项可以提高性能
mRecyclerView.setHasFixedSize(true);
//创建并设置Adapter
mAdapter = newMyAdapter(getDummyDatas());
mRecyclerView.setAdapter(mAdapter);
```

3. 接下来的问题就是Adapter的创建：
```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.ViewHolder> {
    public String[] datas = null;
    public MyAdapter(String[] datas) {
        this.datas = datas;
    }
    //创建新View，被LayoutManager所调用
    @Override
    public ViewHolder onCreateViewHolder(ViewGroup viewGroup, int viewType) {
        View view = LayoutInflater.from(viewGroup.getContext()).inflate(R.layout.item,viewGroup,false);
        ViewHolder vh = new ViewHolder(view);
        return vh;
    }
    //将数据与界面进行绑定的操作
    @Override
    public void onBindViewHolder(ViewHolder viewHolder, int position) {
        viewHolder.mTextView.setText(datas[position]);
    }
    //获取数据的数量
    @Override
    public int getItemCount() {
        return datas.length;
    }
    //自定义的ViewHolder，持有每个Item的的所有界面元素
    public static class ViewHolder extends RecyclerView.ViewHolder {
        public TextView mTextView;
        public ViewHolder(View view){
        super(view);
            mTextView = (TextView) view.findViewById(R.id.text);
        }
    }
}
```
 - 点击事件
1. 首先我们在Adapter中创建一个实现点击接口，其中view是点击的Item，data是我们的数据，因为我们想知道我点击的区域部分的数据是什么，以便我下一步进行操作：
```java
public static interface OnRecyclerViewItemClickListener {
	void onItemClick(View view , DataModel data);
}
```
2. 定义完接口，添加接口和设置Adapter接口的方法：
```java
private OnRecyclerViewItemClickListener mOnItemClickListener = null;
public void setOnItemClickListener(OnRecyclerViewItemClickListener listener) {
    this.mOnItemClickListener = listener;
}
```
3. 为Adapter实现OnClickListener方法：
```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.ViewHolder> implements View.OnClickListener{
    @Override
    public ViewHolder onCreateViewHolder(ViewGroup viewGroup, final int i) {
        View view = LayoutInflater.from(viewGroup.getContext()).inflate(R.layout.item, viewGroup, false);
        ViewHolder vh = new ViewHolder(view);
        //将创建的View注册点击事件
        view.setOnClickListener(this);
        return vh;
    }
    @Override
    public void onBindViewHolder(ViewHolder viewHolder, final int i) {
        viewHolder.mTextView.setText(datas.get(i).title);
        //将数据保存在itemView的Tag中，以便点击时进行获取
        viewHolder.itemView.setTag(datas.get(i));
    }
    ...
    @Override
    public void onClick(View v) {
        if (mOnItemClickListener != null) {
            //注意这里使用getTag方法获取数据
            mOnItemClickListener.onItemClick(v,(DataModel)v.getTag());
        }
    }
    ...
}
```
4. Activity或其他地方为RecyclerView添加项目点击事件：
```java
mAdapter = new MyAdapter(getDummyDatas());
mRecyclerView.setAdapter(mAdapter);
mAdapter.setOnItemClickListener(new MyAdapter.OnRecyclerViewItemClickListener() {
    @Override
    public void onItemClick(View view, DataModel data) {
        //DO your fucking bussiness here!
    }
});
```
### 实现Bitmap上传

**HttpURLConnection**

```java
    public String upload(Context context, String filePath) {
        LogUtils.i(TAG, "upload(" + filePath + ")...");

        File file = new File(filePath);
        if (!file.exists()) {
            LogUtils.i(TAG, "file not exists");
            return null;
        }

        StringBuilder result = new StringBuilder();
        int TIME_OUT = 3000;
        String domain = DynamicDomainContract.getUploadServerUrl(context);
        if (TextUtils.isEmpty(domain)) {
            return null;
        }
        String RequestURL = domain + ServerConstants.UPLOAD_BIGEVENT;
        LogUtils.i(TAG, NETWORK + "url: " + RequestURL);
        LogUtils.i(TAG, NETWORK + "filePath : " + filePath);
        String BOUNDARY = UUID.randomUUID().toString(); // 边界标识 随机生成

        String PREFIX = "--", LINE_END = "\r\n";

        String CONTENT_TYPE = "multipart/form-data"; // 内容类型

        String CHARSET = "utf-8";

        URL url;
        try {
            url = new URL(RequestURL);
            HttpURLConnection conn;
            try {
                conn = (HttpURLConnection) url.openConnection();
                conn.setReadTimeout(TIME_OUT);
                conn.setConnectTimeout(TIME_OUT);
                conn.setDoInput(true); // 允许输入流
                conn.setDoOutput(true); // 允许输出流
                conn.setUseCaches(false); // 不允许使用缓存
                conn.setRequestMethod("POST"); // 请求方式
                conn.setRequestProperty("Charset", CHARSET);
                // 设置编码
                setTokenInfo(context);
                conn.setRequestProperty(DEVICEID, mDeviceId);
                conn.setRequestProperty(TOKEN, mAccessToken);
                conn.setRequestProperty("connection", "keep-alive");
                conn.setRequestProperty("Content-Type", CONTENT_TYPE + ";boundary="+ BOUNDARY);

                OutputStream outputSteam = conn.getOutputStream();
                DataOutputStream dos = new DataOutputStream(outputSteam);
                StringBuffer sb = new StringBuffer();
                sb.append(PREFIX);
                sb.append(BOUNDARY);
                sb.append(LINE_END);

                sb.append("Content-Disposition:form-data;name=\"file1\";filename=\""
                        + file.getName() + "\"" + LINE_END);
                sb.append("Content-Type:image/png\r\n\r\n");

                // sb.append(LINE_END);
                LogUtils.i(TAG, "sb.toString = " + sb.toString());
                dos.write(sb.toString().getBytes());

                InputStream is = new FileInputStream(file);
                IOUtils.copy(is, dos);
                is.close();

                dos.write(LINE_END.getBytes());
                byte[] end_data = (PREFIX + BOUNDARY + PREFIX + LINE_END)
                        .getBytes();
                dos.write(end_data);
                dos.flush();
                dos.close();
                int res = conn.getResponseCode();
                LogUtils.i(TAG, "response code:" + res);
                if (res == 200) {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(
                            conn.getInputStream()));
                    String line = null;
                    while ((line = reader.readLine()) != null) {
                        LogUtils.d(TAG, "result : " + line);
                        result.append(line);
                    }
                }
            } catch (IOException e) {
                LogUtils.e(TAG, "upload()", e);
            }
        } catch (MalformedURLException e) {
            LogUtils.e(TAG, "upload()", e);
        }
        LogUtils.d(TAG, "result = " + result.toString());
        return result.toString();
    }
```
 - **编码**
	编码包括`x-www-form-urlencoded` 与 `form-data`。
	**x-www-form-urlencoded**的编码方式是这样：

	> tel=13637829200&password=123456  
	
	**form-data**的编码方式是这样：

  		----WebKitFormBoundary7MA4YWxkTrZu0gW
  		Content-Disposition: form-data; name="tel"

  		13637829200
  		----WebKitFormBoundary7MA4YWxkTrZu0gW
  		Content-Disposition: form-data; name="password"

  		123456
  		----WebKitFormBoundary7MA4YWxkTrZu0gW
x-www-form-urlencoded只能传键值对，但是form-data可以传二进制
**Get与Post区别本质就是参数是放在请求行中还是放在请求体中**

### 实现Bitmap下载
```java
  /**
     * 根据图片的url从服务器获取图片
     *
     * @param view : 需要设置的图片的控件
     * @param context ： 上下文环境
     * @param url ： 图片的url地址
     * @param fileName ： 获取成功之后需要保存到本地的文件名
     */
    public Bitmap download(final Context context, final String fixUrl) {
        LogUtils.i(TAG, "download()...");
        String downloadDomain = DynamicDomainContract.getDownloadServerUrl(context);
        if (TextUtils.isEmpty(downloadDomain)) {
            return null;
        }
        String url = downloadDomain + fixUrl;
        LogUtils.i(TAG, NETWORK + "url: " + url);
        LogUtils.i(TAG, NETWORK + "fixUrl = " + fixUrl);
      //httpGet连接对象
        HttpGet get = new HttpGet(url);
        //取得HttpClient 对象
        if (mHttpClient == null) {
            LogUtils.i(TAG, "mHttpClient is null...");
            setHttpClient();
        }
        HttpParams params = mHttpClient.getParams();
        params.setIntParameter(HttpConnectionParams.CONNECTION_TIMEOUT, CONNECTION_TIMEOUT);
        params.setIntParameter(HttpConnectionParams.SO_TIMEOUT, SOCKET_TIMEOUT);
        setTokenInfo(context);
        get.addHeader(DEVICEID, mDeviceId);
        get.addHeader(TOKEN, mAccessToken);
        try {
            //请求httpClient ，取得HttpRestponse
            HttpResponse response = mHttpClient.execute(get);
            int statusCode = response.getStatusLine().getStatusCode();
            if ((statusCode == HttpStatus.SC_CREATED) || (statusCode == HttpStatus.SC_OK)) {
                //取得相关信息 取得HttpEntiy
                HttpEntity httpEntity = response.getEntity();
                //获得一个输入流
                InputStream is = httpEntity.getContent();
                byte[] byteValue = toByteArray(is);
                Bitmap bitmap = BitmapFactory.decodeByteArray(byteValue, 0, byteValue.length);
                if (is != null) {
                    is.close();
                }
                return bitmap;
            }

        } catch (ClientProtocolException e) {
            LogUtils.e(TAG, "getBigEventBitmap()", e);
        } catch (IOException e) {
            LogUtils.e(TAG, "getBigEventBitmap()", e);
        }
        return null;
    }

    private byte[] toByteArray(InputStream input) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();
        byte[] buffer = new byte[4096];
        int n = 0;
        while (-1 != (n = input.read(buffer))) {
            output.write(buffer, 0, n);
        }
        return output.toByteArray();
    }

```

### 怎样获取Token/uID/DeviceID 
```java
    private void setTokenInfo(Context context) {
        if (context == null) {
            return;
        }
        Cursor cursor = null;
        try {
            Uri uri = Uri
                    .parse(content://com.kinstalk.setupwazard.provider.tokenProvider/tokenInfo");
            cursor = context.getContentResolver().query(uri, null, null, null, null);
            if (cursor == null) {
                return;
            }
            while (cursor.moveToNext()) {
                mUid = cursor.getLong(cursor.getColumnIndex("uid"));
                mDeviceId = cursor.getString(cursor.getColumnIndex("deviceId"));
                mAccessToken = cursor.getString(cursor.getColumnIndex("accessToken"));
            }
        } catch (Exception e) {
            LogUtils.e(TAG, "setTokenInfo()", e);
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
        LogUtils.i(TAG, mUid + "/" + mDeviceId + "/" + mAccessToken);
    }
```

### Json的处理
 - JSONObject -Java自带
 - GSON -Google提供
 - FastJSON -阿里开源
 - Jackson

```Java
    public ArrayList<HolidayInfo> getHolidayInfo(Context context, String result) {
        Log.i(TAG, "getHolidayInfo: " + result);
        // result =
        // "{" +
        // "\"c\": 0," +
        // "\"m\": \"\"," +
        // "\"d\": {" +
        // "\"lastAccessTime\": 1442284873089," +
        // "\"calendarConfig\": [" +
        // "{" +
        // "\"id\": 123,"+
        // "\"summary\": \"十一黄金金周\","+
        // "\"startDate\": 2015-10-10," +
        // "\"endDate\": 2015-10-10," +
        // "\"status\": 0,"+
        // "\"type\": 1,"+
        // "\"createTime\": 1442284873089,"+
        // "\"updateTime\": 1442284873089"+
        // "}"+
        // "]" +
        // "}"+
        // "}";
        if (TextUtils.isEmpty(result)) {
            return null;
        }
        try {
            JSONObject js = new JSONObject(result);
            int retCode = js.getInt("c");
            if (retCode == 0) {
                JSONObject j = js.getJSONObject("d");
                long time = j.getLong("lastAccessTime");
                CalendarSharedPrefUtils.getInstance().saveGetHolidayInfoTime(context, time);
                JSONArray jsonArray = j.getJSONArray("calendarConfig");
                ArrayList<HolidayInfo> list = new ArrayList<>();
                for (int i = 0; i < jsonArray.length(); i++) {
                    JSONObject json = jsonArray.getJSONObject(i);
                    HolidayInfo info = new HolidayInfo();
                    for (Iterator iter = json.keys(); iter.hasNext();) {
                        String key = (String) iter.next();
                        String value = json.getString(key);
                        if (ServerConstants.NULL.equalsIgnoreCase(value)
                                || TextUtils.isEmpty(value)) {
                            continue;
                        }
                        if (ServerConstants.ID.equals(key)) {
                            info.holidayid = Long.parseLong(value);
                        } else if (ServerConstants.SUMMARY.equals(key)) {
                            info.summary = value;
                        } else if (ServerConstants.STARTDATE.equals(key)) {
                            String[] date = value.split("-");
                            info.year = Integer.parseInt(date[0]);
                            info.month = Integer.parseInt(date[1]);
                            info.day = Integer.parseInt(date[2]);
                        } else if (ServerConstants.STATUS.equals(key)) {
                            info.status = Integer.parseInt(value);
                        } else if (ServerConstants.TYPE.equals(key)) {
                            info.type = Integer.parseInt(value);
                        } else if (ServerConstants.CREATETIME.equals(key)) {
                            info.createTime = Long.parseLong(value);
                        } else if (ServerConstants.UPDATETIME.equals(key)) {
                            info.updateTime = Long.parseLong(value);
                        }
                    }
                    list.add(info);
                }
                return list;
            }
        } catch (JSONException e) {
            LogUtils.e(TAG, "getHolidayInfo()", e);
        }
        return null;
    }
```


### 法定节假日如何实现
```json
{
	"c":0,
	"m":"",
	"d":
	"lastAccessTime": 1442284873089,//拉取最大时间（若calendars内容为空，则返回Q-Love端传来的上次拉取时间作为本次请求的返回值）
	"calendars":
	[
		{
			"id":123,
			"summary":⻩⾦周,//休息⽇描述
			"start_date":2015-10-01,//开始⽇期
			"end_date":2015-10-01,//结束⽇期
			"status":0,//状态 0:正常;1:已删除;2:已发布
			"type":0,//类型 1:上班;2:休息⽇
			"createTime":1442284873089,
			"updateTime":1442284873089
		}
	]
}
```


### UI替换方面的心得
UI设计图导入Photoshop对每一个控件的位置进行测量，避免返工

### Layout调优
`<include></include>`
`<merge></merge>`
`<ViewStab></ViewStub>`

### Android 动画

 - View动画(补间动画)包括：平移(Translate)、旋转(Rotate)、缩放(Scale)、透明度(Alpha)，View动画是一种渐近式动画

```java
    private void loadAnim(){
        Animation backAnimation = (Animation) AnimationUtils.loadAnimation(mContext,R.anim.translate_up);
        mBack.startAnimation(backAnimation);
    }

```
	Activity进入退出动画（要放在finish（）之后）
    overridePendingTransition(R.anim.activity_enter, R.anim.activity_exit);

 - 帧动画(AnimationDrawable)：图片切换动画

```xml
  <?xml version="1.0" encoding="utf-8"?>
  <animation-list xmlns:android="http://schemas.android.com/apk/res/android"
      android:oneshot="false">
      <item android:drawable="@mipmap/ic_launcher" android:duration="100"/>
      <item android:drawable="@mipmap/ic_launcher" android:duration="100"/>
  </animation-list>
```
```java
  view.setBackgroundResource(R.drawable.animation_list);
  AnimationDrawable drawable = (AnimationDrawable)view.getBackground();
  drawable.start();
```
 - 属性动画：通过动态改变对象的属性达到动画效果
 
```java
  ObjectAnimator.ofFloat(view, "translationY", 10).start();
```
