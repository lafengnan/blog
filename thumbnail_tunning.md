

# 缩略图优化

## 0x1 城市里来了新长官

这里要讲的故事说起来真是历史有悠久了，那时候$$HP \neq HPE + HPI$$，还是曾经的硅谷明星Hewleet-Packard，现在一切都变了，老家伙们（IBM、HP）被这个时代狠狠的抛弃了，**迈克尔-记录哪里都有我-乔丹**曾经对魔术师约翰逊和大鸟伯德喷了一句著名的垃圾话：”城市里来了新长官！“，恩，时代变了，大家都得接受现实。

其实，并不是这个世界变化太快，而是老家伙们的动作实在太慢，慢的老	天都看不下去了，只好迫使他们在来不及思考的情况下去做一些仓促的选择。公司如此，我们这些80后老家伙似乎更甚，现在都日渐苍老，各种新事物应接不暇，似乎你现在不会AI(吹)、Cloud(牛)和Bigdata（逼）就是个技术怪胎，除了害人没有任何价值。



## 0x2 背景

在HP时候做的项目[HP Connected Drive(HPCD)](https://support.hp.com/in-en/document/c04016159)是一个类似于Dropbox的服务，可以让用户在PC、平板以及手机上共享图片、文档和视频等文件。这个项目是HP当年打造PC生态的一个尝试，不过随着windows 10的出现，这个项目也就不得不退出历史舞台了。HPCD的后台部署在当时HP仍然大力推广的[HP Helion](https://www.hpe.com/uk/en/software/openstack-cloud-iaas.html)公有云上，其核心存储正是Openstack平台中的[Swift](https://wiki.openstack.org/wiki/Swift)对象存储系统，几乎所有的文件操作都是通过Swift存储完成的。虽然Swift帮助系统解决了可用性和分布式问题，但是这个世界上所有的硬币都有正反面，很显然Swift也不能幸免，在带来好处的同时也带着弊端。由于Swift自身的*account*，*container*和*object*三层逻辑结构以及网络带宽等等各种因素造成用户在上传文件之后设备上同步显示缩略图（thumbnail）的性能太差，在中国团队慢慢接手主要功能开发之后我们亟需通过优化这个弊端来让US的同事们在开会的时候少叨叨叨。



## 0x3包袱

下面这张图是原始的缩略图处理的简化架构，主要由三部分组成：

1. 文件处理（上传、下载）
2. 缩略图处理（上传、下载、裁剪）
3. 元数据管理

![thumb_old_arch](/resources/thumb_old_arch.png)

这个架构看上去貌似还凑合，但是在出现大量文件上传时会出现严重的性能问题----缩略图处理中的上传速度太慢，导致并发进行的元数据上传成功后却无缩略图可以下载到客户端展示。文件上传的处理流程由下列几个步骤组成：

1. 客户端上传原始缩略图，服务端将其暂存于Swift中
2. 服务端从Swift中删除旧的缩略图
3. 服务端处理原始缩略图
   1. 从Swift下载原始缩略图
   2. 解密原始缩略图
   3. 根据缩略图配置，裁剪生成不同分辨率的新缩略图文件
4. 将所有新生成的缩略图文件上传到Swift中
5. 更新元数据中的”has_thumbnail"字段
6. 从Swift中删除原始缩略图

其实系统在设计之初还是考虑到了缩略图处理的负载，因此将原始缩略图的生成工作放在了客户端进行，但是然并卵，这6个步骤有4步是和Swift通信（第2步、第3步、第5步和第6步），严重拉低了系统性能，导致平均每个缩略图的处理时间达到了1000ms左右。下面的表格是在Dev Int和QA环境中实测出来的数据：

![thumb_old_performance](/resources/thumb_old_performance.png)

根据表格中的数据我们可以看出缩略图的处理过程中存在以下诸多问题：

1. 网络交互过多
   - 4次Redis操作（未画出）
   - 至少8次Cassandra操作
   - 至少8次Swift操作
   - 4次Elasticsearch操作
2. 缩略图裁剪操作非常耗时
3. 重复的Swift操作
   - 原始缩略图先暂存到Swift中，然后下载下来进行裁剪，最后再删除Swift中暂存的原始缩略图
4. 依赖Celery队列操作
5. 代码中的细节问题

从这些问题我们可以看出，原始缩略图的暂存以及裁剪操作是最耗时的两步，那么如何优化也就有了思路。由于缩略图裁剪采用的是libmagic处理库，一时难以找到更优的图形库，因此第一阶段就把优化的重心放在了原始缩略图的暂存步骤中。

## 0x4军刀

为了解决原始缩略图的暂存问题，我在设计方案时假借了Linux文件系统中的VFS概念，引入了一个抽象的*Staging Storage Layer*(SSL)层以及抽象的持久化层，用这个抽象层封装所有的IO操作，对API层屏蔽IO细节，并将优化的工作放置在SSL中完成。最初设计时考虑了下面三种需求：

1. 将文件存储在本地文件系统中
2. 将文件存储在分布式文件系统中
3. 将文件存储在分布式缓存中

按照这个设计思路，新的架构就变成下面这个模样了

![thumb_new_arch](/resources/thumb_new_arch.png)

这个架构中最主要引入了Redis作为原始缩略图的暂存场所，也正是这一改动直接将原始缩略图的上传操作从200ms左右降低到5ms左右，一举干掉了第一个瓶颈。新的暂存层提供了以下几个接口：

1. 标准的write接口，用于上传文件
2. 标准的read接口，用于下载文件
3. 标准的delete接口，用于删除暂存文件
4. 内部failvoer接口，用于处理失效情况

持久化层则提供下列接口：

1. 标准的write接口，用于上传文件
2. 标准的metada接口用于上传和下载元数据

经过这个改动，原来的thumbnail处理流程就变成了下面的形式：

1. 通过SSL层的write接口暂存原始缩略图
2. 通过SSL层的read接口下载原始缩略图
3. 使用Wand库裁剪缩略图
4. 通过持久化层的write接口上传裁剪后的缩略图
5. 通过持SSL层的delete接口删除裁剪完成的原始缩略图

## 0x5小刀

新的设计方案中引入了Redis作为原始缩略图的暂存空间，由于Redis本身的持久化特征（一旦Redis服务器崩溃，极端情况下可能会出现1s的数据丢失），特别引入了可信队列作为Redis中的数据存储结构。可信队列由两个不同的Redis列表存储key值，一个Hash set存储键-值数据，即：

* 临时队列（pending queue）
* 工作队列（working queue）
* 散列集（hash set）

因为在HPCD中采用*userId*标识用户信息，*uuid*来标识缩略图信息，因此可信队列的数据接口看起来和下图差不多：

![thumb_reliable_queue](/resources/thumb_reliable_queue.png)

原始缩略图首先存储在临时队列中，当Celery任务开始执行裁剪操作时，将缩略图出队的同时，拷贝副本存储在工作队列中（该操作采用Redis[RPOPLPUSH](https://redis.io/commands/rpoplpush)命令完成，是一个原子操作），在裁剪操作完成之后再清理工作队列中的数据。



## 0x6 提升

通过优化，原始缩略图的上传时间缩短到了15毫秒左右，提升幅度达到90%以上。下面的数据是在我自己的开发机上测试的结果：

[2015-05-05 23:39:38,465] Using <class '__main__.PolarisRedis'> to store thumbnails
[2015-05-05 23:39:38,465] uploading IMGP0652.jpg...
[2015-05-05 23:39:38,504] write spending 0.0156149864197 seconds
[2015-05-05 23:39:38,505] upload spending 0.0401799678802 seconds
[2015-05-05 23:39:38,505] Key:chris:34de1b7a3ea84dd7bb942560703f3b4e
[2015-05-05 23:40:26,486] Using <class '__main__.PolarisRedis'> to store thumbnails
[2015-05-05 23:40:26,486] uploading IMGP0652.jpg...
[2015-05-05 23:40:26,507] get_file_content spending 0.0208430290222 seconds
[2015-05-05 23:40:26,527] write spending 0.0193901062012 seconds
[2015-05-05 23:40:26,528] upload spending 0.0417990684509 seconds
[2015-05-05 23:40:26,528] Key:chris:fe6fed5c582c414eb6193c0ed34e7959
[2015-05-05 23:41:04,754] Using <class '__main__.PolarisFile'> to store thumbnails
[2015-05-05 23:41:04,754] uploading IMGP0652.jpg...
[2015-05-05 23:41:04,776] get_file_content spending 0.0215599536896 seconds
[2015-05-05 23:41:04,809] write spending 0.0328578948975 seconds
[2015-05-05 23:41:04,810] upload spending 0.0559079647064 seconds
[2015-05-05 23:41:04,810] Key:chris:e3c989a5976a4673a6783fb9f470a8c8
[2015-05-05 23:51:52,782] uploading IMGP0652.jpg...
[2015-05-05 23:51:52,810] function:get_file_content spending 0.0265839099884 seconds
[2015-05-05 23:51:52,826] function:_write spending 0.0151500701904 seconds
[2015-05-05 23:51:52,827] function:upload spending 0.0450479984283 seconds
[2015-05-05 23:51:52,827] Key:None
[2015-05-05 23:53:12,200] Using <class '__main__.PolarisFile'> to store thumbnails
[2015-05-05 23:53:12,200] uploading IMGP0652.jpg...
[2015-05-05 23:53:12,227] function:get_file_content spending 0.0260078907013 seconds
[2015-05-05 23:53:12,254] function:write spending 0.0273790359497 seconds
[2015-05-05 23:53:12,255] function:upload spending 0.0552320480347 seconds
[2015-05-05 23:53:12,256] Key:None
[2015-05-05 23:55:35,736] Using <class '__main__.PolarisFile'> to store thumbnails
[2015-05-05 23:55:35,736] uploading IMGP0652.jpg...
[2015-05-05 23:55:35,757] function:get_file_content spending 0.0206401348114 seconds
[2015-05-05 23:55:35,812] function:write spending 0.0546050071716 seconds
[2015-05-05 23:55:35,814] function:upload spending 0.0772271156311 seconds
[2015-05-05 23:55:35,814] Key:anan:249dfb5326854469bd7cf153c4eeca4d
[2015-05-05 23:55:39,462] uploading IMGP0652.jpg...
[2015-05-05 23:55:39,489] function:get_file_content spending 0.0265939235687 seconds
[2015-05-05 23:55:39,504] function:write spending 0.0147728919983 seconds
[2015-05-05 23:55:39,505] function:upload spending 0.0429019927979 seconds
[2015-05-05 23:55:39,505] Key:anan:6bb8f035e47b4f129735d7b638fc2aee
[2015-05-05 23:57:05,820] uploading IMGP0652.jpg...
[2015-05-05 23:57:05,846] function:get_file_content spending 0.026064157486 seconds
[2015-05-05 23:57:05,863] function:write spending 0.0166430473328 seconds
[2015-05-05 23:57:05,865] function:upload spending 0.0448157787323 seconds
[2015-05-05 23:57:05,865] Key:anan:86bf1cad436d453886faa77b2a6f7e8c
[2015-05-05 23:57:33,950] uploading IMGP0652.jpg...
[2015-05-05 23:57:33,972] function:get_file_content spending 0.0211451053619 seconds
[2015-05-05 23:58:36,277] uploading IMGP0652.jpg...
[2015-05-05 23:58:36,304] function:get_file_content spending 0.0261211395264 seconds
[2015-05-05 23:58:36,320] function:write spending 0.0157389640808 seconds
[2015-05-05 23:58:36,321] function:upload spending 0.0440239906311 seconds
[2015-05-05 23:58:36,321] Key:anan:effe2faa74514253b426eefe276b6cd0
[2015-05-06 00:04:16,660] Using <class '__main__.PolarisRedis'> to store thumbnails
[2015-05-06 00:04:16,660] uploading IMGP0652.jpg...
[2015-05-06 00:04:16,681] function:get_file_content spending 0.0207841396332 seconds
[2015-05-06 00:04:16,696] function:write spending 0.0142199993134 seconds
[2015-05-06 00:04:16,697] function:upload spending 0.0369110107422 seconds
[2015-05-06 00:04:16,697] Key:anan:1e4a71e7cada4ddea055db1ba74b1297
[2015-05-06 00:05:00,155] Using <class '__main__.PolarisRedis'> to store thumbnails
[2015-05-06 00:05:00,155] uploading IMGP0652.jpg...
[2015-05-06 00:05:00,177] function:get_file_content spending 0.0208959579468 seconds
[2015-05-06 00:05:00,193] function:write spending 0.0166900157928 seconds
[2015-05-06 00:05:00,194] function:upload spending 0.0392451286316 seconds
[2015-05-06 00:05:00,195] Key:anan:e3bcf794d91b4944b504b4f26bba7a33
[2015-05-06 00:05:44,827] Using PolarisRedis to store thumbnails
[2015-05-06 00:05:44,827] uploading IMGP0652.jpg...
[2015-05-06 00:05:44,850] function:get_file_content spending 0.021879196167 seconds
[2015-05-06 00:05:44,864] function:write spending 0.0143730640411 seconds
[2015-05-06 00:05:44,866] function:upload spending 0.0382318496704 seconds
[2015-05-06 00:05:44,866] Key:anan:bac764469ca24c698330aff6d4e49a91
[2015-05-06 00:06:50,793] Using PolarisRedis to store thumbnails
[2015-05-06 00:06:50,793] uploading IMGP0652.jpg...
[2015-05-06 00:06:50,814] function:get_file_content spending 0.0212450027466 seconds
[2015-05-06 00:06:50,831] function:write spending 0.0165159702301 seconds
[2015-05-06 00:06:50,832] function:upload spending 0.0392718315125 seconds
[2015-05-06 00:06:50,832] Key:{}
[2015-05-06 00:07:10,811] Using PolarisRedis to store thumbnails
[2015-05-06 00:07:10,812] uploading IMGP0652.jpg...
[2015-05-06 00:07:10,838] function:get_file_content spending 0.0264611244202 seconds
[2015-05-06 00:07:10,853] function:write spending 0.0143811702728 seconds
[2015-05-06 00:07:10,854] function:upload spending 0.0427858829498 seconds
[2015-05-06 00:07:38,724] Using PolarisRedis to store thumbnails
[2015-05-06 00:07:38,724] uploading IMGP0652.jpg...
[2015-05-06 00:07:38,745] function:get_file_content spending 0.0208978652954 seconds
[2015-05-06 00:07:38,760] function:write spending 0.0144100189209 seconds
[2015-05-06 00:07:38,761] function:upload spending 0.0368371009827 seconds
[2015-05-06 00:07:38,761] Key:anan:2b9e4dc641e04df280bc5b176f57a455
[2015-05-06 00:08:11,101] Using PolarisFile to store thumbnails
[2015-05-06 00:08:11,102] downloading from None to None
[2015-05-06 00:08:19,937] Using PolarisFile to store thumbnails
[2015-05-06 00:08:19,938] downloading from None to .
[2015-05-06 00:08:31,183] Using PolarisFile to store thumbnails
[2015-05-06 00:08:31,183] downloading from None to tmp
[2015-05-06 00:09:13,764] Using PolarisRedis to store thumbnails
[2015-05-06 00:09:13,764] downloading from Redis to tmp
[2015-05-06 00:11:07,482] Using PolarisRedis to store thumbnails
[2015-05-06 00:11:07,482] downloading from Redis to tmp
[2015-05-06 00:11:23,042] Using PolarisRedis to store thumbnails
[2015-05-06 00:11:23,042] uploading IMGP0652.jpg...
[2015-05-06 00:11:23,069] function:get_file_content spending 0.0265588760376 seconds
[2015-05-06 00:11:23,085] function:write spending 0.0153830051422 seconds
[2015-05-06 00:11:23,086] function:upload spending 0.0436038970947 seconds
[2015-05-06 00:11:23,086] Key:anan:0d1a92981edb4c9a989beb58f4f9a6f2
[2015-05-06 00:11:38,783] Using PolarisRedis to store thumbnails
[2015-05-06 00:11:38,783] downloading from Redis to tmp
[2015-05-06 00:11:38,837] function:read spending 0.0534811019897 seconds
[2015-05-06 00:11:38,838] function:download spending 0.0540020465851 seconds
[2015-05-06 00:17:46,540] Using PolarisRedis to store thumbnails
[2015-05-06 00:17:46,540] downloading from Redis to tmp
[2015-05-06 00:17:55,593] Using PolarisRedis to store thumbnails
[2015-05-06 00:17:55,593] uploading IMGP0652.jpg...
[2015-05-06 00:17:55,615] function:get_file_content spending 0.0211629867554 seconds
[2015-05-06 00:17:55,631] function:write spending 0.0164349079132 seconds
[2015-05-06 00:17:55,633] function:upload spending 0.0395541191101 seconds
[2015-05-06 00:17:55,633] Key:anan:0a17587c0c5b47f3b5bc2dbd422c9f8a
[2015-05-06 00:18:25,037] Using PolarisRedis to store thumbnails
[2015-05-06 00:18:25,037] uploading IMGP0652.jpg...
[2015-05-06 00:18:25,059] function:get_file_content spending 0.0210130214691 seconds
[2015-05-06 00:18:25,074] function:write spending 0.0146110057831 seconds
[2015-05-06 00:18:25,074] function:upload spending 0.0371329784393 seconds
[2015-05-06 00:18:25,075] Key:anan:fb865e15c86249e1ad57db47e899eba2
[2015-05-06 00:22:59,869] Using PolarisRedis to store thumbnails
[2015-05-06 00:22:59,869] uploading IMGP0652.jpg with args IMGP0652.jpg, /tmp, True, {'logger': <logging.Logger object at 0x10c9c22d0>, 'userid': 'anan'}...
[2015-05-06 00:22:59,890] function:get_file_content spending 0.020828962326 seconds
[2015-05-06 00:22:59,905] function:write spending 0.0145888328552 seconds
[2015-05-06 00:22:59,906] function:upload spending 0.0372869968414 seconds
[2015-05-06 00:22:59,906] Key:anan:f84b568262e04555a9797d81089a1731
[2015-05-06 00:25:12,325] Using PolarisRedis to store thumbnails
[2015-05-06 00:25:12,325] uploading IMGP0652.jpg to /tmp. redis?True, args:{'logger': <logging.Logger object at 0x102e1b2d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:25:12,346] function:get_file_content spending 0.0207631587982 seconds
[2015-05-06 00:25:12,365] function:write spending 0.0186431407928 seconds
[2015-05-06 00:25:12,367] function:upload spending 0.041366815567 seconds
[2015-05-06 00:25:12,367] Key:anan:aef001a786694eba8462878518724092
[2015-05-06 00:30:32,157] Using PolarisRedis to store thumbnails
[2015-05-06 00:30:32,158] uploading IMGP0652.jpg to 0. redis:True, args:{'logger': <logging.Logger object at 0x107c572d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:30:32,179] function:get_file_content spending 0.0208029747009 seconds
[2015-05-06 00:30:32,196] function:write spending 0.0170381069183 seconds
[2015-05-06 00:30:32,197] function:upload spending 0.0394489765167 seconds
[2015-05-06 00:30:32,197] Key:anan:39721c33c9404b6baeaf5efd9b9132ef
[2015-05-06 00:30:56,814] Using PolarisRedis to store thumbnails
[2015-05-06 00:31:16,032] Using PolarisRedis to store thumbnails
[2015-05-06 00:31:16,032] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x10ae4c2d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:31:16,053] function:get_file_content spending 0.0209369659424 seconds
[2015-05-06 00:31:16,069] function:write spending 0.0150489807129 seconds
[2015-05-06 00:31:16,070] function:upload spending 0.0377290248871 seconds
[2015-05-06 00:31:16,070] Key:anan:3b4fae77cf6d47ad96e078ed022b8989
[2015-05-06 00:31:25,580] Using PolarisRedis to store thumbnails
[2015-05-06 00:31:25,580] uploading IMGP0652.jpg to db1. redis:True, args:{'logger': <logging.Logger object at 0x10c8002d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:31:25,602] function:get_file_content spending 0.0215101242065 seconds
[2015-05-06 00:31:25,619] function:write spending 0.0174510478973 seconds
[2015-05-06 00:31:25,621] function:upload spending 0.0408990383148 seconds
[2015-05-06 00:31:25,621] Key:anan:cc709f95f4e34f4d8ffa0c93fa4f0b5e
[2015-05-06 00:32:20,658] Using PolarisRedis to store thumbnails
[2015-05-06 00:32:20,658] uploading IMGP0652.jpg to db1. redis:True, args:{'logger': <logging.Logger object at 0x10350c2d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:32:20,679] function:get_file_content spending 0.0207669734955 seconds
[2015-05-06 00:32:20,694] function:write spending 0.013946056366 seconds
[2015-05-06 00:32:20,695] function:upload spending 0.0366480350494 seconds
[2015-05-06 00:32:20,695] Key:anan:a0d4f4949aa944dbac19d75f4b6c3f6c
[2015-05-06 00:32:50,948] Using PolarisRedis to store thumbnails
[2015-05-06 00:32:50,949] uploading IMGP0652.jpg to db1. redis:True, args:{'logger': <logging.Logger object at 0x10e3bf2d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:32:50,970] function:get_file_content spending 0.0209951400757 seconds
[2015-05-06 00:32:50,986] function:write spending 0.0154259204865 seconds
[2015-05-06 00:32:50,987] function:upload spending 0.0384161472321 seconds
[2015-05-06 00:32:50,987] Key:anan:949df1e7f135473bb21062a5932afb86
[2015-05-06 00:34:58,861] Using PolarisRedis to store thumbnails
[2015-05-06 00:34:58,862] uploading IMGP0652.jpg to db1. redis:True, args:{'logger': <logging.Logger object at 0x10db232d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:34:58,883] function:get_file_content spending 0.0211579799652 seconds
[2015-05-06 00:34:58,899] function:write spending 0.015823841095 seconds
[2015-05-06 00:34:58,900] function:upload spending 0.0388250350952 seconds
[2015-05-06 00:34:58,901] Key:anan:487086d6371b411e9fcc881c7cedb785
[2015-05-06 00:35:12,257] Using PolarisRedis to store thumbnails
[2015-05-06 00:35:12,257] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x10e2ff2d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 00:35:12,278] function:get_file_content spending 0.0206711292267 seconds
[2015-05-06 00:35:12,293] function:write spending 0.0149781703949 seconds
[2015-05-06 00:35:12,294] function:upload spending 0.0372240543365 seconds
[2015-05-06 00:35:12,295] Key:anan:25f2a811dfb04ea89e989703b898bf19
[2015-05-06 21:03:06,939] Using PolarisRedis to store thumbnails
[2015-05-06 21:03:06,966] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x10def72d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 21:03:07,146] function:get_file_content spending 0.17821097374 seconds
[2015-05-06 21:03:07,166] function:write spending 0.0193998813629 seconds
[2015-05-06 21:03:07,167] function:upload spending 0.200500965118 seconds
[2015-05-06 21:03:07,167] Key:anan:7825778c807d4d26a7972a4b82602e1b
[2015-05-06 23:26:11,115] Using PolarisRedis to store thumbnails
[2015-05-06 23:26:11,133] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x10e9cb2d0>, 'userid': 'anan', 'ttl': 100}...
[2015-05-06 23:26:11,346] function:get_file_content spending 0.211020946503 seconds
[2015-05-06 23:26:11,365] function:write spending 0.0192952156067 seconds
[2015-05-06 23:26:11,366] function:upload spending 0.233472824097 seconds
[2015-05-06 23:26:11,367] Key:anan:8f3ebf01b68140d7941378e31fc34e1e
[2015-05-06 23:37:28,935] Using PolarisRedis to store thumbnails
[2015-05-06 23:37:28,935] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x10c1e02d0>, 'userid': 'anan', 'ttl': 600}...
[2015-05-06 23:37:28,959] function:get_file_content spending 0.0231368541718 seconds
[2015-05-06 23:37:28,973] function:write spending 0.0140860080719 seconds
[2015-05-06 23:37:28,975] function:upload spending 0.0395538806915 seconds
[2015-05-06 23:37:28,975] Key:anan:cb5c7cea8faa4e77977b456762f5d037
[2015-05-06 23:50:06,202] Using PolarisRedis to store thumbnails
[2015-05-06 23:50:06,222] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x11056b2d0>, 'userid': 'anan', 'ttl': 600}...
[2015-05-06 23:50:06,250] function:get_file_content spending 0.0266029834747 seconds
[2015-05-06 23:50:06,265] function:write spending 0.0144679546356 seconds
[2015-05-06 23:50:06,266] function:upload spending 0.0442810058594 seconds
[2015-05-06 23:50:06,266] Key:anan:7fc242a24a524ac6b804a4905ac2c16b
[2015-05-07 00:06:14,207] Using PolarisRedis to store thumbnails
[2015-05-07 00:06:14,207] uploading IMGP0652.jpg to db0. redis:True, args:{'logger': <logging.Logger object at 0x107f462d0>, 'userid': 'anan', 'ttl': 600}...
[2015-05-07 00:06:14,229] function:get_file_content spending 0.0214579105377 seconds
[2015-05-07 00:06:14,244] function:write spending 0.0148680210114 seconds
[2015-05-07 00:06:14,245] function:upload spending 0.0384109020233 seconds
[2015-05-07 00:06:14,246] Key:anan:cbc26457a40d4a3baf4f217b3c1f3f5b
[2015-05-07 00:11:53,371] Using PolarisFile to store thumbnails
[2015-05-07 00:11:53,371] downloading from None to None
[2015-05-07 00:12:04,391] Using PolarisRedis to store thumbnails
[2015-05-07 00:12:04,391] downloading from Redis to None
[2015-05-07 00:12:52,088] Using PolarisRedis to store thumbnails
[2015-05-07 00:12:52,089] downloading from Redis to .
[2015-05-07 00:12:52,137] function:read spending 0.047700881958 seconds
[2015-05-07 00:12:52,137] function:download spending 0.0482230186462 seconds
[2015-05-07 00:13:27,252] Using PolarisRedis to store thumbnails
[2015-05-07 00:13:27,253] downloading from Redis to .
[2015-05-07 00:13:27,307] function:read spending 0.0540471076965 seconds
[2015-05-07 00:13:27,307] function:download spending 0.054582118988 seconds
[2015-05-07 00:13:49,778] Using PolarisFile to store thumbnails
[2015-05-07 00:13:49,778] downloading from None to .
[2015-05-07 00:13:56,319] Using PolarisRedis to store thumbnails
[2015-05-07 00:13:56,319] downloading from Redis to .
[2015-05-07 00:13:56,369] function:read spending 0.0497679710388 seconds
[2015-05-07 00:13:56,369] function:download spending 0.0503540039062 seconds
[2015-05-07 00:15:35,176] Using PolarisRedis to store thumbnails
[2015-05-07 00:15:35,176] downloading from Redis to .
[2015-05-07 00:15:35,228] function:read spending 0.0511820316315 seconds
[2015-05-07 00:15:35,228] function:download spending 0.0518951416016 seconds