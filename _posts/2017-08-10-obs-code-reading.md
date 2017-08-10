---
layout: post
title:  "OBS代码解析"
date:   2017-08-10
desc: "OBS代码解析"
keywords: "OBS原理"
categories: [直播]
tags: [直播, OBS]
---

>基于[biliobs](https://github.com/Bilibili/biliobs)源码分析

## 概念

### source

obs采用模块化的思想，将所有的采集数据源都抽象成source，每个source就负责产生各自的数据，不同的数据源（例如摄像头、桌面、文字等）就对应不同的source类型，因此有了像win-capture，win-dshow这些组件，它们就是声明这些source类型。

具体实现的时候对应的是obs_source类，但决定source类型的信息都存储在obs_source_info类中，声明一个新类型的source只需要定义一个obs_source_info变量，然后通过obs_register_source注册即可。在obs_source_info中有个id作为唯一标识，当用户点击添加摄像头等操作时，通过obs_source_create方法传入id，根据id查找到对应的obs_source_info实现，生成一个新的source实例。obs通过一个链表保存所有的source，新建的source插入在链表头部。

### scene

obs在直播时是以scene为基本单位的，可以同时有多个scene，但是用户能看到的只能是其中一个，一个scene上可以添加不同的source，就好比，我们有多个舞台，每个舞台上都有不同的演员在表演，而用户同一时间只能选择其中一个舞台来观看。

scene在实现的时候，实际上就是一种特殊类型的source，在source里面保存了一个obs_scene变量，这个变量存储了一个obs_scene_item类型的链表，在obs_scene_item变量里面存储了具体对应的source。在向scene添加一个source时，会先根据source生成一个obs_scene_item，然后将这个item添加至链表的最后。这样在渲染scene的时候，就可以根据这个链表来渲染出所有的source。

### channel

当有多个scene的时候，怎么实现scene的切换呢，obs的解决方案就是channel，channel实际上就是source，但是代表的是需要渲染的source，然后obs保存了这样一个channel的数组，这个数组就是最后渲染时使用的。这个数组默认最多存储64个channel，每个位置一般存储特定类型的channel，例如0对应scene，3对应麦克风，1对应扬声器，这样切换scene的时候，就只需要将channel 0替换成当前scene，渲染时只需要根据channel数组去渲染即可。

### display

根据channel能够找到所有需要渲染的source，然后要做的就是将这些source画出来，这一步的操作就是display负责的。obs中管理了一个display的链表，也就是说可以同时有多个display对象，每个display都可以有自己的渲染方法。

具体实现对应的类是obs_display，通过obs_display_create可以创建一个新的display，然后通过obs_display_add_draw_callback为这个display添加渲染的回调方法，这些回调方法会在视频渲染线程里被调用，实际上在我们自己的应用中，要将obs这些source画到我们自己的界面上时，就需要写自己的回调函数，在这个回调函数里去渲染，参考BiLiOBSMainWid_preview.cpp里的RenderPreview方法。在渲染时，obs提供了obs_render_main_view方法，调用该方法，即可实现source的渲染，但是调用该方法之前，还需要自行定义各种参数，由于涉及到opengl方面，不是特别熟悉，不再深入。

### output

通过前面三者的组合，能够完成obs的本地渲染，但是当需要把本地的视频流推送到远端RTMP服务器，或者要对本地的视频流做处理的时候，就需要使用到output。这里同source一样，也是采用了模块化的设计，每一种类型的output都是独立的模块，它能够通过回调获取到obs生成的音视频数据，根据这些数据，完成特定功能，例如RTMP传输、flv录制等。

具体实现的时候对应的是obs_output这个类，同source类似，决定output类型的信息都存储在obs_output_info类中。声明一个新类型的output只需要定义一个obs_output_info变量，通过obs_register_output注册即可。在obs_output_info中有个id作为唯一标识，当通过obs_source_create创建output时，会传入id，根据id查找到对应的output实现，生成一个新的output实例。obs通过一个链表保存所有的output，新建的output插入在链表头部。需要注意的是，**在实现自己的output时，并不是默认就能获取到obs生成的音视频数据，需要调用obs_output_begin_data_capture方法，将output注册监听，这样才能够在生成音视频数据时被回调。**

### encoder

output时可以设置音视频的编码器，即encoder，像插件obs-x264就是x264编码的实现，coreaudio-encoder插件就是aac编码的实现。

具体实现的时候对应的是obs_encoder类，决定encoder类型的信息存储在obs_encoder_info类中，两者关系同source一样，注册方法是obs_register_encoder，创建实例方法是obs_encoder_create，通过obs_output_set_video_encoder或者obs_output_set_audio_encoder，可以设置output的音视频编码。

### service

service的设计是为了给RTMP传输时提供相关的设置信息，包括服务器地址、用户名、密码等，从代码来看作用有限，目前也仅有rtmp-services一个插件。

具体实现对应obs_service类，类型信息是obs_service_info类，同source类似，注册方法是obs_register_service，创建实例是obs_service_create，通过obs_output_set_service设置output的service，目前仅限于rtmp-output可以使用。

## 数据结构

### obs_core

全局唯一，存储所有的obs相关数据

* obs_core_video video：视频数据，包括视频渲染和传输线程
* obs_core_audio audio：音频数据，包括音频的线程
* obs_core_data data：存放source和view
* obs_core_hotkeys hotkeys：快捷键相关
* signal_handler_t *signals：处理signal（obs会在各种事件发生时发送signal通知，外部可以通过OBSSignal或者signal_handler_connect去关注相关事件）

### obs_core_video

* pthread_t video_thread：用来渲染界面，并生成视频帧的线程，对应方法入口obs_video_thread
* video_output *video：存储视频输出相关信息，包括缓存的视频帧，以及输出线程

### video_output

* video_output_info info：输出的视频流信息，包括视频格式、高度、宽度、帧率等
* pthread_t thread：负责将生成好的视频帧数据丢给各种服务，output注册的回调就是在这里被调用的，对应方法入口video_thread

### obs_core_audio

* audio_output *audio：存储音频输出相关信息，包括音频的输出线程

### audio_output

* audio_output_info info：输出的音频信息，包括格式、采样率等
* pthread_t thread：负责混音以及将音频数据输出，output注册的回调就是在这里被调用的，对应方法入口audio_thread

### obs_core_data

* DARRAY(struct obs_source*) user_sources：调用obs_add_source时，就会将source保存至数组中，在关闭应用时会保存数组中的source，启动时自动加载
* obs_source *first_source：source链表的头部，保存所有生成的source
* obs_display first_display：display链表的头部，保存所有生成的display
* obs_output *first_output：output链表的头部，保存所有生成的output
* obs_encoder *first_encoder：encoder链表的头部，保存所有生成的encoder
* obs_service *first_service：service链表的头部，保存所有生成的service
* obs_view main_view：存储channel数组

### obs_view

* obs_source *channels[MAX_CHANNELS]：渲染的source列表

### obs_source

* obs_context_data context：存储上下文信息
* obs_source_info info：存储不同类型source的具体信息
* bool ref_by_output_flag：标识是否需要渲染

### obs_output

* obs_context_data context：存储上下文信息
* obs_output_info info：存储不同类型output的具体信息
* obs_encoder *video_encoder：视频编码
* obs_encoder *audio_encoders[MAX_AUDIO_MIXES]：音频编码
* obs_service *service：自定义服务

### obs_encoder

* obs_context_data context：存储上下文信息
* obs_encoder_info info：存储不同类型encoder的具体信息

### obs_service

* obs_context_data context：存储上下文信息
* obs_service_info info：存储不同类型service的具体信息

### obs_display

* DARRAY(struct draw_callback) draw_callbacks：回调函数，自定义实现渲染的入口
* obs_display *next：链表
* obs_display  **prev_next：链表

### obs_context_data

每一个source、output、encoder、service对象里面都有个context变量，存储相关信息，包括名称、设置、自定义数据，还有一个重要的就是链表实现，这里实现链表的时候有个技巧，由于obs中source、output、encoder、service和context的数据结构都是struct类型，并且context变量在source、output、encoder、service中都是第一个成员变量，这样source、output、encoder、service的每一个实例对象，其内存地址与其成员变量context的内存地址是相同的，因此，可以通过强制转换实现source、output、encoder、service对象与context对象的任意转换，因此通过context存储的链表，实际上就是source、output、encoder、service的链表，这就是obs_core_data中只存储first_source、first_output、first_encoder、first_service的原因。通过context实现的链表，都是将最新的插入到链表头部。

* char *name：描述名称
* void *data：可以用来存储不同类型的自定义数据，例如scene类型的obs_source这里存储的就是obs_scene
* obs_context_data *next：链表实现，指向下一个
* obs_context_data **prev_next：链表实现，指向前一个的下一个的地址

### obs_source_info、obs_output_info、obs_encoder_info、obs_service_info

对应前面介绍的概念，用来定义各种类型的source、output、encoder、service

## 方法

### obs_startup

入口，主要完成三个操作
* 初始化数据，各种锁等
* 加载所有的三方插件（像win-capture,obs-filter这种），插件的搜索路径是写死在代码中的
* 注册scene类型的source

### obs_reset_video

* 初始化视频信息，包括视频格式、帧率、分辨率等
* 启动video_output里的video_thread线程，负责将准备好的视频帧信息传给各个output
* 启动obs_core_video里的obs_video_thread线程，负责渲染界面并生成视频帧数据

### obs_reset_audio

* 初始化音频信息，包括采样率、格式等
* 启动audio_output里的audio_thread线程，负责混音然后将数据传给各个output

### obs_load_all_modules

加载所有模块，包括各种类型的source、output等

## 线程

### obs_video_thread

主要负责获取当前scene里的所有source的最近帧，然后将其渲染出来，最后再将合并后的画面帧生成出来，等待video_thread去取。

1. tick_sources：准备需要渲染的source的视频帧
    * obs_mark_output_sources：用来标识所有需要渲染的source，也就是当前scene里包括的source
    * obs_source_video_tick：根据上一个方法打的标识，然后对需要渲染的source调用此方法。这个方法主要有两步，第一步是调用get_closest_frame获取最近的一个视频帧赋值给cur_async_frame，下一步渲染时使用，第二步是去调用自定义obs_source_info的video_tick方法
2. render_displays：根据obs_core_data中的first_display去遍历所有的display，然后调用display里的回调，实现自定义的界面渲染。在自定义的界面渲染中，肯定会用到obs_render_main_view方法来渲染所有的source。
    * obs_render_main_view
        - obs_view_render：遍历channels数组，调用obs_source_video_render方法，该方法会根据obs_source_info定义的类型以及是否有自定义video_render方法，然后调用相应的渲染方法。由于channel 0对应的是scene，所以scene里面的各个source的渲染都是由scene类型的source中自定义的渲染函数scene_video_render来负责的。
        - scene_video_render：根据obs_scene中的obs_scene_item链表，遍历所有的item，然后根据item得到对应的source，最后同样是调用obs_source_video_render方法实现渲染
3. output_frame
    * render_video：涉及opengl，不太懂干啥
    * download_frame：涉及opengl，根据名字推测是生成视频帧
    * output_video_data
        - video_output_lock_frame：向video_output里的cache数组里添加一个视频帧，但是这时候这个视频帧是空的
        - 将download_frame生成的frame复制到上一步的空视频帧里，但是根据原frame格式，决定是否需要转换格式，目标格式是rgb
        - video_output_unlock_frame：通过信号量update_semaphore，告诉video_thread已经有一个新的视频帧生成

### video_thread

主要负责将生成的视频帧传给各个output，然后各自处理，主要就是调用video_output_cur_frame方法。在该方法里，首先从video_output里的cache数组取最近生成的帧，然后遍历video_output里的video_input数组，调用回调，通知各个output。这个video_input数组，实际上在obs_output里面的obs_output_begin_data_capture方法中就会去注册回调方法。最后通过层层的方法传递，就会去调用obs_output_info中的raw_video或者encoded_packet方法，如果obs_output_info中的flag声明没有OBS_OUTPUT_ENCODED，那么就是前者，反之则后者。例如，如果创建了一个obs-outputs模块里的rtmp_output，那么此时rtmp-output里面的rtmp_stream_data（obs_output_info中的encoded_packet）方法就会被调用，然后就会将这个视频帧数据塞入队列，等待传输。

### audio_thread

负责音频的混音以及输出，其原理基本上同video_thread类似。
