---
layout: post
title: "Android录制音频并使用Lame转成mp3"
date: 2018-09-13
description: "Android录制音频并使用Lame转成mp3"
tag: multimedia
---  
这篇文章主要介绍在Android平台上使用AudioRecord采集声音数据，于需要上传以及在其他平台设备上播放，所以使用Lame库将PCM数据进行编码转成Mp3格式，有关于声音采集的基础知识可以参考这篇笔记[声音采集-笔记](https://www.jianshu.com/p/3846b9fadad8)
### 声音录制
Android中使用AudioRecord录制声音，根据上面讲述的声音采集原理，需要传递给AudioRecord采样频率、采样位数和声道数，除此之外还需要传入两个参数，一个是声音源，一个是缓冲区大小。

#### 权限
在Android中录制声音需要相应的权限，6.0需要动态申请权限。
```
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```
#### 初始化AudioRecord
```
  public AudioRecord(int audioSource, int sampleRateInHz, int channelConfig, int audioFormat,
            int bufferSizeInBytes)
```

##### audioSource
声音源(在MediaRecorder.AudioSource中进行定义),支持的音频源有如下几种，这里我们使用的是MIC。
```
/** 默认声音 **/
public static final int DEFAULT = 0;
/** 麦克风声音 */
public static final int MIC = 1;
/** 通话上行声音 */
public static final int VOICE_UPLINK = 2;
/** 通话下行声音 */
public static final int VOICE_DOWNLINK = 3;
/** 通话上下行声音 */
public static final int VOICE_CALL = 4;
/** 根据摄像头转向选择麦克风*/
public static final int CAMCORDER = 5;
/** 对麦克风声音进行声音识别，然后进行录制 */
public static final int VOICE_RECOGNITION = 6;
/** 对麦克风中类似ip通话的交流声音进行识别，默认会开启回声消除和自动增益 */
public static final int VOICE_COMMUNICATION = 7;
/** 录制系统内置声音 */
public static final int REMOTE_SUBMIX = 8;
```
##### sampleRateInHz
第二个参数就是采样频率
```
44100Hz is currently the only
     *   rate that is guaranteed to work on all devices, but other rates such as 22050,
     *   16000, and 11025 may work on some devices.
```
根据文档可以看到，Android系统要求所有的设备都要支持44100HZ的采样频率，而其他的在一些设备上不一定支持。
```
8000, 11025, 16000, 22050, 44100, 48000
```
上面是一些常用的采样频率，可以通过如下代码获取手机支持的音频采样率：
```
public void getValidSampleRates() {
    for (int rate : new int[] {8000, 11025, 16000, 22050, 44100}) {  // add the rates you wish to check against
        int bufferSize = AudioRecord.getMinBufferSize(rate, AudioFormat.CHANNEL_CONFIGURATION_DEFAULT, AudioFormat.ENCODING_PCM_16BIT);
        if (bufferSize > 0) {
            // buffer size is valid, Sample rate supported

        }
    }
}
```
##### channelConfig
```
See {@link AudioFormat#CHANNEL_IN_MONO} and
     *   {@link AudioFormat#CHANNEL_IN_STEREO}.  {@link AudioFormat#CHANNEL_IN_MONO} is guaranteed
     *   to work on all devices.
```
MONO是单声道，而STEREO是立体声，想要在所有设备上都适用的话，推荐使用单声道。
##### audioFormat
即我们所说的采样位数。
```
 See {@link AudioFormat#ENCODING_PCM_8BIT}, {@link AudioFormat#ENCODING_PCM_16BIT},
     *   and {@link AudioFormat#ENCODING_PCM_FLOAT}.
```
常用的是ENCODING_PCM_8BIT，和ENCODING_PCM_16BIT，ENCODING_PCM_16BIT能够兼容大多数设备。
想要进一步了解PCM格式的编码的可以看雷神的这篇文章。

[视音频数据处理入门：PCM音频采样数据处理](https://blog.csdn.net/leixiaohua1020/article/details/50534316)

##### bufferSizeInBytes
缓冲区的大小，采集到的数据会先写到缓冲区，之后从缓冲区中读取数据，从而获取到麦克风录制的音频数据。在Android中不同的声道数、采样位数和采样频率会有不同的最小缓冲区大小，当AudioRecord传入的缓冲区大小小于最小缓冲区大小的时候则会初始化失败。大的缓冲区大小可以达到更为平滑的录制效果，相应的也会带来更大一点的延时。
```
mBufferSize=AudioRecord.getMinBufferSize(sampleRateInHz,
				channelConfig, audioFormat);
```
通过上面的代码可以获取到最小缓冲区的大小。
在我们自己使用lame对pcm数据进行编码时，需要周期性的通知，所以需要将bufferSize像上取整到满足周期的大小。
```
private static final int FRAME_COUNT = 160;
/**
*bytesPerFrame
*PCM_8BIT 1字节
*PCM_16BIT 2字节
**/
int frameSize = mBufferSize / bytesPerFrame;
if (frameSize % FRAME_COUNT != 0) {
	frameSize += (FRAME_COUNT - frameSize % FRAME_COUNT);
	mBufferSize = frameSize * bytesPerFrame;
}
```
#### 读取数据
AudioRecord可以通过下面的方法进行数据读取。读取失败的话会返回失败码。
```
public int read(@NonNull byte[] audioData, int offsetInBytes, int sizeInBytes) {    
        return read(audioData, offsetInBytes, sizeInBytes, READ_BLOCKING);
}
```
#### 监听AudioRecord进行转码
给AudioRecord设置刷新监听，待录音帧数每次达到FRAME_COUNT，就通知转换线程转换一次数据。
```
audioRecord.setRecordPositionUpdateListener(OnRecordPositionUpdateListener listener, Handler handler);
audioRecord.setPositionNotificationPeriod(FRAME_COUNT);
```
在OnRecordPositionUpdateListener的onPeriodicNotification(AudioRecord recorder)的回调方法中就可以使用Lame对读取到的数据进行编码，然后写入文件。

### 导入lame库
Android studio已经支持使用CMake了，所以这里就使用CMake来集成lame。如何创建项目可以参考我之前的这篇文章[《android opencv JNI开发环境搭建》](https://www.jianshu.com/p/4566553a4831)。
##### 下载Lame源码
[下载地址](https://sourceforge.net/projects/lame/files/lame/)。
##### 修改Lame内容
1. 下载完之后解压，然后找到libmp3lame文件夹，将里面的.c和.h文件全部复制到项目的cpp目录中。
注意：libmp3lame文件夹内还包含其他文件夹，不用管它。
然后，再找到include文件夹，将lame.h文件拷贝到cpp目录中。（总共43个文件）
2. 接下来需要将源文件导入到项目中修改CMakeLists将Lame的源码加入。
```
aux_source_directory(src/main/cpp/libmp3lame SRC_LIST)

add_library(lamemp3
             SHARED
             src/main/cpp/native-lib.cpp
              ${SRC_LIST})
```
3.移植修改
首先，需要对lame中的三个文件进行一些小改动。

- fft.c中47行将vector/lame_intrin.h这个头文件注释了或者去掉
```
#ifdef HAVE_CONFIG_H
# include <config.h>
#endif

#include "lame.h"
#include "machine.h"
#include "encoder.h"
#include "util.h"
#include "fft.h"

//#include "vector/lame_intrin.h"
```

- 修改set_get.h文件的24行的#include“lame.h”
```
#ifndef __SET_GET_H__
#define __SET_GET_H__

#include "lame.h"
```

- 将util.h文件的574行的”extern ieee754_float32_t fast_log2(ieee754_float32_t x);”
替换为 “extern float fast_log2(float x);”因为android下不支持该类型。

这些跟ndk-builde是一样的，网上有很多教程。
然后，需要修改app -> build.gradle文件
```
android {
...
    defaultConfig {
    ...
        externalNativeBuild{
            cmake{
                cFlags "-DSTDC_HEADERS"
            }
        }
    }
}
```
添加-D标志的意思就是给编译器添加宏定义。那么-DSTDC_HEADERS就相当于给项目增加一句"#define STDC_HEADERS"。
我们打开machine.h文件看一下第34行：
```
#ifdef STDC_HEADERS
# include <stdlib.h>
# include <string.h>
#else
# ifndef HAVE_STRCHR
#  define strchr index
#  define strrchr rindex
# endif
char   *strchr(), *strrchr();
# ifndef HAVE_MEMCPY
#  define memcpy(d, s, n) bcopy ((s), (d), (n))
#  define memmove(d, s, n) bcopy ((s), (d), (n))
# endif
#endif
```
意思很明白，如果没有定义STDC_HEADERS这个宏则会用到bcopy方法，而这个方法我们根本没有，于是就报错了。

##### 测试
打开native-lib.cpp文件，进行修改
```
extern "C"
JNIEXPORT jstring

JNICALL
Java_zeller_com_mp3recorder_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(get_lame_version());
}
```
app中显示Lame的版本信息说明导入Lame库成功。

### 编写JNI代码
我们需要Lame提供如下几个方法供Java层调用
```
    public native static void close();

    public native static int encode(short[] buffer_l, short[] buffer_r, int samples, byte[] mp3buf);

    public native static int flush(byte[] mp3buf);

    public native static void init(int inSampleRate, int outChannel, int outSampleRate, int outBitrate, int quality);
```

##### init方法,初始化Lame
```
static lame_global_flags *glf = NULL;

extern "C"
JNIEXPORT void JNICALL
Java_zeller_com_mp3recorder_Utils_LameUtils_init(JNIEnv *env, jclass type, jint inSampleRate,
                                                 jint outChannel, jint outSampleRate,
                                                 jint outBitrate, jint quality) {
    if (glf != NULL) {
        lame_close(glf);
        glf = NULL;
    }
    glf = lame_init();
    lame_set_in_samplerate(glf, inSampleRate);
    lame_set_num_channels(glf, outChannel);
    lame_set_out_samplerate(glf, outSampleRate);
    lame_set_brate(glf, outBitrate);
    lame_set_quality(glf, quality);
    lame_init_params(glf);
}
```
##### encode方法，将PCM编码成MP3格式
```
extern "C"
JNIEXPORT jint JNICALL
Java_zeller_com_mp3recorder_Utils_LameUtils_encode(JNIEnv *env, jclass type, jshortArray buffer_l_,
                                                   jshortArray buffer_r_, jint samples,
                                                   jbyteArray mp3buf_) {
    jshort *buffer_l = env->GetShortArrayElements(buffer_l_, NULL);
    jshort *buffer_r = env->GetShortArrayElements(buffer_r_, NULL);
    jbyte *mp3buf = env->GetByteArrayElements(mp3buf_, NULL);

    const jsize mp3buf_size = env->GetArrayLength(mp3buf_);

    int result =lame_encode_buffer(glf, buffer_l, buffer_r, samples, (u_char*)mp3buf, mp3buf_size);

    env->ReleaseShortArrayElements(buffer_l_, buffer_l, 0);
    env->ReleaseShortArrayElements(buffer_r_, buffer_r, 0);
    env->ReleaseByteArrayElements(mp3buf_, mp3buf, 0);

    return result;
}
```

##### flush方法
将MP3结尾信息写入buffer中
```
extern "C"
JNIEXPORT jint JNICALL
Java_zeller_com_mp3recorder_Utils_LameUtils_flush(JNIEnv *env, jclass type, jbyteArray mp3buf_) {
    jbyte *mp3buf = env->GetByteArrayElements(mp3buf_, NULL);

    const jsize  mp3buf_size = env->GetArrayLength(mp3buf_);

    int result = lame_encode_flush(glf, (u_char*)mp3buf, mp3buf_size);

    env->ReleaseByteArrayElements(mp3buf_, mp3buf, 0);

    return result;
}
```

##### close方法
```
extern "C"
JNIEXPORT void JNICALL
Java_zeller_com_mp3recorder_Utils_LameUtils_close(JNIEnv *env, jclass type) {
    lame_close(glf);
    glf = NULL;
}
```
### Java层代码
Jni层的事情到这里就做完了，接下来就交给Java层去做了。
##### 初始化
首先需要对AudioRecord以及Lame进行初始化，初始化需要的参数在前面已经分析过。初始化完之后设置监听，周期性的对数据进行重新编码，编码的操作需要放在一个新的线程中完成。
```
private void initAudioRecorder() throws IOException {
		mBufferSize = AudioRecord.getMinBufferSize(DEFAULT_SAMPLING_RATE,
				DEFAULT_CHANNEL_CONFIG, DEFAULT_AUDIO_FORMAT.getAudioFormat());

		int bytesPerFrame = DEFAULT_AUDIO_FORMAT.getBytesPerFrame();
		/* Get number of samples. Calculate the buffer size
		 * (round up to the factor of given frame size)
		 * 使能被整除，方便下面的周期性通知
		 * */
		int frameSize = mBufferSize / bytesPerFrame;
		if (frameSize % FRAME_COUNT != 0) {
			frameSize += (FRAME_COUNT - frameSize % FRAME_COUNT);
			mBufferSize = frameSize * bytesPerFrame;
		}

		/* Setup audio recorder */
		mAudioRecord = new AudioRecord(DEFAULT_AUDIO_SOURCE,
				DEFAULT_SAMPLING_RATE, DEFAULT_CHANNEL_CONFIG, DEFAULT_AUDIO_FORMAT.getAudioFormat(),
				mBufferSize);

		mPCMBuffer = new short[mBufferSize];
		/*
		 * Initialize lame buffer
		 * mp3 sampling rate is the same as the recorded pcm sampling rate
		 * The bit rate is 32kbps
		 *
		 */
		LameUtil.init(DEFAULT_SAMPLING_RATE, DEFAULT_LAME_IN_CHANNEL, DEFAULT_SAMPLING_RATE, DEFAULT_LAME_MP3_BIT_RATE, DEFAULT_LAME_MP3_QUALITY);
		// Create and run thread used to encode data
		// The thread will
		mEncodeThread = new DataEncodeThread(mRecordFile, mBufferSize);
		mEncodeThread.start();
		mAudioRecord.setRecordPositionUpdateListener(mEncodeThread, mEncodeThread.getHandler());
		mAudioRecord.setPositionNotificationPeriod(FRAME_COUNT);
	}
```
不断的从audioRecord中读取数据，然后交给EncodeThread进行编码。
```
public void start() throws IOException {
		if (mIsRecording) {
			return;
		}
		mIsRecording = true; // 提早，防止init或startRecording被多次调用
	    initAudioRecorder();
		mAudioRecord.startRecording();
		new Thread() {
			@Override
			public void run() {
				//设置线程权限
			android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_URGENT_AUDIO);
				while (mIsRecording) {
					int readSize = mAudioRecord.read(mPCMBuffer, 0, mBufferSize);
					if (readSize > 0) {
						mEncodeThread.addTask(mPCMBuffer, readSize);
					}
				}
				// release and finalize audioRecord
				mAudioRecord.stop();
				mAudioRecord.release();
				mAudioRecord = null;
				// stop the encoding thread and try to wait
				// until the thread finishes its job
				mEncodeThread.sendStopMessage();
			}
		}.start();
	}
```
在DataEncodeThread中把数据转码然后写入文件。
```
private int processData() {
		if (mTasks.size() > 0) {
			Task task = mTasks.remove(0);
			short[] buffer = task.getData();
			int readSize = task.getReadSize();
			int encodedSize = LameUtil.encode(buffer, buffer, readSize, mMp3Buffer);
			if (encodedSize > 0){
				try {
					mFileOutputStream.write(mMp3Buffer, 0, encodedSize);
				} catch (IOException e) {
                    e.printStackTrace();
				}
			}
			return readSize;
		}
		return 0;
}
```
结束录制的时候需要把mp3的结尾信息写入,然后释放资源。
```
if (msg.what == PROCESS_STOP) {
				//处理缓冲区中的数据
				while (encodeThread.processData() > 0);
				// Cancel any event left in the queue
				removeCallbacksAndMessages(null);
				encodeThread.flushAndRelease();
				getLooper().quit();
			}

private void flushAndRelease() {
		//将MP3结尾信息写入buffer中
		final int flushResult = LameUtil.flush(mMp3Buffer);
		if (flushResult > 0) {
			try {
				mFileOutputStream.write(mMp3Buffer, 0, flushResult);
			} catch (IOException e) {
				e.printStackTrace();
			}finally{
				if (mFileOutputStream != null) {
					try {
						mFileOutputStream.close();
					} catch (IOException e) {
						e.printStackTrace();
					}
				}
				LameUtil.close();
			}
		}
	}			
```



### 参考文章
[Android手机直播（三）声音采集](https://www.jianshu.com/p/2cb75a71009f)

[利用Cmake在AndroidStudio来使用lame库](https://www.jianshu.com/p/065bfe6d3ec2#)

[Android NDK 开发之 CMake 必知必会](https://glumes.com/post/android/cmake-best-practices/)

[Android移植lame库(采用CMake)](https://blog.csdn.net/javine/article/details/73277816)
