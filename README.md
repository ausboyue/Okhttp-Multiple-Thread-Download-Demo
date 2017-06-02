# Okhttp-Multiple-Thread-Download-Demo
Android Okhttp多线程断点续传下载Demo
<p>
	<span style="white-space:pre"></span><span style="white-space:pre"></span>&nbsp; &nbsp; &nbsp; &nbsp; 时下，得力于谷歌官方强烈推荐，加上自身优秀特性，OkHttp成了当前最火的HTTP框架之一。现在公司的项目我也全都换成了基于OkHttp+Gson底层网络访问和解析平台。
</p>
<p>
	<span style="white-space:pre"></span>&nbsp; &nbsp; &nbsp; &nbsp; 最近项目需要使用到断点下载功能，笔者比较喜欢折腾，想方设法抛弃SharedPreferences，尤其是sqlite作记录辅助，改用临时记录文件的形式记录下载进度，本文以断点下载为例。先看看demo运行效果图：
</p>
<p>
	&nbsp; &nbsp; &nbsp;<img src="http://img.blog.csdn.net/20170504002218302?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXVzYm95dWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" align="middle" width="240" height="426" alt="" />
</p>
<p>
	<br />
	
</p>
<p>
	<span style="white-space:pre"></span><span style="white-space:pre"></span>&nbsp; &nbsp; &nbsp; &nbsp; 断点续传：记录上次上传（下载）节点位置，下次接着该位置继续上传（下载）。多线程断点续传下载则是根据目标下载文件长度，尽可能地等分给多个线程同时下载文件块，当各个线程全部完成下载后，将文件块合并成一个文件，即目标文件。多线程断点续传不仅为用户避免了断网等突发事故需要重新下载浪费流量的尴尬局面，也大大提高了下载速率，当然，不是线程越多越好，网络带宽才是硬道理！以下为原理图：
</p>
<blockquote style="margin:0 0 0 40px; border:none; padding:0px">
	<blockquote style="margin:0 0 0 40px; border:none; padding:0px">
		<blockquote style="margin:0 0 0 40px; border:none; padding:0px">
			<p>
			</p>
		</blockquote>
	</blockquote>
</blockquote>
<p>
	&nbsp; &nbsp;&nbsp;<img src="http://img.blog.csdn.net/20170504223713505?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXVzYm95dWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" align="middle" width="600" height="311" alt="" />
</p>
<blockquote style="margin:0 0 0 40px; border:none; padding:0px">
	<blockquote style="margin:0 0 0 40px; border:none; padding:0px">
		<blockquote style="margin:0 0 0 40px; border:none; padding:0px">
			<p>
			</p>
		</blockquote>
	</blockquote>
</blockquote>
<p>
	&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; java，android中可以使用RandomAccessFile类生成一个同目标文件大小的占位文件，以便于各个线程可以同时操作该文件，并写入各线程实时下载的数据。
</p>
<p>
	<span style="white-space:pre"></span>&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 下面贴出OkHttp实现的单个多线程下载任务类的DownloadTask.java文件：
</p>
<pre code_snippet_id="2370854" snippet_file_name="blog_20170503_1_936933" name="code" class="java">package cn.icheny.download;

import android.os.Handler;
import android.os.Message;
import java.io.Closeable;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.RandomAccessFile;
import okhttp3.Call;
import okhttp3.Response;

/**
 * 多线程下载任务
 * Created by Cheny on 2017/05/03.
 */

public class DownloadTask extends Handler {

    private final int THREAD_COUNT = 4;//下载线程数量
    private FilePoint mPoint;
    private long mFileLength;//文件大小

    private boolean isDownloading = false;//是否正在下载
    private int childCanleCount;//子线程取消数量
    private int childPauseCount;//子线程暂停数量
    private int childFinishCount;//子线程完成下载数量
    private HttpUtil mHttpUtil;//http网络通信工具
    private long[] mProgress;//各个子线程下载进度集合
    private File[] mCacheFiles;//各个子线程下载缓存数据文件
    private File mTmpFile;//临时占位文件
    private boolean pause;//是否暂停
    private boolean cancel;//是否取消下载

    private final int MSG_PROGRESS = 1;//进度
    private final int MSG_FINISH = 2;//完成下载
    private final int MSG_PAUSE = 3;//暂停
    private final int MSG_CANCEL = 4;//暂停
    private DownloadListner mListner;//下载回调监听

    DownloadTask(FilePoint point, DownloadListner l) {
        this.mPoint = point;
        this.mListner = l;
        this.mProgress = new long[THREAD_COUNT];
        this.mCacheFiles = new File[THREAD_COUNT];
        this.mHttpUtil = HttpUtil.getInstance();
    }

