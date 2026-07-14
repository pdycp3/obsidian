# RK3588 MPP 硬解码开发实战指南

> 针对 Tronlong RK3588 + ffmpeg-rockchip 环境，重点解决码流波动下 rkvdec core resting、DMA-buf 泄漏、重启无法恢复等核心问题。

---

## 目录

1. [架构总览：ffmpeg-rockchip 与 MPP 的关系](#1-架构总览)
2. [rkmppdec.c 关键源码解析（你用的就是这层）](#2-rkmppdec源码解析)
3. [Buffer 模式选择与 DMA-buf 生命周期管理](#3-buffer模式与dma-buf管理)
4. [码流波动下的解码器健康检测与 Reset 机制](#4-健康检测与reset机制)
5. [DMA-buf 泄漏根因分析与修复](#5-dma-buf泄漏修复)
6. [完整封装：健壮的 RKMppDecoder 类](#6-完整封装)
7. [调试与排查工具箱](#7-调试排查)

---

## 1. 架构总览

```
┌─────────────────────────────────────────────────────┐
│                    你的应用层                        │
│         avcodec_send_packet / avcodec_receive_frame  │
├─────────────────────────────────────────────────────┤
│              ffmpeg-rockchip (libavcodec)            │
│              rkmppdec.c ← 你目前使用的层              │
│   ┌───────────────────────────────────────────────┐ │
│   │  rkmpp_send_packet() → rkmpp_write_data()    │ │
│   │  rkmpp_receive_frame() → rkmpp_retrieve_frame│ │
│   │  rkmpp_flush() → mpi->reset()                │ │
│   │  rkmpp_release_frame() → buffer 归池          │ │
│   └───────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│                  MPP 用户态库                        │
│   mpp_create / mpp_init / mpi->control / mpi->reset │
│   MppBufferGroup (帧 buffer 池)                      │
├─────────────────────────────────────────────────────┤
│                 rkvdec 内核驱动                      │
│   /dev/rkvdec0  /dev/rkvdec1                        │
│   DMA-buf fd 管理 / VPU 硬件状态机                    │
├─────────────────────────────────────────────────────┤
│                 VPU 硬件 (RK3588)                    │
│   rkvdec core0 / core1  (双核解码)                   │
└─────────────────────────────────────────────────────┘
```

**关键认知**：你通过 ffmpeg-rockchip 调用的每个 API，底层都映射到 MPP 的 `mpi` 接口。理解这一映射关系是解决问题的前提。

---

## 2. rkmppdec.c 源码解析

ffmpeg-rockchip 的 `rkmppdec.c` 是你实际运行的解码器胶水层，以下是关键常量和函数：

### 2.1 关键常量

```c
#define RECEIVE_FRAME_TIMEOUT  100    // 接收帧超时 100ms
#define INPUT_MAX_PACKETS      4      // 输入队列最大深度
#define FRAMEGROUP_MAX_FRAMES  16     // 帧缓冲组最大帧数
```

### 2.2 核心调用链映射

| ffmpeg-rockchip 层 | MPP 底层调用 | 说明 |
|---|---|---|
| `rkmpp_init_decoder()` | `mpp_create()` + `mpp_init()` | 创建并初始化 MPP 上下文 |
| `rkmpp_send_packet()` | `mpi->decode_put_packet()` | 喂入压缩数据 |
| `rkmpp_retrieve_frame()` | `mpi->decode_get_frame()` | 获取解码帧 |
| `rkmpp_release_frame()` | `mpp_frame_deinit()` / buffer 归池 | 帧释放回调 |
| `rkmpp_flush()` | `mpi->reset()` | **重置解码器状态** |
| `rkmpp_close_decoder()` | `mpp_destroy()` | 销毁 MPP 上下文 |

### 2.3 rkmpp_flush() — 你最需要关注的函数

```c
// ffmpeg-rockchip 内部实现 (简化)
static void rkmpp_flush(AVCodecContext *avctx)
{
    RKMPPDecodeContext *rkctx = avctx->priv_data;
    // 这就是关键！调用了 mpi->reset(ctx)
    rkctx->mpi->reset(rkctx->ctx);
}
```

**对应 ffmpeg 用户态调用**：
```c
avcodec_flush_buffers(avctx);  // → 内部调用 rkmpp_flush() → mpi->reset()
```

> ⚠️ **核心发现**：很多开发者不知道 `avcodec_flush_buffers()` 底层会调用 `mpi->reset()`。这是从 ffmpeg 层面复位 VPU 硬件的唯一正确入口。

---

## 3. Buffer 模式与 DMA-buf 管理

### 3.1 三种 Buffer 模式对比

| 模式 | ffmpeg-rockchip 对应 | 内存控制 | DMA-buf 泄漏风险 | 适用场景 |
|---|---|---|---|---|
| **纯内部模式** | `buf_mode=0` (默认 half) | ❌ 不可控 | 🔴 **高** | 简单播放 |
| **半内部模式** | `buf_mode=0` + `MppBufferGroup` | ⚠️ 部分可控 | 🟡 中 | 需要限制内存 |
| **纯外部模式** | `buf_mode=1` (ext) | ✅ 完全可控 | 🟢 **低** | 零拷贝/AI推理 |

### 3.2 你应该用 `buf_mode=1`（纯外部模式）

**原因**：你的场景（AI 检测 + 码流波动）需要：
1. 精确控制 DMA-buf 生命周期，防止泄漏
2. 零拷贝传递解码帧给 AI 推理模块
3. 异常时能主动回收所有 buffer

**ffmpeg-rockchip 中的配置**：
```c
AVDictionary *opts = NULL;
av_dict_set_int(&opts, "buf_mode", 1, 0);  // 纯外部模式
avcodec_open2(avctx, codec, &opts);
```

### 3.3 DMA-buf 生命周期（正常 vs 异常）

```
正常流程:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ MPP 申请  │ →  │ VPU 解码  │ →  │ 用户使用  │ →  │ 释放归池  │
│ DMA-buf  │    │ 写入数据  │    │ AI推理等  │    │ refcnt--  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                    ↓
                                              refcnt=0 → 内核释放

异常流程（你遇到的问题）:
┌──────────┐    ┌──────────┐    ┌──────────┐
│ MPP 申请  │ →  │ VPU 解码  │ →  ❌ rkvdec core resting
│ DMA-buf  │    │ 卡死     │     卡死 → 无输出
└──────────┘    └──────────┘
                     ↓
            应用层超时退出/异常退出
                     ↓
            MPP frame 没有被 retrieve
            → rkmpp_release_frame() 没被调用
            → MppFrame refcnt > 0
            → DMA-buf fd 没有 close
            → 内核 DMA-buf 永久泄漏
```

---

## 4. 健康检测与 Reset 机制

### 4.1 问题定位：为什么 rkvdec 会卡住

```
码流波动场景:
  RTSP/网络流 → 比特率突变 / 丢包 / 关键帧损坏
        ↓
  ffmpeg 解封装层 → 输出残缺的 AVPacket
        ↓
  rkmpp_send_packet() → 喂入残缺码流
        ↓
  VPU 硬件解析异常 → rkvdec core resting
        ↓
  rkmpp_retrieve_frame() → 永远等不到输出
        ↓
  RECEIVE_FRAME_TIMEOUT(100ms) 超时 → 返回 AVERROR(EAGAIN)
        ↓
  循环重试 → 持续超时 → DMA-buf 越积越多
```

### 4.2 解码器健康检测方案

```c
/**
 * 解码器健康状态监控器
 * 检测连续解码超时/失败，自动触发 reset
 */
typedef struct {
    int consecutive_timeouts;     // 连续超时次数
    int consecutive_errors;       // 连续错误次数
    int total_timeouts;           // 总超时次数
    int total_resets;             // 总 reset 次数
    int64_t last_success_ms;      // 上次成功解码时间戳
    int need_idr_frame;           // reset 后是否需要等待 IDR 帧
} DecoderHealthMonitor;

#define MAX_CONSECUTIVE_TIMEOUTS  10   // 连续超时阈值 → 触发 reset
#define MAX_CONSECUTIVE_ERRORS    3    // 连续错误阈值 → 触发 reset
#define STALE_THRESHOLD_MS       5000  // 5秒无成功输出 → 触发 reset

static void health_monitor_init(DecoderHealthMonitor *hm)
{
    memset(hm, 0, sizeof(*hm));
    hm->need_idr_frame = 0;
}

/**
 * 记录解码成功
 */
static void health_monitor_record_success(DecoderHealthMonitor *hm)
{
    hm->consecutive_timeouts = 0;
    hm->consecutive_errors   = 0;
    hm->last_success_ms      = av_gettime_relative() / 1000;
}

/**
 * 记录解码超时 (EAGAIN)
 * @return 1 表示需要 reset，0 表示继续
 */
static int health_monitor_record_timeout(DecoderHealthMonitor *hm)
{
    hm->consecutive_timeouts++;
    hm->total_timeouts++;

    if (hm->consecutive_timeouts >= MAX_CONSECUTIVE_TIMEOUTS) {
        return 1;  // 需要 reset
    }

    // 检查是否长时间无输出
    int64_t now = av_gettime_relative() / 1000;
    if (hm->last_success_ms > 0 &&
        (now - hm->last_success_ms) > STALE_THRESHOLD_MS) {
        return 1;  // 需要 reset
    }

    return 0;
}

/**
 * 记录解码错误
 * @return 1 表示需要 reset，0 表示继续
 */
static int health_monitor_record_error(DecoderHealthMonitor *hm)
{
    hm->consecutive_errors++;

    if (hm->consecutive_errors >= MAX_CONSECUTIVE_ERRORS) {
        return 1;  // 需要 reset
    }
    return 0;
}
```

### 4.3 安全 Reset 流程

```c
/**
 * 安全地重置解码器
 *
 * 关键步骤：
 * 1. 先 flush ffmpeg 内部缓冲（触发 rkmpp_flush → mpi->reset）
 * 2. 清空输入队列中的残余 packet
 * 3. 标记需要等待 IDR 帧
 * 4. 重新打开解码器（如果 flush 后仍异常）
 */
static int safe_reset_decoder(AVCodecContext *avctx,
                               AVFormatContext *fmtctx,
                               DecoderHealthMonitor *hm)
{
    int ret;

    hm->total_resets++;
    fprintf(stderr, "[RKMPP] 触发解码器 reset (第 %d 次)\n", hm->total_resets);

    // === 步骤 1：flush 解码器 ===
    // 这会调用 rkmpp_flush() → mpi->reset(ctx)
    avcodec_flush_buffers(avctx);

    // === 步骤 2：清空 ffmpeg 内部缓存的残余 packet ===
    AVPacket *flush_pkt = av_packet_alloc();
    while (avcodec_receive_frame(avctx, NULL) == 0) {
        // 丢弃残余帧
    }
    av_packet_free(&flush_pkt);

    // === 步骤 3：标记需要等待 IDR 帧 ===
    hm->need_idr_frame = 1;
    hm->consecutive_timeouts = 0;
    hm->consecutive_errors   = 0;

    // === 步骤 4：如果 flush 后仍然异常，重建解码器 ===
    // 这是最激进的方式，但能彻底恢复内核驱动状态
    // 仅在 flush 无效时使用
    return 0;
}

/**
 * 完全重建解码器（最后手段）
 * 当 avcodec_flush_buffers 无法恢复时使用
 */
static int rebuild_decoder(AVCodecContext **avctx_ref,
                            const AVCodec *codec,
                            AVStream *video_stream)
{
    AVCodecContext *avctx = *avctx_ref;

    // 销毁旧解码器
    avcodec_free_context(&avctx);

    // 重新创建
    avctx = avcodec_alloc_context3(codec);
    if (!avctx)
        return AVERROR(ENOMEM);

    int ret = avcodec_parameters_to_context(avctx, video_stream->codecpar);
    if (ret < 0) {
        avcodec_free_context(&avctx);
        return ret;
    }

    AVDictionary *opts = NULL;
    av_dict_set_int(&opts, "buf_mode", 1, 0);  // 纯外部模式
    ret = avcodec_open2(avctx, codec, &opts);
    av_dict_free(&opts);
    if (ret < 0) {
        avcodec_free_context(&avctx);
        return ret;
    }

    *avctx_ref = avctx;
    return 0;
}
```

### 4.4 IDR 帧等待机制

```c
/**
 * 检查 AVPacket 是否为 IDR 帧
 * 支持 H.264 和 H.265
 */
static int is_idr_packet(const AVPacket *pkt, enum AVCodecID codec_id)
{
    if (pkt->size < 4)
        return 0;

    const uint8_t *data = pkt->data;
    int nalu_type;

    if (codec_id == AV_CODEC_ID_H264) {
        // H.264: NAL type 在第一个字节低 5 位
        // 跳过 start code (00 00 00 01 或 00 00 01)
        int offset = (data[0] == 0 && data[1] == 0 && data[2] == 0) ? 4 :
                     (data[0] == 0 && data[1] == 0) ? 3 : 0;
        if (offset >= pkt->size)
            return 0;
        nalu_type = data[offset] & 0x1F;
        return (nalu_type == 5);  // H.264 IDR slice
    }
    else if (codec_id == AV_CODEC_ID_H265) {
        int offset = (data[0] == 0 && data[1] == 0 && data[2] == 0) ? 4 :
                     (data[0] == 0 && data[1] == 0) ? 3 : 0;
        if (offset >= pkt->size)
            return 0;
        nalu_type = (data[offset] >> 1) & 0x3F;
        // H.265: BLA_W_LP=16,16~21 都是 IDR/BLA/CRA
        return (nalu_type >= 16 && nalu_type <= 21);
    }

    return 0;
}
```

---

## 5. DMA-buf 泄漏修复

### 5.1 根因总结

| 泄漏场景 | 根因 | 修复方式 |
|---|---|---|
| 解码超时后继续喂帧 | VPU 卡死，MPP 无法 dequeue frame，buffer 无法归池 | 超时检测 + reset |
| 进程异常退出 | MPP ctx 未走 destroy 流程，内核 DMA-buf refcnt > 0 | 信号捕获 + 优雅退出 |
| Info Change 未处理 | 分辨率变化后旧 buffer 未释放 | 正确处理 INFO_CHANGE |
| buf_mode=0 纯内部模式 | MPP 内部分配的 buffer 无法主动释放 | 改用 buf_mode=1 |

### 5.2 信号捕获与优雅退出

```c
#include <signal.h>

static volatile int g_interrupted = 0;

static void signal_handler(int sig)
{
    g_interrupted = 1;
    fprintf(stderr, "[RKMPP] 收到信号 %d，准备优雅退出...\n", sig);
}

// 在 main() 开头注册
void setup_signal_handlers(void)
{
    signal(SIGINT,  signal_handler);
    signal(SIGTERM, signal_handler);
    signal(SIGPIPE, SIG_IGN);  // 防止 RTSP 断连时 SIGPIPE
}
```

### 5.3 资源释放的严格顺序

```c
/**
 * 严格按顺序释放 MPP 资源
 * 顺序错误会导致 DMA-buf 泄漏！
 */
void rkmpp_cleanup(AVCodecContext *avctx)
{
    if (!avctx)
        return;

    // 1. 先 flush，触发 mpi->reset()，让硬件完成当前操作
    avcodec_flush_buffers(avctx);

    // 2. 取出所有残余帧，确保 rkmpp_release_frame() 被调用
    AVFrame *frame = av_frame_alloc();
    while (avcodec_receive_frame(avctx, frame) == 0) {
        av_frame_unref(frame);
    }
    av_frame_free(&frame);

    // 3. 销毁解码器上下文
    //    rkmpp_close_decoder() 内部会执行：
    //    - mpi->reset(ctx)    ← 再次确保硬件复位
    //    - mpp_destroy(ctx)   ← 销毁 MPP 上下文，释放所有 buffer group
    avcodec_free_context(&avctx);
}
```

### 5.4 监控 DMA-buf 泄漏

```c
/**
 * 读取当前 DMA-buf 数量和总大小
 * 从 /sys/kernel/debug/dma_buf/bufinfo 获取
 * 用于运行时监控
 */
typedef struct {
    int count;        // DMA-buf 数量
    long total_size;  // 总大小 (字节)
} DmaBufInfo;

static int get_dmabuf_info(DmaBufInfo *info)
{
    FILE *fp = fopen("/sys/kernel/debug/dma_buf/bufinfo", "r");
    if (!fp) {
        // 可能需要 root 权限
        return -1;
    }

    info->count = 0;
    info->total_size = 0;

    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        // 每个 buffer 以 "Dma-buf:" 开头
        if (strncmp(line, "Dma-buf:", 8) == 0) {
            info->count++;
            // 解析 size 字段
            char *size_str = strstr(line, "size:");
            if (size_str) {
                long size;
                if (sscanf(size_str, "size:%ld", &size) == 1) {
                    info->total_size += size;
                }
            }
        }
    }

    fclose(fp);
    return 0;
}

/**
 * 在解码循环中定期检查
 */
void check_dmabuf_leak(void)
{
    static DmaBufInfo last_info = {0, 0};
    DmaBufInfo cur_info;

    if (get_dmabuf_info(&cur_info) == 0) {
        if (cur_info.count > last_info.count + 5) {
            fprintf(stderr,
                    "[RKMPP] ⚠️ DMA-buf 泄漏告警: count=%d (+%d), "
                    "total_size=%.2f MB\n",
                    cur_info.count,
                    cur_info.count - last_info.count,
                    cur_info.total_size / (1024.0 * 1024.0));
        }
        last_info = cur_info;
    }
}
```

---

## 6. 完整封装：健壮的 RKMppDecoder

```c
/**
 * RKMppDecoder - 针对 RK3588 码流波动场景的健壮解码器封装
 *
 * 特性：
 * - 自动健康检测 + 超时 reset
 * - IDR 帧等待机制
 * - DMA-buf 泄漏监控
 * - 优雅退出
 * - 自动重建解码器
 */

#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/hwcontext.h>
#include <libavutil/hwcontext_drm.h>
#include <libavutil/time.h>

#define RKMPP_DECODER_VERSION "1.0.0"

typedef struct RKMppDecoder {
    // FFmpeg 上下文
    AVFormatContext    *fmt_ctx;
    AVCodecContext     *codec_ctx;
    const AVCodec      *codec;
    int                 video_stream_idx;

    // 健康监控
    DecoderHealthMonitor health;

    // 配置
    int                 buf_mode;          // 0=half, 1=ext
    int                 max_consecutive_timeouts;
    int                 max_consecutive_errors;
    int                 stale_threshold_ms;
    int                 auto_rebuild;      // reset 无效时自动重建

    // 统计
    int64_t             frames_decoded;
    int64_t             frames_dropped;
    int                 reset_count;
    int                 rebuild_count;

} RKMppDecoder;

/* ========== 初始化 ========== */

int rkmpp_decoder_init(RKMppDecoder *dec,
                       const char *input_url,
                       const char *decoder_name,  // "h264_rkmpp" / "hevc_rkmpp"
                       int buf_mode)
{
    int ret;

    memset(dec, 0, sizeof(*dec));
    dec->buf_mode = buf_mode;
    dec->max_consecutive_timeouts = 10;
    dec->max_consecutive_errors   = 3;
    dec->stale_threshold_ms       = 5000;
    dec->auto_rebuild             = 1;

    health_monitor_init(&dec->health);

    // 打开输入
    ret = avformat_open_input(&dec->fmt_ctx, input_url, NULL, NULL);
    if (ret < 0) {
        fprintf(stderr, "[RKMPP] 无法打开输入: %s\n", input_url);
        return ret;
    }

    ret = avformat_find_stream_info(dec->fmt_ctx, NULL);
    if (ret < 0) {
        fprintf(stderr, "[RKMPP] 无法获取流信息\n");
        return ret;
    }

    // 查找视频流
    dec->video_stream_idx = av_find_best_stream(dec->fmt_ctx,
        AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);
    if (dec->video_stream_idx < 0) {
        fprintf(stderr, "[RKMPP] 未找到视频流\n");
        return dec->video_stream_idx;
    }

    // 查找解码器
    dec->codec = avcodec_find_decoder_by_name(decoder_name);
    if (!dec->codec) {
        fprintf(stderr, "[RKMPP] 解码器 '%s' 未找到\n", decoder_name);
        return AVERROR_DECODER_NOT_FOUND;
    }

    // 创建解码器上下文
    dec->codec_ctx = avcodec_alloc_context3(dec->codec);
    if (!dec->codec_ctx)
        return AVERROR(ENOMEM);

    AVStream *video_stream = dec->fmt_ctx->streams[dec->video_stream_idx];
    ret = avcodec_parameters_to_context(dec->codec_ctx, video_stream->codecpar);
    if (ret < 0)
        return ret;

    // 配置 MPP 特有参数
    AVDictionary *opts = NULL;
    av_dict_set_int(&opts, "buf_mode", dec->buf_mode, 0);
    av_dict_set_int(&opts, "fast_parse", 1, 0);  // 快速解析提升并行性

    ret = avcodec_open2(dec->codec_ctx, dec->codec, &opts);
    av_dict_free(&opts);
    if (ret < 0) {
        fprintf(stderr, "[RKMPP] 无法打开解码器\n");
        return ret;
    }

    fprintf(stderr, "[RKMPP] 初始化完成: decoder=%s, buf_mode=%d\n",
            decoder_name, dec->buf_mode);

    return 0;
}

/* ========== 核心解码循环 ========== */

/**
 * 从输入读取一个 packet 并送入解码器
 *
 * @return:
 *   0  - 成功送入
 *   1  - 跳过（等待 IDR 帧）
 *   <0 - 错误
 */
int rkmpp_decoder_send(RKMppDecoder *dec, AVPacket *pkt)
{
    int ret;

    // 读取输入 packet
    ret = av_read_frame(dec->fmt_ctx, pkt);
    if (ret < 0) {
        if (ret == AVERROR_EOF) {
            fprintf(stderr, "[RKMPP] 输入 EOF\n");
        }
        return ret;
    }

    // 仅处理视频流
    if (pkt->stream_index != dec->video_stream_idx) {
        av_packet_unref(pkt);
        return 1;  // 非视频流，跳过
    }

    // Reset 后需要等待 IDR 帧
    if (dec->health.need_idr_frame) {
        enum AVCodecID codec_id = dec->codec_ctx->codec_id;
        if (!is_idr_packet(pkt, codec_id)) {
            dec->frames_dropped++;
            av_packet_unref(pkt);
            return 1;  // 非 IDR 帧，丢弃
        }
        // 找到 IDR 帧，恢复正常
        dec->health.need_idr_frame = 0;
        fprintf(stderr, "[RKMPP] 捕获 IDR 帧，恢复解码\n");
    }

    // 发送到解码器
    ret = avcodec_send_packet(dec->codec_ctx, pkt);
    if (ret < 0 && ret != AVERROR(EAGAIN)) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "[RKMPP] send_packet 错误: %s\n", errbuf);

        if (health_monitor_record_error(&dec->health)) {
            // 连续错误超阈值，触发 reset
            safe_reset_decoder(dec->codec_ctx, dec->fmt_ctx, &dec->health);
            dec->reset_count++;
        }
    }
    av_packet_unref(pkt);

    return ret;
}

/**
 * 从解码器获取一帧
 *
 * @return:
 *   0  - 成功获取一帧
 *   AVERROR(EAGAIN) - 需要更多输入
 *   AVERROR_EOF     - 解码结束
 *   <0              - 错误
 */
int rkmpp_decoder_receive(RKMppDecoder *dec, AVFrame *frame)
{
    int ret = avcodec_receive_frame(dec->codec_ctx, frame);

    if (ret == 0) {
        // 解码成功
        health_monitor_record_success(&dec->health);
        dec->frames_decoded++;
        return 0;
    }

    if (ret == AVERROR(EAGAIN)) {
        // 超时/需要更多输入
        if (health_monitor_record_timeout(&dec->health)) {
            // 连续超时超阈值，触发 reset
            fprintf(stderr, "[RKMPP] 连续超时 %d 次，触发 reset\n",
                    dec->health.consecutive_timeouts);
            safe_reset_decoder(dec->codec_ctx, dec->fmt_ctx, &dec->health);
            dec->reset_count++;

            // 如果 reset 多次无效，尝试重建
            if (dec->auto_rebuild && dec->health.total_resets % 3 == 0) {
                AVStream *vs = dec->fmt_ctx->streams[dec->video_stream_idx];
                int rret = rebuild_decoder(&dec->codec_ctx, dec->codec, vs);
                if (rret == 0) {
                    dec->rebuild_count++;
                    fprintf(stderr, "[RKMPP] 解码器已重建 (第 %d 次)\n",
                            dec->rebuild_count);
                }
            }
        }
        return AVERROR(EAGAIN);
    }

    if (ret == AVERROR_EOF) {
        return AVERROR_EOF;
    }

    // 其他错误
    char errbuf[AV_ERROR_MAX_STRING_SIZE];
    av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "[RKMPP] receive_frame 错误: %s\n", errbuf);

    if (health_monitor_record_error(&dec->health)) {
        safe_reset_decoder(dec->codec_ctx, dec->fmt_ctx, &dec->health);
        dec->reset_count++;
    }

    return ret;
}

/* ========== 清理 ========== */

void rkmpp_decoder_destroy(RKMppDecoder *dec)
{
    fprintf(stderr, "[RKMPP] 统计: decoded=%ld, dropped=%ld, "
            "resets=%d, rebuilds=%d\n",
            dec->frames_decoded, dec->frames_dropped,
            dec->reset_count, dec->rebuild_count);

    rkmpp_cleanup(dec->codec_ctx);
    dec->codec_ctx = NULL;

    if (dec->fmt_ctx) {
        avformat_close_input(&dec->fmt_ctx);
    }
}

/* ========== 使用示例 ========== */

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "用法: %s <rtsp_url_or_file>\n", argv[0]);
        return 1;
    }

    setup_signal_handlers();

    RKMppDecoder dec;
    int ret = rkmpp_decoder_init(&dec, argv[1], "h264_rkmpp", 1);
    if (ret < 0)
        return 1;

    AVPacket *pkt = av_packet_alloc();
    AVFrame  *frame = av_frame_alloc();
    int dmabuf_check_counter = 0;

    while (!g_interrupted) {
        // 发送数据
        ret = rkmpp_decoder_send(&dec, pkt);
        if (ret == AVERROR_EOF) {
            // 发送 flush packet
            avcodec_send_packet(dec.codec_ctx, NULL);
        } else if (ret < 0 && ret != AVERROR(EAGAIN)) {
            // 严重错误，可能需要重建
            if (dec.auto_rebuild) {
                AVStream *vs = dec.fmt_ctx->streams[dec.video_stream_idx];
                rebuild_decoder(&dec.codec_ctx, dec.codec, vs);
                dec.rebuild_count++;
            }
            continue;
        }

        // 接收帧
        ret = rkmpp_decoder_receive(&dec, frame);
        if (ret == 0) {
            // ✅ 解码成功，将 frame 送入 AI 推理模块
            // 如果 frame 是 DRM_PRIME 格式（零拷贝），
            // 可以直接拿到 DMA-buf fd
            if (frame->format == AV_PIX_FMT_DRM_PRIME) {
                AVDRMFrameDescriptor *desc = (AVDRMFrameDescriptor *)frame->data[0];
                // desc->objects[0].fd 就是 DMA-buf fd
                // 可以传给 RGA / RKNN 等硬件模块
            }

            // 使用完毕后必须 unref，否则 DMA-buf 不会归还
            av_frame_unref(frame);
        }

        // 定期检查 DMA-buf 泄漏（每 100 帧检查一次）
        if (++dmabuf_check_counter >= 100) {
            check_dmabuf_leak();
            dmabuf_check_counter = 0;
        }
    }

    rkmpp_decoder_destroy(&dec);
    av_packet_free(&pkt);
    av_frame_free(&frame);

    return 0;
}
```

---

## 7. 调试排查

### 7.1 实时监控 DMA-buf

```bash
# 查看 DMA-buf 总量
cat /sys/kernel/debug/dma_buf/bufinfo | grep -c "^Dma-buf"

# 查看详细的 DMA-buf 信息（大小、exporter）
cat /sys/kernel/debug/dma_buf/bufinfo

# 实时监控（每 2 秒刷新）
watch -n 2 'cat /sys/kernel/debug/dma_buf/bufinfo | head -5'

# 查看进程持有的 fd 数量（间接反映 buffer 泄漏）
ls -la /proc/$(pgrep your_app)/fd | wc -l

# 查看进程的内存映射
cat /proc/$(pgrep your_app)/maps | grep -c dmabuf
```

### 7.2 查看 rkvdec 驱动状态

```bash
# 查看 rkvdec 内核日志
dmesg | grep -i rkvdec | tail -50

# 实时监控 rkvdec 日志
dmesg -w | grep --line-buffered -i "rkvdec\|vpu\|mpp"

# 查看 VPU 设备节点
ls -la /dev/rkvdec*

# 查看 VPU 频率状态
cat /sys/kernel/debug/clk/rkvdec0/clk_rate
cat /sys/kernel/debug/clk/rkvdec1/clk_rate
```

### 7.3 内核模块重载（不需重启设备）

```bash
# 当 rkvdec 驱动卡死，但不想重启设备时
sudo rmmod rkvdec
sudo modprobe rkvdec

# 如果 rkvdec 是内建的，尝试通过 sysfs 复位
sudo bash -c 'echo 1 > /sys/kernel/debug/rkvdec0/reset'  # 4.19 内核
sudo bash -c 'echo 1 > /sys/kernel/debug/rkvdec1/reset'  # RK3588 双核
```

### 7.4 MPP 版本与配置确认

```bash
# 查看 MPP 版本
strings /usr/lib/aarch64-linux-gnu/librockchip_mpp.so | grep "MPP version"

# 查看 ffmpeg-rockchip 支持的解码器
ffmpeg -hide_banner -decoders 2>/dev/null | grep rkmpp

# 查看解码器详细选项
ffmpeg -hide_banner -h decoder=h264_rkmpp
ffmpeg -hide_banner -h decoder=hevc_rkmpp

# 测试解码性能
ffmpeg -stream_loop -1 -hwaccel rkmpp \
  -hwaccel_output_format drm_prime \
  -i test.mp4 -an -sn -vframes 5000 -f null -
```

### 7.5 码流分析工具

```bash
# 分析 H.264 码流结构（确认 IDR 帧间隔）
ffprobe -show_frames -select_streams v:0 input.mp4 | \
  grep -E "key_frame|pict_type|pkt_pts"

# 提取裸流（MPP 需要裸流输入）
ffmpeg -i input.mp4 -c:v copy -bsf:v h264_mp4toannexb output.h264

# 检查网络流质量
ffprobe -v error -show_entries stream=codec_name,width,height,bit_rate \
  -of csv=p=0 rtsp://your_stream_url
```

---

## 附录：关键 API 速查

### ffmpeg-rockchip 层

| API | 功能 | 注意事项 |
|---|---|---|
| `avcodec_find_decoder_by_name("h264_rkmpp")` | 查找硬解码器 | 不是 `avcodec_find_decoder()` |
| `av_dict_set_int(&opts, "buf_mode", 1, 0)` | 设置 buffer 模式 | **必须用 ext 模式** |
| `av_dict_set_int(&opts, "fast_parse", 1, 0)` | 快速解析 | 提升并行解码性能 |
| `av_dict_set_int(&opts, "afbc", 1, 0)` | AFBC 压缩 | 不能与 hwdownload 共用 |
| `avcodec_flush_buffers(avctx)` | **Flush + Reset** | 底层调用 `mpi->reset()` |
| `frame->format == AV_PIX_FMT_DRM_PRIME` | 零拷贝帧 | DMA-buf fd 在 `data[0]` |

### MPP 底层（供参考）

| API | 功能 |
|---|---|
| `mpp_create(&ctx, &mpi)` | 创建 MPP 上下文 |
| `mpp_init(ctx, MPP_CTX_DEC, type)` | 初始化解码器 |
| `mpi->decode_put_packet(ctx, pkt)` | 输入码流 |
| `mpi->decode_get_frame(ctx, &frame)` | 获取解码帧 |
| `mpi->reset(ctx)` | **复位硬件状态机** |
| `mpp_destroy(ctx)` | 销毁上下文 |
| `mpp_buffer_group_limit_config(grp, size, count)` | 限制 buffer 池大小 |
| `MPP_DEC_SET_PARSER_SPLIT_MODE` | 开启自动拼包 |
| `MPP_DEC_SET_EXT_BUF_GROUP` | 设置外部 buffer 组 |

---

## 附录：ffmpeg-rockchip 解码器列表 (RK3588)

| 解码器名 | 编码格式 | 备注 |
|---|---|---|
| `h264_rkmpp` | H.264/AVC | 最常用 |
| `hevc_rkmpp` | H.265/HEVC | AI检测常用 |
| `vp9_rkmpp` | VP9 | WebM |
| `av1_rkmpp` | AV1 | 新增支持 |
| `mjpeg_rkmpp` | MJPEG | 摄像头直出 |
| `mpeg1_rkmpp` | MPEG1 | 老旧格式 |
| `mpeg2_rkmpp` | MPEG2 | 老旧格式 |
| `mpeg4_rkmpp` | MPEG4 | 老旧格式 |
| `h263_rkmpp` | H.263 | 老旧格式 |
| `vp8_rkmpp` | VP8 | WebM |

---

## 总结：你问题的完整修复路径

```
1. 设置 buf_mode=1（纯外部模式）           → 可控 DMA-buf 生命周期
2. 添加健康检测 + 超时计数                → 及时发现 rkvdec 卡死
3. 超时时调用 avcodec_flush_buffers()     → 触发 mpi->reset() 复位硬件
4. Reset 后等待 IDR 帧                   → 避免残缺码流再次卡死
5. av_frame_unref() 及时释放              → DMA-buf 归池，防止泄漏
6. 优雅退出 + 严格释放顺序               → 进程退出不残留内核对象
7. 定期监控 /sys/kernel/debug/dma_buf    → 运行时泄漏告警
```
