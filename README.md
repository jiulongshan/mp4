# MP4 Demuxer 

[![CMake](https://github.com/akanchi/mp4/actions/workflows/cmake.yml/badge.svg?branch=main)](https://github.com/akanchi/mp4/actions/workflows/cmake.yml)

A simple implementation of mp4 demuxing, supports AVC/HEVC/VP9/AAC/mp3/Opus.

### Usage

Build Demuxer by:

```shell
mkdir cmake-build-debug && cd cmake-build-debug && cmake .. && make
# or
sh build.sh
```

Run Demuxer by:

```shell
./mp4_demuxer ../docs/SampleVideo_360x240_5mb-base.mp4
```

After run demuxer, here is the box tree:

```
[0, 4298693)
    ftyp[0, 32)
        major_brand: isom
        minor_version: 512
        compatible_brands: isomiso2avc1mp41
    free[32, 40)
    mdat[40, 4270608)
    moov[4270608, 4298693)
        mvhd[4270616, 4270724)
            version: 0
            flags: 0
            creation_time: 0
            modification_time: 0
            time_scale: 1000
            duration: 64896
            preferred_rate: 1.000000
            preferred_volume: 1.000000
            matrix: 
                | a, b, u |   | 1.000000, 0.000000, 0.000000 |
                | c, d, v | = | 0.000000, 1.000000, 0.000000 |
                | x, y, w |   | 0.000000, 0.000000, 1.000000 |
            next_track_id: 3
        trak[4270724, 4279127)
            tkhd[4270732, 4270824)
            edts[4270824, 4270860)
                elst[4270832, 4270860)
            mdia[4270860, 4279127)
                mdhd[4270868, 4270900)
                    version: 0
                    flags: 0
                    creation_time: 0
                    modification_time: 0
                    time_scale: 15360
                    duration: 996352
                    language: 21956
                    quality: 0
                hdlr[4270900, 4270945)
                minf[4270945, 4279127)
                    vmhd[4270953, 4270973)
                    dinf[4270973, 4271009)
                        dref[4270981, 4271009)
                    stbl[4271009, 4279127)
                        stsd[4271017, 4271187)
                            avc1[4271033, 4271187)
                                avcC[4271119, 4271171)
                        stts[4271187, 4271211)
                        stss[4271211, 4271279)
                        stsc[4271279, 4271307)
                        stsz[4271307, 4275219)
                        stco[4275219, 4279127)
        trak[4279127, 4298595)
            tkhd[4279135, 4279227)
            edts[4279227, 4279263)
                elst[4279235, 4279263)
            mdia[4279263, 4298595)
                mdhd[4279271, 4279303)
                    version: 0
                    flags: 0
                    creation_time: 0
                    modification_time: 0
                    time_scale: 48000
                    duration: 3115008
                    language: 21956
                    quality: 0
                hdlr[4279303, 4279348)
                minf[4279348, 4298595)
                    smhd[4279356, 4279372)
                    dinf[4279372, 4279408)
                        dref[4279380, 4279408)
                    stbl[4279408, 4298595)
                        stsd[4279416, 4279519)
                            mp4a[4279432, 4279519)
                                esds[4279468, 4279519) codec.0x40
                        stts[4279519, 4279543)
                        stsc[4279543, 4282499)
                        stsz[4282499, 4294687)
                        stco[4294687, 4298595)
        udta[4298595, 4298693)
            meta[4298603, 4298693)
                hdlr[4298615, 4298648)
                ilst[4298648, 4298693)
```

Play the extracted file by:

```shell
ffplay aac_audio.aac
```

### Key points

我的目的是将`mp4`文件中的音视频数据解析出来，并保存为对应媒体文件。

众所周知，`mp4`的媒体数据是保存在`mdat`中，但 `mdat`没有告诉我们哪些是音频/视频数据，所以需要依赖其它`box`（主要是`stco`、`stsz`、`stsc`）去定位`mdat`中的音视频数据。

#### stco(Chunk Offset Box)

用于定位媒体数据中的块偏移，因为有多个块，所以是一个数组格式，chunk_offsets[number_of_entries]

#### stsz(Sample Size Box)

用于表示每个媒体`sample`的的大小，通常`sample`的大小是可变的，sample_sizes[number_of_entries]保存所有sample的size

#### stsc(Sample-to-Chunk Box)

如果只知道`stco` 和`stsz`是无法定位媒体数据的，所以需要`sample`和`chunk`的对应表，才能解析出媒体数据

以下表格来自于`docs/qtff.pdf`

**Figure 2-47**    An example of a sample-to-chunk table

| First chunk | Samples per chunk | Sample description ID |
| :---------: | :---------------: | :-------------------: |
|      1      |         3         |           1           |
|      3      |         1         |           1           |
|      5      |         1         |           1           |

下面是展开之后的表格，可以看出可以通过`stco`去索引`chunk`，以及通过`stsz`去索引`sample`。

|       |     First chunk      |   Samples per chunk   | Sample description ID |
| :---: | :------------------: | :-------------------: | :-------------------: |
|       |          1           |           3           |           1           |
| ***** |          2           |           3           |           1           |
|       |          3           |           1           |           1           |
| ***** |          4           |           1           |           1           |
|       |          5           |           1           |           1           |
|       | Number of chunks = 5 | Number of samples = 9 |                       |

#### 提取媒体数据

知道了所需要的信息就可以着手提取媒体数据了，相关代码可以参考`int TrackContext::extract()`

```c++
uint32_t current_entry_index = 0;// stsc中，entries的下标
uint32_t current_entry_sample_index = 0; // stsc中，对应current_entry_index的entry.sample的下标
uint32_t chunk_logic_index = 0;	// stco中，chunk_offsets的下标
uint32_t sample_offset = 0;// sample的偏移量
for (int i = 0; i < sample_sizes.count; i++) {
  // 1. 取出当前entry = stsc->entries[current_entry_index]
  // 2. 计算当前sample的起始位置pos = chunk_offsets[chunk_logic_index] + sample_offset
  // 3. 起始位置pos 和 sample_sizes[i]可以获取当前的sample数据
  // 4. current_entry_sample_index++
  // 5. 计算sample_offset += sample_sizes[i]
  // 6. 如果current_entry_sample_index >= entry.samples_per_chunk 则切换到下一个chunk，并重置sample相关标记
}
```

提取`avc`的时候，一个`sample`可能含有多个`nalu`，格式是``` [nalu length(4bytes)][nalu data(nalu length bytes)]+[nalu length][nalu data]+...```。这个规则同样适用于`hevc`的提取。

#### 关于Opus

按照正常的音频提取的逻辑，提取出来的是`Opus`裸流，播放器无法播放

看了一下`FFmpeg`的处理，需要封装成`Ogg`格式，同时查阅了`Ogg`和`Opus`的协议文档最终封装成可播放的音频文件

`Ogg`封装主要有几部分

* `Ogg`是以`page`为单位封装数据的，`page`以`OggS`开头
* `OpusHead`，这部分可以从`dOps`中读取，但需要手动添加`OpusHead`
* `CommentHeader`，以`OpusTags`开头，主要放一些无关紧要的信息
* 音频数据的封装，如果`sample size > 255 bytes`，则需要拆分成多个`page`，因为记录`page payload`大小的字段只有`1个字节`，不然播放器会报`CRC mismatch!`
* 最后一个`page`的页码和前一个`page`的页码是一样的