    /**
     * 开始下载
     */
    public synchronized void start() {
        try {
            if (isDownloading) return;
            isDownloading = true;
            mHttpUtil.getContentLength(mPoint.getUrl(), new okhttp3.Callback() {
                @Override
                public void onResponse(Call call, Response response) throws IOException {
                    if (response.code() != 200) {
                        close(response.body());
                        resetStutus();
                        return;
                    }
                    // 获取资源大小
                    mFileLength = response.body().contentLength();
                    close(response.body());
                    // 在本地创建一个与资源同样大小的文件来占位
                    mTmpFile = new File(mPoint.getFilePath(), mPoint.getFileName() + &quot;.tmp&quot;);
                    if (!mTmpFile.getParentFile().exists()) mTmpFile.getParentFile().mkdirs();
                    RandomAccessFile tmpAccessFile = new RandomAccessFile(mTmpFile, &quot;rw&quot;);
                    tmpAccessFile.setLength(mFileLength);
                    /*将下载任务分配给每个线程*/
                    long blockSize = mFileLength / THREAD_COUNT;// 计算每个线程理论上下载的数量.

                    /*为每个线程配置并分配任务*/
                    for (int threadId = 0; threadId &lt; THREAD_COUNT; threadId++) {
                        long startIndex = threadId * blockSize; // 线程开始下载的位置
                        long endIndex = (threadId + 1) * blockSize - 1; // 线程结束下载的位置
                        if (threadId == (THREAD_COUNT - 1)) { // 如果是最后一个线程,将剩下的文件全部交给这个线程完成
                            endIndex = mFileLength - 1;
                        }
                        download(startIndex, endIndex, threadId);// 开启线程下载
                    }
                }

                @Override
                public void onFailure(Call call, IOException e) {
                    resetStutus();
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
            resetStutus();
        }
    }

    /**
     * 下载
     * @param startIndex 下载起始位置
     * @param endIndex  下载结束位置
     * @param threadId 线程id
     * @throws IOException
     */
    public void download(final long startIndex, final long endIndex, final int threadId) throws IOException {
        long newStartIndex = startIndex;
        // 分段请求网络连接,分段将文件保存到本地.
        // 加载下载位置缓存数据文件
        final File cacheFile = new File(mPoint.getFilePath(), &quot;thread&quot; + threadId + &quot;_&quot; + mPoint.getFileName() + &quot;.cache&quot;);
        mCacheFiles[threadId] = cacheFile;
        final RandomAccessFile cacheAccessFile = new RandomAccessFile(cacheFile, &quot;rwd&quot;);
        if (cacheFile.exists()) {// 如果文件存在
            String startIndexStr = cacheAccessFile.readLine();
            try {
                newStartIndex = Integer.parseInt(startIndexStr);//重新设置下载起点
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }
        final long finalStartIndex = newStartIndex;
        mHttpUtil.downloadFileByRange(mPoint.getUrl(), finalStartIndex, endIndex, new okhttp3.Callback() {
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (response.code() != 206) {// 206：请求部分资源成功码，表示服务器支持断点续传
                    resetStutus();
                    return;
                }
                InputStream is = response.body().byteStream();// 获取流
                RandomAccessFile tmpAccessFile = new RandomAccessFile(mTmpFile, &quot;rw&quot;);// 获取前面已创建的文件.
                tmpAccessFile.seek(finalStartIndex);// 文件写入的开始位置.
                  /*  将网络流中的文件写入本地*/
                byte[] buffer = new byte[1024 &lt;&lt; 2];
                int length = -1;
                int total = 0;// 记录本次下载文件的大小
                long progress = 0;
                while ((length = is.read(buffer)) &gt; 0) {//读取流
                    if (cancel) {
                        close(cacheAccessFile, is, response.body());//关闭资源
                        cleanFile(cacheFile);//删除对应缓存文件
                        sendMessage(MSG_CANCEL);
                        return;
                    }
                    if (pause) {
                        //关闭资源
                        close(cacheAccessFile, is, response.body());
                        //发送暂停消息
                        sendMessage(MSG_PAUSE);
                        return;
                    }
                    tmpAccessFile.write(buffer, 0, length);
                    total += length;
                    progress = finalStartIndex + total;

                    //将该线程最新完成下载的位置记录并保存到缓存数据文件中
                    //建议转成Base64码，防止数据被修改，导致下载文件出错（若真有这样的情况，这样的朋友可真是无聊透顶啊）
                    cacheAccessFile.seek(0);
                    cacheAccessFile.write((progress + &quot;&quot;).getBytes(&quot;UTF-8&quot;));
                    //发送进度消息
                    mProgress[threadId] = progress - startIndex;
                    sendMessage(MSG_PROGRESS);
                }
                //关闭资源
                close(cacheAccessFile, is, response.body());
                // 删除临时文件
                cleanFile(cacheFile);
                //发送完成消息
                sendMessage(MSG_FINISH);
            }

            @Override
            public void onFailure(Call call, IOException e) {
                isDownloading = false;
            }
        });
    }
    /**
     * 轮回消息回调
     *
     * @param msg
     */
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        if (null == mListner) {
            return;
        }
        switch (msg.what) {
            case MSG_PROGRESS://进度
                long progress = 0;
                for (int i = 0, length = mProgress.length; i &lt; length; i++) {
                    progress += mProgress[i];
                }
                mListner.onProgress(progress * 1.0f / mFileLength);
                break;
            case MSG_PAUSE://暂停
                childPauseCount++;
                if (childPauseCount % THREAD_COUNT != 0) return;//等待所有的线程完成暂停，真正意义的暂停，以下同理
                resetStutus();
                mListner.onPause();
                break;
            case MSG_FINISH://完成
                childFinishCount++;
                if (childFinishCount % THREAD_COUNT != 0) return;
                mTmpFile.renameTo(new File(mPoint.getFilePath(), mPoint.getFileName()));//下载完毕后，重命名目标文件名
                resetStutus();
                mListner.onFinished();
                break;
            case MSG_CANCEL://取消
                childCanleCount++;
                if (childCanleCount % THREAD_COUNT != 0) return;
                resetStutus();
                mProgress = new long[THREAD_COUNT];
                mListner.onCancel();
                break;
        }
    }

    /**
     * 发送消息到轮回器
     *
     * @param what
     */
    private void sendMessage(int what) {
        //发送暂停消息
        Message message = new Message();
        message.what = what;
        sendMessage(message);
    }


    /**
     * 关闭资源
     *
     * @param closeables
     */
    private void close(Closeable... closeables) {
        int length = closeables.length;
        try {
            for (int i = 0; i &lt; length; i++) {
                Closeable closeable = closeables[i];
                if (null != closeable)
                    closeables[i].close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            for (int i = 0; i &lt; length; i++) {
                closeables[i] = null;
            }
        }
    }

    /**
     * 暂停
     */
    public void pause() {
        pause = true;
    }

    /**
     * 取消
     */
    public void cancel() {
        cancel = true;
        cleanFile(mTmpFile);
        if (!isDownloading) {//针对非下载状态的取消，如暂停
            if (null != mListner) {
                cleanFile(mCacheFiles);
                resetStutus();
                mListner.onCancel();
            }
        }
    }

    /**
     * 重置下载状态
     */
    private void resetStutus() {
        pause = false;
        cancel = false;
        isDownloading = false;
    }

    /**
     * 删除临时文件
     */
    private void cleanFile(File... files) {
        for (int i = 0, length = files.length; i &lt; length; i++) {
            if (null != files[i])
                files[i].delete();
        }
    }

    /**
     * 获取下载状态
     * @return boolean
     */
    public boolean isDownloading() {
        return isDownloading;
    }
}
</pre>
<p>
	&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;先网络请求获取文件的长度mFileLength，根据长度借助RandomAccessFile类在本地生成相同长度的占位文件mTmpFile，再根据线程数量THREAD_COUNT拆分下载任务，最后for循环出THREAD_COUNT数量的异步请求下载拆分内容（字节）并从mTmpFile的对应位置写入mTmpFile，每个线程（任务）每写入一定的数据后将任务的下载进度写入通过RandomAccessFile生成的对应任务的记录缓存文件中，以便于下次下载读取该线程已下载的进度。注释比较多，好像也没啥好解释的，有问题的朋友下方留言。
</p>
<p>
	&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 在贴上由OkHttp简单封装的网络请求工具类HttpUtil的.java文件：
</p>
<pre code_snippet_id="2370854" snippet_file_name="blog_20170504_2_1740984" name="code" class="java">package cn.icheny.download;

import java.io.IOException;
import java.util.concurrent.TimeUnit;
import okhttp3.Call;
import okhttp3.Callback;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

/**
 * Http网络工具,基于OkHttp
 * Created by Cheny on 2017/05/03.
 */

public class HttpUtil {
    private OkHttpClient mOkHttpClient;
    private static HttpUtil mInstance;
    private final static long CONNECT_TIMEOUT = 60;//超时时间，秒
    private final static long READ_TIMEOUT = 60;//读取时间，秒
    private final static long WRITE_TIMEOUT = 60;//写入时间，秒

    /**
     * @param url        下载链接
     * @param startIndex 下载起始位置
     * @param endIndex   结束为止
     * @param callback   回调
     * @throws IOException
     */
    public void downloadFileByRange(String url, long startIndex, long endIndex, Callback callback) throws IOException {
        // 创建一个Request
        // 设置分段下载的头信息。 Range:做分段数据请求,断点续传指示下载的区间。格式: Range bytes=0-1024或者bytes:0-1024
        Request request = new Request.Builder().header(&quot;RANGE&quot;, &quot;bytes=&quot; + startIndex + &quot;-&quot; + endIndex)
                .url(url)
                .build();
        doAsync(request, callback);
    }

    public void getContentLength(String url, Callback callback) throws IOException {
        // 创建一个Request
        Request request = new Request.Builder()
                .url(url)
                .build();
        doAsync(request, callback);
    }

    /**
     * 同步GET请求
     */
    public void doGetSync(String url) throws IOException {
        //创建一个Request
        Request request = new Request.Builder()
                .url(url)
                .build();
        doSync(request);
    }

    /**
     * 异步请求
     */
    private void doAsync(Request request, Callback callback) throws IOException {
        //创建请求会话
        Call call = mOkHttpClient.newCall(request);
        //同步执行会话请求
        call.enqueue(callback);
    }

    /**
     * 同步请求
     */
    private Response doSync(Request request) throws IOException {
        //创建请求会话
        Call call = mOkHttpClient.newCall(request);
        //同步执行会话请求
        return call.execute();
    }


    /**
     * @return HttpUtil实例对象
     */
    public static HttpUtil getInstance() {
        if (null == mInstance) {
            synchronized (HttpUtil.class) {
                if (null == mInstance) {
                    mInstance = new HttpUtil();
                }
            }
        }
        return mInstance;
    }

    /**
     * 构造方法,配置OkHttpClient
     */
    private&nbsp;HttpUtil() {
        //创建okHttpClient对象
        OkHttpClient.Builder builder = new OkHttpClient.Builder()
                .connectTimeout(CONNECT_TIMEOUT, TimeUnit.SECONDS)
                .writeTimeout(READ_TIMEOUT, TimeUnit.SECONDS)
                .readTimeout(WRITE_TIMEOUT, TimeUnit.SECONDS);
        mOkHttpClient = builder.build();
    }
}
</pre>
<p>
	&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;header(&quot;RANGE&quot;, &quot;bytes=&quot; + startIndex + &quot;-&quot; + endIndex)，在OkHttp请求头中添加RANGE（范围）参数，告诉服务器需要下载文件内容的始末位置。鉴于OkHttp的火热程度，好像人人都会使用OkHttp，我就不赘言了。
</p>
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;为了更清晰的教程思路，这里也贴出FilePoint.java：
<pre code_snippet_id="2370854" snippet_file_name="blog_20170504_3_4390943" name="code" class="java">package cn.icheny.download;

/**
 * 目标文件
 * Created by Cheny on 2017/05/03.
 */

public class FilePoint {
    private String fileName;//文件名
    private String url;//文件url
    private String filePath;//文件下载路径

    public FilePoint(String url) {
        this.url = url;
    }

    public FilePoint(String filePath, String url) {
        this.filePath = filePath;
        this.url = url;
    }

    public FilePoint(String url, String filePath, String fileName) {
        this.url = url;
        this.filePath = filePath;
        this.fileName = fileName;
    }

    public String getFileName() {
        return fileName;
    }

    public void setFileName(String fileName) {
        this.fileName = fileName;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getFilePath() {
        return filePath;
    }

    public void setFilePath(String filePath) {
        this.filePath = filePath;
    }

}</pre>
&nbsp; &nbsp; &nbsp; 下面是下载管理器DownloadManager&nbsp;代码，统一管理所有文件的下载任务：
<pre code_snippet_id="2370854" snippet_file_name="blog_20170504_4_3104187" name="code" class="java">package cn.icheny.download;

import android.os.Environment;
import android.text.TextUtils;
import java.io.File;
import java.util.HashMap;
import java.util.Map;

/**
 * 下载管理器，断点续传
 *
 * @author Cheny
 */
public class DownloadManager {

    private String DEFAULT_FILE_DIR;//默认下载目录
    private Map&lt;String, DownloadTask&gt; mDownloadTasks;//文件下载任务索引，String为url,用来唯一区别并操作下载的文件
    private static DownloadManager mInstance;

    /**
     * 下载文件
     */
    public void download(String... urls) {
        for (int i = 0, length = urls.length; i &lt; length; i++) {
            String url = urls[i];
            if (mDownloadTasks.containsKey(url)) {
                mDownloadTasks.get(url).start();
            }
        }
    }

    /**
     * 通过url获取下载文件的名称
     */
    public String getFileName(String url) {
        return url.substring(url.lastIndexOf(&quot;/&quot;) + 1);
    }

    /**
     * 暂停
     */
    public void pause(String... urls) {
        for (int i = 0, length = urls.length; i &lt; length; i++) {
            String url = urls[i];
            if (mDownloadTasks.containsKey(url)) {
                mDownloadTasks.get(url).pause();
            }
        }
    }

    /**
     * 取消下载
     */
    public void cancel(String... urls) {
        for (int i = 0, length = urls.length; i &lt; length; i++) {
            String url = urls[i];
            if (mDownloadTasks.containsKey(url)) {
                mDownloadTasks.get(url).cancel();
            }
        }
    }

    /**
     * 添加下载任务
     */
    public void add(String url, DownloadListner l) {
        add(url, null, null, l);
    }

    /**
     * 添加下载任务
     */
    public void add(String url, String filePath, DownloadListner l) {
        add(url, filePath, null, l);
    }

    /**
     * 添加下载任务
     */
    public void add(String url, String filePath, String fileName, DownloadListner l) {
        if (TextUtils.isEmpty(filePath)) {//没有指定下载目录,使用默认目录
            filePath = getDefaultDirectory();
        }
        if (TextUtils.isEmpty(fileName)) {
            fileName = getFileName(url);
        }
        mDownloadTasks.put(url, new DownloadTask(new FilePoint(url, filePath, fileName), l));
    }

    /**
     * 获取默认下载目录
     *
     * @return
     */
    private String getDefaultDirectory() {
        if (TextUtils.isEmpty(DEFAULT_FILE_DIR)) {
            DEFAULT_FILE_DIR = Environment.getExternalStorageDirectory().getAbsolutePath()
                    + File.separator + &quot;icheny&quot; + File.separator;
        }
        return DEFAULT_FILE_DIR;
    }

    /**
     * 是否正在下载
     * @param urls
     * @return boolean
     */
    public boolean isDownloading(String... urls) {
        boolean result = false;
        for (int i = 0, length = urls.length; i &lt; length; i++) {
            String url = urls[i];
            if (mDownloadTasks.containsKey(url)) {
                result = mDownloadTasks.get(url).isDownloading();
            }
        }
        return result;
    }

    public static DownloadManager getInstance() {
        if (mInstance == null) {
            synchronized (DownloadManager.class) {
                if (mInstance == null) {
                    mInstance = new DownloadManager();
                }
            }
        }
        return mInstance;
    }
    /**
     * 初始化下载管理器
     */
    private&nbsp;DownloadManager() {
        mDownloadTasks = new HashMap&lt;&gt;();
    }
}</pre>
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;下载管理器通过一个Map将下载链接（url，教程图方便使用url的方式。建议使用其他唯一标识，毕竟一般url长度都很长，会影响一定性能。另外，考虑一个项目中可能需要下载同一个文件到不同的目录，url做索引显得生硬）与对应的下载任务(&nbsp;DownloadTask&nbsp;)绑定在一起，以便于根据url判断或获取对应的下载任务，进行下载,取消和暂停等操作。
<p>
</p>
<p>
	<span style="white-space:pre"></span>&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;OK，时间关系，文章到此结束，有问题或需要Demo源码的朋友下方留言。半夜了。。。浓浓的倦意。。。
</p>
<p>
	<strong><br />
	</strong>
</p>
<p>
	<strong><span style="font-size:12px"><span style="white-space:pre"></span>&nbsp; &nbsp; &nbsp; &nbsp; 2017年6月2日更新：鉴于CSDN库无缘无故把我以前文章上传的Demo源码以及库弄没了，决定还是传github靠谱，下面贴上Demo源码地址：</span></strong>
</p>
<p>
	<strong><span style="font-size:12px">&nbsp; &nbsp; &nbsp; &nbsp; https://github.com/ausboyue/Okhttp-Multiple-Thread-Download-Demo &nbsp;</span></strong>
</p>
<p>
	<strong><span style="font-size:12px"><span style="white-space:pre"></span>临时赶时间写的，难免有些bug，有问题请及时下方反馈。。。</span></strong>
</p>
<p>
	<br />
	
</p>
<p>
	<br />
	
</p>
<p>
	<br />
	
</p>
<br />

<p>
</p>