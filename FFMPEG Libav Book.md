# FFMPEG Libav Book

TOC - TODO
* Motivation
* Thanks
* Concepts
* Setup
* Hello World
* Makefile
* Trimming 
* Muxing mov to mp4
* Transcode video (mov to mp4)
* Transcode video FPS/bitrate
* Transcode audio (aac to aac)
* Transcode audio change sampling rate
* Transcode audio and video together
* Dropping streams
* Transcode audio with buffer for sampling
* Transcode audio scl16 format to aac
* Transcode video webm to mp4
* Transcode gif to mp4
* Complex filters
* 
# Motivation
April 2024
I am not new to FFMPEG as a user, but I am new to using ffmpeg as a developer with its libav C libraries. 
The official documentation is good as an API doc, but it doesn't explain concepts to a newbie. The examples too don't run out of the box on all files.
I was stumped when I started and struggled for even small things.

https://github.com/leandromoreira/ffmpeg-libav-tutorial
is an excellent resource and a recommended read for starting. It helped me understand the concepts and use of the data structures in libav. 

Still I had my own path and I found myself struggling to put together code which ran without Seg faults and did what I  needed. Somehow I managed to cobble code from various sources  stackoverflow, ffmpeg mailing lists, the ffmpeg source code, leadndromoriera's git and other blogs.

I am creating the book I wish I had when I started on this journey.

Comments  to improve this or a shoutout if you find this useful are appreciated.
Email - amythical@gmail.com
Twitter/X  - @amythical

#	Concepts 
Thanks to https://github.com/leandromoreira/ffmpeg-libav-tutorial
* AVPacket - encoded data
* AVFrame - decoded data
* AVStream has AVPackets
* AVFormatContext ~ Container has AVStreams

# Setup
## Environment
* Mac M1 Sonoma
* Docker Desktop client
* OS Debain

### Debian Docker instance
Pull a docker debian instance, 'unstable-slim' is the  debian image I searched and got on the docker repos

```
	docker run -it --entrypoint=bash debian:unstable-slim
```

### Install FFMPEG
	https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu

## For Standalone FFMPEG on Mac M1 (No Docker)
*pkg-config not finding gnutls error*
```
brew outdated "pkg-config" || brew upgrade "pkg-config"
```

### New docker image
Save container as a new image, so we have an image with ffmpeg installed
```
docker commit commithashabcd345r debian-ffmpeg-libav
```

### Attach a docker volume
A docker volume will help us keep the code external from the VM.
In the docker desktop client create volume in docker desktop called ffmpeg-code
docker run -v ffmpeg-code:/ffmpeg-code -it --entrypoint=bash debian-ffmpeg-libav



# Hello World Libav
The hello world of libav would be to read a video file and print some information from it like the format name, duration and streams.

## Mp4Info
```
#include <stdio.h>
#include <libavformat/avformat.h>
#include <libavutil/avutil.h>

void getinfo(){
  int tWriteFlag = 0;
  
  AVFormatContext* fmt_ctx = avformat_alloc_context();
  // Open the file - demo.mp4 (any mp4 would do)
  int ret;
  if ((ret = avformat_open_input(&fmt_ctx, "demo.mp4", NULL,NULL)) < 0) {
      printf("%s", av_err2str(ret));
  }
  avformat_find_stream_info(fmt_ctx, NULL);// we dont get the duration without this

  printf("format: %s\n duration: %f seconds\n streams: %d\n",
    fmt_ctx->iformat->name,
    (fmt_ctx->duration/10e5),// micro sec to sec conversion
    fmt_ctx->nb_streams);
  

   // Cleanup
  avformat_close_input(&fmt_ctx);
  avformat_free_context(fmt_ctx);
  fflush(stdin);
}

void main(){
  getinfo();
}

```

## Running mp4info
After numerous tried, trying to check missing libraries while linking here is the final command

Make sure any missing libraries from this command are installed.

Compile and Link command
```
 gcc -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib -o trim trim.c -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm
```
Later we will put this in a make file

## Output
```
// ./mp4info on a terminal should give
format: mov,mp4,m4a,3gp,3g2,mj2
duration: 20.640000 secs
streams: 2
```


# Trimming a video 
Trim.c
```
#include <libavformat/avformat.h>
#include <libavformat/avio.h>
#include <libavcodec/packet.h>
#include <libavutil/log.h>
#include <libavutil/timestamp.h>
#include <libavutil/rational.h>
#include <stdio.h>
#include <stdarg.h>
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>

static void logging(char *msg){
  av_log(NULL, AV_LOG_DEBUG, "%s\n",msg);
}

//int cutFile(const char* inputFilePath, const long long& startSeconds, const long long& endSeconds,
//                            const char* outputFilePath) {
int cutFile(){

  char *inputFilePath="kohli.mp4";
  char *outputFilePath="out.mp4";

  int startSeconds = 5;
  int endSeconds = 15;// indicates the duration and not the time

  int operationResult;

  AVPacket* avPacket = NULL;
  AVFormatContext* avInputFormatContext = NULL;
  AVFormatContext* avOutputFormatContext = NULL;

  avPacket = av_packet_alloc();
  if (!avPacket) {
    logging("Failed to allocate AVPacket.");
    return -1;
  }

    operationResult = avformat_open_input(&avInputFormatContext, inputFilePath, 0, 0);
    if (operationResult < 0) {
      logging("Failed to open the input file");
      return -1;
    }

    operationResult = avformat_find_stream_info(avInputFormatContext, 0);
    if (operationResult < 0) {
      logging("Failed to retrieve the input stream information.");
    }

    avformat_alloc_output_context2(&avOutputFormatContext, NULL, NULL, outputFilePath);
    if (!avOutputFormatContext) {
      operationResult = AVERROR_UNKNOWN;
      logging("Failed to create the output context.");
    }

    int streamIndex = 0;
    int streamMapping[avInputFormatContext->nb_streams];
    int streamRescaledStartSeconds[avInputFormatContext->nb_streams];
    int streamRescaledEndSeconds[avInputFormatContext->nb_streams];

    // Copy streams from the input file to the output file.
    for (int i = 0; i < avInputFormatContext->nb_streams; i++) {
      AVStream* outStream;
      AVStream* inStream = avInputFormatContext->streams[i];

      streamRescaledStartSeconds[i] = av_rescale_q(startSeconds * AV_TIME_BASE, AV_TIME_BASE_Q, inStream->time_base);
      streamRescaledEndSeconds[i] = av_rescale_q(endSeconds * AV_TIME_BASE, AV_TIME_BASE_Q, inStream->time_base);

      if (inStream->codecpar->codec_type != AVMEDIA_TYPE_AUDIO &&
          inStream->codecpar->codec_type != AVMEDIA_TYPE_VIDEO &&
          inStream->codecpar->codec_type != AVMEDIA_TYPE_SUBTITLE) {
        streamMapping[i] = -1;
        continue;
      }

      streamMapping[i] = streamIndex++;

      outStream = avformat_new_stream(avOutputFormatContext, NULL);
      if (!outStream) {
        operationResult = AVERROR_UNKNOWN;
        logging("Failed to allocate the output stream.");
      }

      operationResult = avcodec_parameters_copy(outStream->codecpar, inStream->codecpar);
      if (operationResult < 0) {
        logging("Failed to copy codec parameters from input stream to output stream.");
      }
      outStream->codecpar->codec_tag = 0;
    }//for

    if (!(avOutputFormatContext->oformat->flags & AVFMT_NOFILE)) {
      operationResult = avio_open(&avOutputFormatContext->pb, outputFilePath, AVIO_FLAG_WRITE);
      if (operationResult < 0) {
        logging("Failed to open the output file");
      }
    }

    operationResult = avformat_write_header(avOutputFormatContext, NULL);
    if (operationResult < 0) {
      logging("Error occurred when opening output file.");
    }

    operationResult = avformat_seek_file(avInputFormatContext, -1, INT64_MIN, startSeconds * AV_TIME_BASE,
                                         startSeconds * AV_TIME_BASE, 0);
    if (operationResult < 0) {
      logging("Failed to seek the input file to the targeted start position.");
    }

    while (1) {
      operationResult = av_read_frame(avInputFormatContext, avPacket);
      if (operationResult < 0) break;

      // Skip packets from unknown streams and packets after the end cut position.
      if (avPacket->stream_index >= avInputFormatContext->nb_streams || streamMapping[avPacket->stream_index] < 0){
        av_packet_unref(avPacket);
        continue;
      }
       if(avPacket->pts >= streamRescaledEndSeconds[avPacket->stream_index]) {
        av_packet_unref(avPacket);
	break;
       }

      avPacket->stream_index = streamMapping[avPacket->stream_index];
      //logPacket(avInputFormatContext, avPacket, "in");

      // Shift the packet to its new position by subtracting the rescaled start seconds.
      avPacket->pts -= streamRescaledStartSeconds[avPacket->stream_index];
      avPacket->dts -= streamRescaledStartSeconds[avPacket->stream_index];

      av_packet_rescale_ts(avPacket, avInputFormatContext->streams[avPacket->stream_index]->time_base,
                           avOutputFormatContext->streams[avPacket->stream_index]->time_base);
      avPacket->pos = -1;
      //logPacket(avOutputFormatContext, avPacket, "out");

      operationResult = av_interleaved_write_frame(avOutputFormatContext, avPacket);
      if (operationResult < 0) {
        logging("Failed to mux the packet.");
      }
    }//while

av_write_trailer(avOutputFormatContext);
av_packet_free(&avPacket);
avformat_close_input(&avInputFormatContext;

  if (avOutputFormatContext && !(avOutputFormatContext->oformat->flags & AVFMT_NOFILE))
    avio_closep(&avOutputFormatContext->pb);
  avformat_free_context(avOutputFormatContext);

  if (operationResult < 0 && operationResult != AVERROR_EOF) {
    logging("Error occurred 2");
    return -1;
  }

  return 0;
}

/*
 * trims a file without encoding
 * starts at 5 seconds n trims 15 seconds from there
 */
int main(int args,const char* argv[])
{
 cutFile();
}
```
Makefile
```
https://www3.nd.edu/~zxu2/acms60212-40212/Makefile.pdf

# $^ refers to source i.e trim.c and $@ refers to targer i.e. trim.js
trim: trim.c
	gcc -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib $^ -o $@ -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lz -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm
```




# Muxing a video 
Muxing is keeping the same codec and changing the container, does not involve transcoding

## Mov to mp4
Both mp4 and mov have the same codec but diferent containers. Using the same codec but changing the container is called 'muxing' or 'transmuxing' and is a light weight process because it does not involve codec changes.

The same trim.c changes the container to a mp4 container because we have specified the output with a mp4 container
```
// output container is mp4
char *outputFilePath="out.mp4";
and
avformat_alloc_output_context2(&avOutputFormatContext, NULL, NULL, outputFilePath);
```

### Problem - Video duration is shorter than trim duration
On running this on one of the mov files with audio and video streams,  I encountered an issue where the trimmed file was shorted than expected video duration, example trimmed 1-9 seconds but final file was about 7 seconds.

#### Debugging
Lets debug this by looking at the source code.
* We see that trim.c breaks the packet reading while loop based on the stream index of the packet.
```
        if (tAvPacket->pts > tStreamRescaledEndSeconds[tAvPacket->stream_index])
        {
            av_packet_unref(tAvPacket);
            break;
        }
```
* If we do a *ffmpeg -i* we see that the audio stream index is 0 and the video stream index is 1. 
* We also see that the audio duration is less than the video duration
* So what is happening is the trimming stops when the audio streams end time is reached. The audio stream an index of 0 so it reached the condition to break on exceeding trim time first . 
* The end time is calculated as per each stream's TIME_BASE so that fetches us a different number for the audio stream, and in our case shorter than the actual trim time and so reading and copy of packets stops by breaking out of the loop and the video duration neds up being shorter than the expected duration

#### The fix 
* We have to stop the trimming only when the video PTS has reached the desired end trim time

```
/*
 * Trims 10 seconds of video starting from 5 seconds
 * Fixed to record video stream index and stop copying packets when the PTS exceeds the video stream PTS
*/

#include <libavformat/avformat.h>
#include <libavformat/avio.h>
#include <libavcodec/packet.h>
#include <libavutil/log.h>
#include <libavutil/timestamp.h>
#include <libavutil/rational.h>
#include <stdio.h>
#include <stdarg.h>
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>

static void logging(char *msg){
  av_log(NULL, AV_LOG_DEBUG, "%s\n",msg);
}

//int cutFile(const char* inputFilePath, const long long& startSeconds, const long long& endSeconds,
//                            const char* outputFilePath) {
int cutFile(){

  char *inputFilePath="kohli.mp4";
  char *outputFilePath="out.mp4";

  int startSeconds = 5;
  int endSeconds = 15;

  int operationResult;

  AVPacket* avPacket = NULL;
  AVFormatContext* avInputFormatContext = NULL;
  AVFormatContext* avOutputFormatContext = NULL;

  avPacket = av_packet_alloc();
  if (!avPacket) {
    logging("Failed to allocate AVPacket.");
    return -1;
  }

    operationResult = avformat_open_input(&avInputFormatContext, inputFilePath, 0, 0);
    if (operationResult < 0) {
      logging("Failed to open the input file");
      return -1;
    }

    operationResult = avformat_find_stream_info(avInputFormatContext, 0);
    if (operationResult < 0) {
      logging("Failed to retrieve the input stream information.");
    }

    avformat_alloc_output_context2(&avOutputFormatContext, NULL, NULL, outputFilePath);
    if (!avOutputFormatContext) {
      operationResult = AVERROR_UNKNOWN;
      logging("Failed to create the output context.");
    }

    int streamIndex = 0;
    int streamMapping[avInputFormatContext->nb_streams];
    int streamRescaledStartSeconds[avInputFormatContext->nb_streams];
    int streamRescaledEndSeconds[avInputFormatContext->nb_streams];
    int videoStreamIndex = 0;// FIX - record the video streams index

    // Copy streams from the input file to the output file.
    for (int i = 0; i < avInputFormatContext->nb_streams; i++) {
      AVStream* outStream;
      AVStream* inStream = avInputFormatContext->streams[i];

      streamRescaledStartSeconds[i] = av_rescale_q(startSeconds * AV_TIME_BASE, AV_TIME_BASE_Q, inStream->time_base);
      streamRescaledEndSeconds[i] = av_rescale_q(endSeconds * AV_TIME_BASE, AV_TIME_BASE_Q, inStream->time_base);

      if (inStream->codecpar->codec_type != AVMEDIA_TYPE_AUDIO &&
          inStream->codecpar->codec_type != AVMEDIA_TYPE_VIDEO &&
          inStream->codecpar->codec_type != AVMEDIA_TYPE_SUBTITLE) {
        streamMapping[i] = -1;
        continue;
      }
// FIX - record video streams index
 if(tInStream->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
                videoStreamIndex = streamIndex;


      streamMapping[i] = streamIndex++;

      outStream = avformat_new_stream(avOutputFormatContext, NULL);
      if (!outStream) {
        operationResult = AVERROR_UNKNOWN;
        logging("Failed to allocate the output stream.");
      }

      operationResult = avcodec_parameters_copy(outStream->codecpar, inStream->codecpar);
      if (operationResult < 0) {
        logging("Failed to copy codec parameters from input stream to output stream.");
      }
      outStream->codecpar->codec_tag = 0;
    }//for

    if (!(avOutputFormatContext->oformat->flags & AVFMT_NOFILE)) {
      operationResult = avio_open(&avOutputFormatContext->pb, outputFilePath, AVIO_FLAG_WRITE);
      if (operationResult < 0) {
        logging("Failed to open the output file");
      }
    }

    operationResult = avformat_write_header(avOutputFormatContext, NULL);
    if (operationResult < 0) {
      logging("Error occurred when opening output file.");
    }

    operationResult = avformat_seek_file(avInputFormatContext, -1, INT64_MIN, startSeconds * AV_TIME_BASE,
                                         startSeconds * AV_TIME_BASE, 0);
    if (operationResult < 0) {
      logging("Failed to seek the input file to the targeted start position.");
    }

    while (1) {
      operationResult = av_read_frame(avInputFormatContext, avPacket);
      if (operationResult < 0) break;

      // Skip packets from unknown streams and packets after the end cut position.
      if (avPacket->stream_index >= avInputFormatContext->nb_streams || streamMapping[avPacket->stream_index] < 0){
        av_packet_unref(avPacket);
        continue;
      }
      // FIX - break only when video pts > trim time
      if (tAvPacket->stream_index == videoStreamIndex && tAvPacket->pts > tStreamRescaledEndSeconds[videoStreamIndex])
        {
            printf("%lld %d\n",tAvPacket->pts, tStreamRescaledEndSeconds[videoStreamIndex]);
            av_packet_unref(tAvPacket);
            break;
        }

      avPacket->stream_index = streamMapping[avPacket->stream_index];
      //logPacket(avInputFormatContext, avPacket, "in");

      // Shift the packet to its new position by subtracting the rescaled start seconds.
      avPacket->pts -= streamRescaledStartSeconds[avPacket->stream_index];
      avPacket->dts -= streamRescaledStartSeconds[avPacket->stream_index];

      av_packet_rescale_ts(avPacket, avInputFormatContext->streams[avPacket->stream_index]->time_base,
                           avOutputFormatContext->streams[avPacket->stream_index]->time_base);
      avPacket->pos = -1;
      //logPacket(avOutputFormatContext, avPacket, "out");

      operationResult = av_interleaved_write_frame(avOutputFormatContext, avPacket);
      if (operationResult < 0) {
        logging("Failed to mux the packet.");
      }
    }//while

av_write_trailer(avOutputFormatContext);
av_packet_free(&avPacket);
avformat_close_input(&avInputFormatContext;

  if (avOutputFormatContext && !(avOutputFormatContext->oformat->flags & AVFMT_NOFILE))
    avio_closep(&avOutputFormatContext->pb);
  avformat_free_context(avOutputFormatContext);

  if (operationResult < 0 && operationResult != AVERROR_EOF) {
    logging("Error occurred 2");
    return -1;
  }

  return 0;
}

/*
 * trims a file without encoding
 * starts at 5 seconds n trims 15 seconds from there
 */
int main(int args,const char* argv[])
{
 cutFile();
}
```

Now on running the muxing program, the video duration is as expected

# Transcode Audio
```
/*

// Audio transcoding to aac
// Code from ffmpeg7.0 examples rehashed and rewritten some parts for my understanding

* Copyright (c) 2013-2022 Andreas Unterweger
*
* This file is part of FFmpeg.
*
* FFmpeg is free software; you can redistribute it and/or
* modify it under the terms of the GNU Lesser General Public
* License as published by the Free Software Foundation; either
* version 2.1 of the License, or (at your option) any later version.
*
* FFmpeg is distributed in the hope that it will be useful,
* but WITHOUT ANY WARRANTY; without even the implied warranty of
* MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
* Lesser General Public License for more details.
*
* You should have received a copy of the GNU Lesser General Public
* License along with FFmpeg; if not, write to the Free Software
* Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA
*/
/**

* @file audio transcoding to MPEG/AAC API usage example

* @example transcode_aac.c

*

* Convert an input audio file to AAC in an MP4 container. Formats other than

* MP4 are supported based on the output file extension.

* @author Andreas Unterweger (dustsigns@gmail.com)

*/

#include  <stdio.h>

#include  "libavformat/avformat.h"

#include  "libavformat/avio.h"

#include  "libavcodec/avcodec.h"

#include  "libavutil/audio_fifo.h"

#include  "libavutil/avassert.h"

#include  "libavutil/avstring.h"

#include  "libavutil/channel_layout.h"

#include  "libavutil/frame.h"

#include  "libavutil/opt.h"

#include  "libswresample/swresample.h"

/* The output bit rate in bit/s */

#define OUTPUT_BIT_RATE 96000

/* The number of output channels */

#define OUTPUT_CHANNELS 2

/**

* Open an input file and the required decoder.

* @param  filename File to be opened

* @param[out] input_format_context Format context of opened file

* @param[out] input_codec_context Codec context of opened file

* @return Error code (0 if successful)

*/

static  int  open_input_file(const  char *filename,

AVFormatContext **input_format_context,

AVCodecContext **input_codec_context)

{

AVCodecContext *avctx;

const AVCodec *input_codec;

const AVStream *stream;

int error;

/* Open the input file to read from it. */

if ((error = avformat_open_input(input_format_context, filename, NULL,

NULL)) < 0) {

fprintf(stderr, "Could not open input file '%s' (error '%s')\n",

filename, av_err2str(error));

*input_format_context = NULL;

return error;

}

/* Get information on the input file (number of streams etc.). */

if ((error = avformat_find_stream_info(*input_format_context, NULL)) < 0) {

fprintf(stderr, "Could not open find stream info (error '%s')\n",

av_err2str(error));

avformat_close_input(input_format_context);

return error;

}

/* Make sure that there is only one stream in the input file. */

if ((*input_format_context)->nb_streams > 2) {

fprintf(stderr, "Expected one audio input stream, but found %d\n",

(*input_format_context)->nb_streams);

avformat_close_input(input_format_context);

return AVERROR_EXIT;

}

stream = (*input_format_context)->streams[0];

/* Find a decoder for the audio stream. */

if (!(input_codec = avcodec_find_decoder(stream->codecpar->codec_id))) {

fprintf(stderr, "Could not find input codec\n");

avformat_close_input(input_format_context);

return AVERROR_EXIT;

}

/* Allocate a new decoding context. */

avctx = avcodec_alloc_context3(input_codec);

if (!avctx) {

fprintf(stderr, "Could not allocate a decoding context\n");

avformat_close_input(input_format_context);

return  AVERROR(ENOMEM);

}

/* Initialize the stream parameters with demuxer information. */

error = avcodec_parameters_to_context(avctx, stream->codecpar);

if (error < 0) {

avformat_close_input(input_format_context);

avcodec_free_context(&avctx);

return error;

}

/* Open the decoder for the audio stream to use it later. */

if ((error = avcodec_open2(avctx, input_codec, NULL)) < 0) {

fprintf(stderr, "Could not open input codec (error '%s')\n",

av_err2str(error));

avcodec_free_context(&avctx);

avformat_close_input(input_format_context);

return error;

}

/* Set the packet timebase for the decoder. */

avctx->pkt_timebase = stream->time_base;

/* Save the decoder context for easier access later. */

*input_codec_context = avctx;

return  0;

}

/**

* Open an output file and the required encoder.

* Also set some basic encoder parameters.

* Some of these parameters are based on the input file's parameters.

* @param  filename File to be opened

* @param  input_codec_context Codec context of input file

* @param[out] output_format_context Format context of output file

* @param[out] output_codec_context Codec context of output file

* @return Error code (0 if successful)

*/

static  int  open_output_file(const  char *filename,

AVCodecContext *input_codec_context,

AVFormatContext **output_format_context,

AVCodecContext **output_codec_context)

{

AVCodecContext *avctx = NULL;

AVIOContext *output_io_context = NULL;

AVStream *stream = NULL;

const AVCodec *output_codec = NULL;

int error;

/* Open the output file to write to it. */

if ((error = avio_open(&output_io_context, filename,

AVIO_FLAG_WRITE)) < 0) {

fprintf(stderr, "Could not open output file '%s' (error '%s')\n",

filename, av_err2str(error));

return error;

}

/* Create a new format context for the output container format. */

if (!(*output_format_context = avformat_alloc_context())) {

fprintf(stderr, "Could not allocate output format context\n");

return  AVERROR(ENOMEM);

}

/* Associate the output file (pointer) with the container format context. */

(*output_format_context)->pb = output_io_context;

/* Guess the desired container format based on the file extension. */

if (!((*output_format_context)->oformat = av_guess_format(NULL, filename,

NULL))) {

fprintf(stderr, "Could not find output file format\n");

goto cleanup;

}

if (!((*output_format_context)->url = av_strdup(filename))) {

fprintf(stderr, "Could not allocate url.\n");

error = AVERROR(ENOMEM);

goto cleanup;

}

/* Find the encoder to be used by its name. */

if (!(output_codec = avcodec_find_encoder(AV_CODEC_ID_AAC))) {

fprintf(stderr, "Could not find an AAC encoder.\n");

goto cleanup;

}

/* Create a new audio stream in the output file container. */

if (!(stream = avformat_new_stream(*output_format_context, NULL))) {

fprintf(stderr, "Could not create new stream\n");

error = AVERROR(ENOMEM);

goto cleanup;

}

avctx = avcodec_alloc_context3(output_codec);

if (!avctx) {

fprintf(stderr, "Could not allocate an encoding context\n");

error = AVERROR(ENOMEM);

goto cleanup;

}

/* Set the basic encoder parameters.

* The input file's sample rate is used to avoid a sample rate conversion. */

av_channel_layout_default(&avctx->ch_layout, OUTPUT_CHANNELS);

avctx->sample_rate = input_codec_context->sample_rate;

avctx->sample_fmt = output_codec->sample_fmts[0];

avctx->bit_rate = OUTPUT_BIT_RATE;

/* Set the sample rate for the container. */

stream->time_base.den = input_codec_context->sample_rate;

stream->time_base.num = 1;

/* Some container formats (like MP4) require global headers to be present.

* Mark the encoder so that it behaves accordingly. */

if ((*output_format_context)->oformat->flags & AVFMT_GLOBALHEADER)

avctx->flags |= AV_CODEC_FLAG_GLOBAL_HEADER;

/* Open the encoder for the audio stream to use it later. */

if ((error = avcodec_open2(avctx, output_codec, NULL)) < 0) {

fprintf(stderr, "Could not open output codec (error '%s')\n",

av_err2str(error));

goto cleanup;

}

error = avcodec_parameters_from_context(stream->codecpar, avctx);

if (error < 0) {

fprintf(stderr, "Could not initialize stream parameters\n");

goto cleanup;

}

/* Save the encoder context for easier access later. */

*output_codec_context = avctx;

return  0;

cleanup:

avcodec_free_context(&avctx);

avio_closep(&(*output_format_context)->pb);

avformat_free_context(*output_format_context);

*output_format_context = NULL;

return error < 0 ? error : AVERROR_EXIT;

}

/**

* Initialize one data packet for reading or writing.

* @param[out] packet Packet to be initialized

* @return Error code (0 if successful)

*/

static  int  init_packet(AVPacket **packet)

{

if (!(*packet = av_packet_alloc())) {

fprintf(stderr, "Could not allocate packet\n");

return  AVERROR(ENOMEM);

}

return  0;

}

/**

* Initialize one audio frame for reading from the input file.

* @param[out] frame Frame to be initialized

* @return Error code (0 if successful)

*/

static  int  init_input_frame(AVFrame **frame)

{

if (!(*frame = av_frame_alloc())) {

fprintf(stderr, "Could not allocate input frame\n");

return  AVERROR(ENOMEM);

}

return  0;

}

/**

* Initialize the audio resampler based on the input and output codec settings.

* If the input and output sample formats differ, a conversion is required

* libswresample takes care of this, but requires initialization.

* @param  input_codec_context Codec context of the input file

* @param  output_codec_context Codec context of the output file

* @param[out] resample_context Resample context for the required conversion

* @return Error code (0 if successful)

*/

static  int  init_resampler(AVCodecContext *input_codec_context,

AVCodecContext *output_codec_context,

SwrContext **resample_context)

{

int error;

/*

* Create a resampler context for the conversion.

* Set the conversion parameters.

*/

error = swr_alloc_set_opts2(resample_context,

&output_codec_context->ch_layout,

output_codec_context->sample_fmt,

output_codec_context->sample_rate,

&input_codec_context->ch_layout,

input_codec_context->sample_fmt,

input_codec_context->sample_rate,

0, NULL);

if (error < 0) {

fprintf(stderr, "Could not allocate resample context\n");

return error;

}

/*

* Perform a sanity check so that the number of converted samples is

* not greater than the number of samples to be converted.

* If the sample rates differ, this case has to be handled differently

*/

av_assert0(output_codec_context->sample_rate == input_codec_context->sample_rate);

/* Open the resampler with the specified parameters. */

if ((error = swr_init(*resample_context)) < 0) {

fprintf(stderr, "Could not open resample context\n");

swr_free(resample_context);

return error;

}

return  0;

}

/**

* Initialize a FIFO buffer for the audio samples to be encoded.

* @param[out] fifo Sample buffer

* @param  output_codec_context Codec context of the output file

* @return Error code (0 if successful)

*/

static  int  init_fifo(AVAudioFifo **fifo, AVCodecContext *output_codec_context)

{

/* Create the FIFO buffer based on the specified output sample format. */

if (!(*fifo = av_audio_fifo_alloc(output_codec_context->sample_fmt,

output_codec_context->ch_layout.nb_channels, 1))) {

fprintf(stderr, "Could not allocate FIFO\n");

return  AVERROR(ENOMEM);

}

return  0;

}

/**

* Write the header of the output file container.

* @param  output_format_context Format context of the output file

* @return Error code (0 if successful)

*/

static  int  write_output_file_header(AVFormatContext *output_format_context)

{

int error;

if ((error = avformat_write_header(output_format_context, NULL)) < 0) {

fprintf(stderr, "Could not write output file header (error '%s')\n",

av_err2str(error));

return error;

}

return  0;

}

/**

* Decode one audio frame from the input file.

* @param  frame Audio frame to be decoded

* @param  input_format_context Format context of the input file

* @param  input_codec_context Codec context of the input file

* @param[out] data_present Indicates whether data has been decoded

* @param[out] finished Indicates whether the end of file has

* been reached and all data has been

* decoded. If this flag is false, there

* is more data to be decoded, i.e., this

* function has to be called again.

* @return Error code (0 if successful)

*/

static  int  decode_audio_frame(AVFrame *frame,

AVFormatContext *input_format_context,

AVCodecContext *input_codec_context,

int *data_present, int *finished)

{

/* Packet used for temporary storage. */

AVPacket *input_packet;

int error;

error = init_packet(&input_packet);

if (error < 0)

return error;

*data_present = 0;

*finished = 0;

/* Read one audio frame from the input file into a temporary packet. */

if ((error = av_read_frame(input_format_context, input_packet)) < 0) {

/* If we are at the end of the file, flush the decoder below. */

if (error == AVERROR_EOF)

*finished = 1;

else {

fprintf(stderr, "Could not read frame (error '%s')\n",

av_err2str(error));

goto cleanup;

}

}

/* Send the audio frame stored in the temporary packet to the decoder.

* The input audio stream decoder is used to do this. */

if ((error = avcodec_send_packet(input_codec_context, input_packet)) < 0) {

fprintf(stderr, "Could not send packet for decoding (error '%s')\n",

av_err2str(error));

goto cleanup;

}

/* Receive one frame from the decoder. */

error = avcodec_receive_frame(input_codec_context, frame);

/* If the decoder asks for more data to be able to decode a frame,

* return indicating that no data is present. */

if (error == AVERROR(EAGAIN)) {

error = 0;

goto cleanup;

/* If the end of the input file is reached, stop decoding. */

} else  if (error == AVERROR_EOF) {

*finished = 1;

error = 0;

goto cleanup;

} else  if (error < 0) {

fprintf(stderr, "Could not decode frame (error '%s')\n",

av_err2str(error));

goto cleanup;

/* Default case: Return decoded data. */

} else {

*data_present = 1;

goto cleanup;

}

cleanup:

av_packet_free(&input_packet);

return error;

}

/**

* Initialize a temporary storage for the specified number of audio samples.

* The conversion requires temporary storage due to the different format.

* The number of audio samples to be allocated is specified in frame_size.

* @param[out] converted_input_samples Array of converted samples. The

* dimensions are reference, channel

* (for multi-channel audio), sample.

* @param  output_codec_context Codec context of the output file

* @param  frame_size Number of samples to be converted in

* each round

* @return Error code (0 if successful)

*/

static  int  init_converted_samples(uint8_t ***converted_input_samples,

AVCodecContext *output_codec_context,

int  frame_size)

{

int error;

/* Allocate as many pointers as there are audio channels.

* Each pointer will point to the audio samples of the corresponding

* channels (although it may be NULL for interleaved formats).

* Allocate memory for the samples of all channels in one consecutive

* block for convenience. */

if ((error = av_samples_alloc_array_and_samples(converted_input_samples, NULL,

output_codec_context->ch_layout.nb_channels,

frame_size,

output_codec_context->sample_fmt, 0)) < 0) {

fprintf(stderr,

"Could not allocate converted input samples (error '%s')\n",

av_err2str(error));

return error;

}

return  0;

}

/**

* Convert the input audio samples into the output sample format.

* The conversion happens on a per-frame basis, the size of which is

* specified by frame_size.

* @param  input_data Samples to be decoded. The dimensions are

* channel (for multi-channel audio), sample.

* @param[out] converted_data Converted samples. The dimensions are channel

* (for multi-channel audio), sample.

* @param  frame_size Number of samples to be converted

* @param  resample_context Resample context for the conversion

* @return Error code (0 if successful)

*/

static  int  convert_samples(const  uint8_t **input_data,

uint8_t **converted_data, const  int  frame_size,

SwrContext *resample_context)

{

int error;

/* Convert the samples using the resampler. */

if ((error = swr_convert(resample_context,

converted_data, frame_size,

input_data , frame_size)) < 0) {

fprintf(stderr, "Could not convert input samples (error '%s')\n",

av_err2str(error));

return error;

}

return  0;

}

/**

* Add converted input audio samples to the FIFO buffer for later processing.

* @param  fifo Buffer to add the samples to

* @param  converted_input_samples Samples to be added. The dimensions are channel

* (for multi-channel audio), sample.

* @param  frame_size Number of samples to be converted

* @return Error code (0 if successful)

*/

static  int  add_samples_to_fifo(AVAudioFifo *fifo,

uint8_t **converted_input_samples,

const  int  frame_size)

{

int error;

/* Make the FIFO as large as it needs to be to hold both,

* the old and the new samples. */

if ((error = av_audio_fifo_realloc(fifo, av_audio_fifo_size(fifo) + frame_size)) < 0) {

fprintf(stderr, "Could not reallocate FIFO\n");

return error;

}

/* Store the new samples in the FIFO buffer. */

if (av_audio_fifo_write(fifo, (void **)converted_input_samples,

frame_size) < frame_size) {

fprintf(stderr, "Could not write data to FIFO\n");

return AVERROR_EXIT;

}

return  0;

}

/**

* Read one audio frame from the input file, decode, convert and store

* it in the FIFO buffer.

* @param  fifo Buffer used for temporary storage

* @param  input_format_context Format context of the input file

* @param  input_codec_context Codec context of the input file

* @param  output_codec_context Codec context of the output file

* @param  resampler_context Resample context for the conversion

* @param[out] finished Indicates whether the end of file has

* been reached and all data has been

* decoded. If this flag is false,

* there is more data to be decoded,

* i.e., this function has to be called

* again.

* @return Error code (0 if successful)

*/

static  int  read_decode_convert_and_store(AVAudioFifo *fifo,

AVFormatContext *input_format_context,

AVCodecContext *input_codec_context,

AVCodecContext *output_codec_context,

SwrContext *resampler_context,

int *finished)

{

/* Temporary storage of the input samples of the frame read from the file. */

AVFrame *input_frame = NULL;

/* Temporary storage for the converted input samples. */

uint8_t **converted_input_samples = NULL;

int data_present;

int ret = AVERROR_EXIT;

/* Initialize temporary storage for one input frame. */

if (init_input_frame(&input_frame))

goto cleanup;

/* Decode one frame worth of audio samples. */

if (decode_audio_frame(input_frame, input_format_context,

input_codec_context, &data_present, finished))

goto cleanup;

/* If we are at the end of the file and there are no more samples

* in the decoder which are delayed, we are actually finished.

* This must not be treated as an error. */

if (*finished) {

ret = 0;

goto cleanup;

}

/* If there is decoded data, convert and store it. */

if (data_present) {

/* Initialize the temporary storage for the converted input samples. */

if (init_converted_samples(&converted_input_samples, output_codec_context,

input_frame->nb_samples))

goto cleanup;

/* Convert the input samples to the desired output sample format.

* This requires a temporary storage provided by converted_input_samples. */

if (convert_samples((const  uint8_t**)input_frame->extended_data, converted_input_samples,

input_frame->nb_samples, resampler_context))

goto cleanup;

/* Add the converted input samples to the FIFO buffer for later processing. */

if (add_samples_to_fifo(fifo, converted_input_samples,

input_frame->nb_samples))

goto cleanup;

ret = 0;

}

ret = 0;

cleanup:

if (converted_input_samples)

av_freep(&converted_input_samples[0]);

av_freep(&converted_input_samples);

av_frame_free(&input_frame);

return ret;

}

/**

* Initialize one input frame for writing to the output file.

* The frame will be exactly frame_size samples large.

* @param[out] frame Frame to be initialized

* @param  output_codec_context Codec context of the output file

* @param  frame_size Size of the frame

* @return Error code (0 if successful)

*/

static  int  init_output_frame(AVFrame **frame,

AVCodecContext *output_codec_context,

int  frame_size)

{

int error;

/* Create a new frame to store the audio samples. */

if (!(*frame = av_frame_alloc())) {

fprintf(stderr, "Could not allocate output frame\n");

return AVERROR_EXIT;

}

/* Set the frame's parameters, especially its size and format.

* av_frame_get_buffer needs this to allocate memory for the

* audio samples of the frame.

* Default channel layouts based on the number of channels

* are assumed for simplicity. */

(*frame)->nb_samples = frame_size;

av_channel_layout_copy(&(*frame)->ch_layout, &output_codec_context->ch_layout);

(*frame)->format = output_codec_context->sample_fmt;

(*frame)->sample_rate = output_codec_context->sample_rate;

/* Allocate the samples of the created frame. This call will make

* sure that the audio frame can hold as many samples as specified. */

if ((error = av_frame_get_buffer(*frame, 0)) < 0) {

fprintf(stderr, "Could not allocate output frame samples (error '%s')\n",

av_err2str(error));

av_frame_free(frame);

return error;

}

return  0;

}

/* Global timestamp for the audio frames. */

static  int64_t pts = 0;

/**

* Encode one frame worth of audio to the output file.

* @param  frame Samples to be encoded

* @param  output_format_context Format context of the output file

* @param  output_codec_context Codec context of the output file

* @param[out] data_present Indicates whether data has been

* encoded

* @return Error code (0 if successful)

*/

static  int  encode_audio_frame(AVFrame *frame,

AVFormatContext *output_format_context,

AVCodecContext *output_codec_context,

int *data_present)

{

/* Packet used for temporary storage. */

AVPacket *output_packet;

int error;

error = init_packet(&output_packet);

if (error < 0)

return error;

/* Set a timestamp based on the sample rate for the container. */

if (frame) {

frame->pts = pts;

pts += frame->nb_samples;

}

*data_present = 0;

/* Send the audio frame stored in the temporary packet to the encoder.

* The output audio stream encoder is used to do this. */

error = avcodec_send_frame(output_codec_context, frame);

/* Check for errors, but proceed with fetching encoded samples if the

* encoder signals that it has nothing more to encode. */

if (error < 0 && error != AVERROR_EOF) {

fprintf(stderr, "Could not send packet for encoding (error '%s')\n",

av_err2str(error));

goto cleanup;

}

/* Receive one encoded frame from the encoder. */

error = avcodec_receive_packet(output_codec_context, output_packet);

/* If the encoder asks for more data to be able to provide an

* encoded frame, return indicating that no data is present. */

if (error == AVERROR(EAGAIN)) {

error = 0;

goto cleanup;

/* If the last frame has been encoded, stop encoding. */

} else  if (error == AVERROR_EOF) {

error = 0;

goto cleanup;

} else  if (error < 0) {

fprintf(stderr, "Could not encode frame (error '%s')\n",

av_err2str(error));

goto cleanup;

/* Default case: Return encoded data. */

} else {

*data_present = 1;

}

/* Write one audio frame from the temporary packet to the output file. */

if (*data_present &&

(error = av_write_frame(output_format_context, output_packet)) < 0) {

fprintf(stderr, "Could not write frame (error '%s')\n",

av_err2str(error));

goto cleanup;

}

cleanup:

av_packet_free(&output_packet);

return error;

}

/**

* Load one audio frame from the FIFO buffer, encode and write it to the

* output file.

* @param  fifo Buffer used for temporary storage

* @param  output_format_context Format context of the output file

* @param  output_codec_context Codec context of the output file

* @return Error code (0 if successful)

*/

static  int  load_encode_and_write(AVAudioFifo *fifo,

AVFormatContext *output_format_context,

AVCodecContext *output_codec_context)

{

/* Temporary storage of the output samples of the frame written to the file. */

AVFrame *output_frame;

/* Use the maximum number of possible samples per frame.

* If there is less than the maximum possible frame size in the FIFO

* buffer use this number. Otherwise, use the maximum possible frame size. */

const  int frame_size = FFMIN(av_audio_fifo_size(fifo),

output_codec_context->frame_size);

int data_written;

/* Initialize temporary storage for one output frame. */

if (init_output_frame(&output_frame, output_codec_context, frame_size))

return AVERROR_EXIT;

/* Read as many samples from the FIFO buffer as required to fill the frame.

* The samples are stored in the frame temporarily. */

if (av_audio_fifo_read(fifo, (void **)output_frame->data, frame_size) < frame_size) {

fprintf(stderr, "Could not read data from FIFO\n");

av_frame_free(&output_frame);

return AVERROR_EXIT;

}

/* Encode one frame worth of audio samples. */

if (encode_audio_frame(output_frame, output_format_context,

output_codec_context, &data_written)) {

av_frame_free(&output_frame);

return AVERROR_EXIT;

}

av_frame_free(&output_frame);

return  0;

}

/**

* Write the trailer of the output file container.

* @param  output_format_context Format context of the output file

* @return Error code (0 if successful)

*/

static  int  write_output_file_trailer(AVFormatContext *output_format_context)

{

int error;

if ((error = av_write_trailer(output_format_context)) < 0) {

fprintf(stderr, "Could not write output file trailer (error '%s')\n",

av_err2str(error));

return error;

}

return  0;

}

int  main(int  argc, char **argv)

{

AVFormatContext *input_format_context = NULL, *output_format_context = NULL;

AVCodecContext *input_codec_context = NULL, *output_codec_context = NULL;

SwrContext *resample_context = NULL;

AVAudioFifo *fifo = NULL;

int ret = AVERROR_EXIT;

if (argc != 3) {

fprintf(stderr, "Usage: %s <input file> <output file>\n", argv[0]);

exit(1);

}

/* Open the input file for reading. */

if (open_input_file(argv[1], &input_format_context,

&input_codec_context))

goto cleanup;

/* Open the output file for writing. */

if (open_output_file(argv[2], input_codec_context,

&output_format_context, &output_codec_context))

goto cleanup;

/* Initialize the resampler to be able to convert audio sample formats. */

if (init_resampler(input_codec_context, output_codec_context,

&resample_context))

goto cleanup;

/* Initialize the FIFO buffer to store audio samples to be encoded. */

if (init_fifo(&fifo, output_codec_context))

goto cleanup;

/* Write the header of the output file container. */

if (write_output_file_header(output_format_context))

goto cleanup;

/* Loop as long as we have input samples to read or output samples

* to write; abort as soon as we have neither. */

while (1) {

/* Use the encoder's desired frame size for processing. */

const  int output_frame_size = output_codec_context->frame_size;

int finished = 0;

/* Make sure that there is one frame worth of samples in the FIFO

* buffer so that the encoder can do its work.

* Since the decoder's and the encoder's frame size may differ, we

* need to FIFO buffer to store as many frames worth of input samples

* that they make up at least one frame worth of output samples. */

while (av_audio_fifo_size(fifo) < output_frame_size) {

/* Decode one frame worth of audio samples, convert it to the

* output sample format and put it into the FIFO buffer. */

if (read_decode_convert_and_store(fifo, input_format_context,

input_codec_context,

output_codec_context,

resample_context, &finished))

goto cleanup;

/* If we are at the end of the input file, we continue

* encoding the remaining audio samples to the output file. */

if (finished)

break;

}

/* If we have enough samples for the encoder, we encode them.

* At the end of the file, we pass the remaining samples to

* the encoder. */

while (av_audio_fifo_size(fifo) >= output_frame_size ||

(finished && av_audio_fifo_size(fifo) > 0))

/* Take one frame worth of audio samples from the FIFO buffer,

* encode it and write it to the output file. */

if (load_encode_and_write(fifo, output_format_context,

output_codec_context))

goto cleanup;

/* If we are at the end of the input file and have encoded

* all remaining samples, we can exit this loop and finish. */

if (finished) {

int data_written;

/* Flush the encoder as it may have delayed frames. */

do {

if (encode_audio_frame(NULL, output_format_context,

output_codec_context, &data_written))

goto cleanup;

} while (data_written);

break;

}

}

/* Write the trailer of the output file container. */

if (write_output_file_trailer(output_format_context))

goto cleanup;

ret = 0;

cleanup:

if (fifo)

av_audio_fifo_free(fifo);

swr_free(&resample_context);

if (output_codec_context)

avcodec_free_context(&output_codec_context);

if (output_format_context) {

avio_closep(&output_format_context->pb);

avformat_free_context(output_format_context);

}

if (input_codec_context)

avcodec_free_context(&input_codec_context);

if (input_format_context)

avformat_close_input(&input_format_context);

return ret;

}
```
## makefile
```
audiotranscode: audiotranscode.c

gcc -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib $^ -o $@ -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lz -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm -lvorbis -lvpx -lfdk-aac -lavfilter -lswscale -lpostproc -lass -lfreetype -lavutil -lavcodec
```
# Transcode Video
```
// Video transcoding code
// Code from https://github.com/leandromoreira/ffmpeg-libav-tutorial rehashed and rewritten for my understanding

#include  <libavcodec/avcodec.h>

#include  <libavformat/avformat.h>

#include  <libavutil/timestamp.h>

#include  <stdio.h>

#include  <stdarg.h>

#include  <stdlib.h>

#include  <libavutil/opt.h>

#include  <string.h>

#include  <inttypes.h>

  

void  logger(char *msg){

printf("%s\n",msg);

}

  
  

int  encode_video(int  inputVideoStreamIndex,AVStream* inputVideoStream,AVFormatContext* outputFormatContext,AVStream* outputVideoStream,AVCodecContext* outputVideoCodecContext,AVFrame *inputFrame){

if (inputFrame) inputFrame->pict_type = AV_PICTURE_TYPE_NONE;

  

AVPacket *outputPacket = av_packet_alloc();

if (!outputPacket) {

logger("could not allocate memory for output packet");

return -1;

}

// encode decoded frame to output packet with output codec

int response = avcodec_send_frame(outputVideoCodecContext, inputFrame);

while (response >= 0) {

response = avcodec_receive_packet(outputVideoCodecContext, outputPacket);

if (response == AVERROR(EAGAIN) || response == AVERROR_EOF) {

break;

} else  if (response < 0) {

logger("Error while receiving packet from encoder");//: %s", av_err2str(response));

return -1;

}

  

outputPacket->stream_index = inputVideoStreamIndex;

outputPacket->duration = outputVideoStream->time_base.den / outputVideoStream->time_base.num / inputVideoStream->avg_frame_rate.num * inputVideoStream->avg_frame_rate.den;

  

av_packet_rescale_ts(outputPacket, inputVideoStream->time_base, outputVideoStream->time_base);

response = av_interleaved_write_frame(outputFormatContext, outputPacket);

if (response != 0) {

logger("Error %d while receiving packet from decoder");//: %s", response, av_err2str(response));

return -1;

}

}// while receive_packet

av_packet_unref(outputPacket);

av_packet_free(&outputPacket);

return  0;

}

  

int  main(int  argc, char *argv[])

{

char* inputFile = "input.mov";

char* outputFile = "output.mp4";

char *videoOuputCodecName = "libx264";

  

char *videoOutputCodecPrivateKey = "x264-params";

char *videoOutputCodecPrivateValue = "keyint=60:min-keyint=60:scenecut=0:force-cfr=1";

char *muxerOptionsKey = NULL;

char *muxerOptionaValue = NULL;

int transcodeVideo = 1;

int operationResult = 0;

int inputVideoStreamIndex = 0;

  
  

AVFormatContext *inputFormatContext = NULL, *outputFormatContext = NULL;

AVCodecContext *inputCodecContext = NULL, *outputCodecContext = NULL;

AVStream *inputVideoStream, *outputVideoStream;

  

// av_log_set_level(AV_LOG_QUIET);// or we get debug messages from libx264

  

/* Open the input file for reading. */

// **format context, filename,format, options dict decoder priv options

operationResult= avformat_open_input(&inputFormatContext,inputFile,NULL,NULL);

if(operationResult < 0){

logger("Error opening input");

return -1;

}

  
  

/* Get information on the input file (number of streams etc.). */

if ((operationResult = avformat_find_stream_info(inputFormatContext, NULL)) < 0) {

logger("Could not open find stream info");// (error '%s')\ av_err2str(error));

goto cleanup;

}

  

/* Make sure that there is only one stream in the input file. */

if (inputFormatContext->nb_streams > 1) {

logger("Expected one audio input stream, but found more");//%d\n",

// (*input_format_context)->nb_streams);

goto cleanup;

}

// TODO: Loop through streams and extract index based on if (inputFormatContext->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {

inputVideoStream = inputFormatContext->streams[0];// video stream - only 1 stream

inputVideoStreamIndex = 0;

  

// Create codec and codec context to be used for decoding input

AVCodec* inputVideoCodec = avcodec_find_decoder(inputVideoStream->codecpar->codec_id);

if(!inputVideoCodec){

printf("Cant find codec for encoder codec id %d\n",inputVideoStream->codecpar->codec_id);

goto cleanup;

}

// create a codec context for input (decoder)

AVCodecContext* inputVideoCodecContext = avcodec_alloc_context3(inputVideoCodec);

if (!inputVideoCodecContext) {

logger("failed to alloc memory for inputVideoCodecContext");

goto cleanup;

}

// copy codec params from input stream to codec

if (operationResult = avcodec_parameters_to_context(inputVideoCodecContext, inputVideoStream->codecpar) < 0) {

logger("failed to fill inputVideoCodecContext ");

goto cleanup;

}

// Avcodeccontext, avcodec,options-codec params

if (operationResult=avcodec_open2(inputVideoCodecContext,inputVideoCodec, NULL) < 0) {

logger("failed to open inputVideoCodecContext");

goto cleanup;

}

  
  

// Create encoder with context for output

// *format context, AVOutputformat, formatname, filename

avformat_alloc_output_context2(&outputFormatContext, NULL, NULL, outputFile);

if (!outputFormatContext) {

logger("could not allocate memory for output format");

goto cleanup;

}

  

AVCodecContext* outputVideoCodecContext = NULL;

if(transcodeVideo){

// formatcontext,stream, frame

AVRational inputFramerate = av_guess_frame_rate(inputFormatContext,inputVideoStream, NULL);

// printf("input frame rate %d %d\n",inputFramerate.num, inputFramerate.den);

  

// create output stream

outputVideoStream = avformat_new_stream(outputFormatContext, NULL);

AVCodec* outputVideoCodec = avcodec_find_encoder_by_name(videoOuputCodecName);

if(!outputVideoCodec){

printf("Cant find codec by name %s\n",videoOuputCodecName);

goto cleanup;

}

  

outputVideoCodecContext = avcodec_alloc_context3(outputVideoCodec);

if(!outputVideoCodecContext){

logger("Cant allocate memory for outputVideoCodecContext");

goto cleanup;

}

// set output codec options (encoder)

av_opt_set(outputVideoCodecContext->priv_data, "preset", "ultrafast", 0);

if (videoOutputCodecPrivateKey && videoOutputCodecPrivateValue)

av_opt_set(outputVideoCodecContext->priv_data, videoOutputCodecPrivateKey, videoOutputCodecPrivateValue, 0);

  

// printf("wxh = %d x %d\n",inputVideoCodecContext->width,inputVideoCodecContext->height);

  

// Set outputcodec params from input

outputVideoCodecContext->height = inputVideoCodecContext->height;

outputVideoCodecContext->width = inputVideoCodecContext->width;

outputVideoCodecContext->sample_aspect_ratio = inputVideoCodecContext->sample_aspect_ratio;

if (outputVideoCodec->pix_fmts)

outputVideoCodecContext->pix_fmt = outputVideoCodec->pix_fmts[0];

else

outputVideoCodecContext->pix_fmt = inputVideoCodecContext->pix_fmt;

outputVideoCodecContext->bit_rate = 2 * 1000 * 1000;

outputVideoCodecContext->rc_buffer_size = 4 * 1000 * 1000;

outputVideoCodecContext->rc_max_rate = 2 * 1000 * 1000;

outputVideoCodecContext->rc_min_rate = 2.5 * 1000 * 1000;

  

outputVideoCodecContext->time_base = av_inv_q(inputFramerate);

outputVideoStream->time_base = outputVideoCodecContext->time_base;

  

if (avcodec_open2(outputVideoCodecContext, outputVideoCodec, NULL) < 0) {

logger("could not open the codec outputVideoCodec");

goto cleanup;

}

avcodec_parameters_from_context(outputVideoStream->codecpar, outputVideoCodecContext);

  

}// encode video stream

else {

//copy video stream

/* int prepare_copy(AVFormatContext *avfc, AVStream **avs, AVCodecParameters *decoder_par) {

*avs = avformat_new_stream(avfc, NULL);

avcodec_parameters_copy((*avs)->codecpar, decoder_par);

return 0;

} */

}// copy video stream - dont encode

  

if (outputFormatContext->oformat->flags & AVFMT_GLOBALHEADER)

outputFormatContext->flags |= AV_CODEC_FLAG_GLOBAL_HEADER;

  

if (!(outputFormatContext->oformat->flags & AVFMT_NOFILE)) {

if (avio_open(&outputFormatContext->pb, outputFile, AVIO_FLAG_WRITE) < 0) {

logger("could not open the output file");

goto cleanup;

}

}

  

AVDictionary* muxerOptions = NULL;

if (muxerOptionsKey && muxerOptionaValue) {

av_dict_set(&muxerOptions,muxerOptionsKey,muxerOptionaValue, 0);

}

// Write header to output file before writing frame/packet data

if (avformat_write_header(outputFormatContext, &muxerOptions) < 0) {

logger("an error occurred writing header to output file");

goto cleanup;

}

  

AVFrame *inputFrame = av_frame_alloc();

if(!inputFrame){

logger("Failed to allocate memory for input frame");

goto cleanup;

}

AVPacket *inputPacket = av_packet_alloc();

if (!inputPacket) {

logger("failed to allocated memory for input packet");

goto cleanup;

}

  

while (av_read_frame(inputFormatContext, inputPacket) >= 0)

{

if (inputFormatContext->streams[inputPacket->stream_index]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {

if(transcodeVideo){

operationResult = avcodec_send_packet(inputVideoCodecContext, inputPacket);

if(operationResult < 0){

logger("Error while sending packet to decoder");

goto cleanup;

}

while(operationResult >=0){

operationResult = avcodec_receive_frame(inputVideoCodecContext, inputFrame);

if (operationResult == AVERROR(EAGAIN) || operationResult == AVERROR_EOF) {

break;

}

else  if (operationResult < 0) {

logger("Error while receiving frame from decoder");//, av_err2str(response));

goto cleanup;

}

if (operationResult >= 0) {

if(operationResult=encode_video(inputVideoStreamIndex,inputVideoStream,outputFormatContext,outputVideoStream,outputVideoCodecContext,inputFrame)) break;

}// if avcodec_receive_frame

av_frame_unref(inputFrame);

}// while operationResult > = 0

av_packet_unref(inputPacket);

}// if transcodeVideo

else{

  

}

}

}// while av_read_frame

  

// TODO: should I also flush the audio encoder?

if(operationResult=encode_video(inputVideoStreamIndex,inputVideoStream,outputFormatContext,outputVideoStream,outputVideoCodecContext,NULL)){

logger("Error trying to flush video enoder");

goto cleanup;

}

  

av_write_trailer(outputFormatContext);

  
  

goto cleanup;

cleanup:

if(inputFormatContext)

avformat_close_input(&inputFormatContext);

if(muxerOptions)

av_dict_free(&muxerOptions);

if(inputFrame)

av_frame_free(&inputFrame);

if(inputPacket)

av_packet_free(&inputPacket);

  

if(inputFormatContext){

avformat_close_input(&inputFormatContext);

avformat_free_context(inputFormatContext);

inputFormatContext = NULL;

}

if(outputFormatContext){

avformat_free_context(outputFormatContext);

outputFormatContext = NULL;

}

if(inputVideoCodecContext){

avcodec_free_context(&inputVideoCodecContext);

inputVideoCodecContext = NULL;

}

return operationResult <0 ? operationResult: AVERROR_EXIT;

}//main
```
## makefile
```

videotranscode: videotranscode.c

gcc -g -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib $^ -o $@ -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lz -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm -lvorbis -lvpx -lfdk-aac -lavfilter -lswscale -lpostproc -lass -lfreetype -lavutil -lavcodec -lvorbisenc -lx265
```

# Transcode and mux video and audio
```
// Video transcoding code
// Code from https://github.com/leandromoreira/ffmpeg-libav-tutorial rehashed and rewritten for my understanding

#include  <libavcodec/avcodec.h>

#include  <libavformat/avformat.h>

#include  <libavutil/timestamp.h>

#include  <stdio.h>

#include  <stdarg.h>

#include  <stdlib.h>

#include  <libavutil/opt.h>

#include  <string.h>

#include  <inttypes.h>

  

void  logger(char *msg){

printf("%s\n",msg);

}

  
  

int  encode_video(int  inputVideoStreamIndex,AVStream* inputVideoStream,AVFormatContext* outputFormatContext,AVStream* outputVideoStream,AVCodecContext* outputVideoCodecContext,AVFrame *inputFrame){

if (inputFrame) inputFrame->pict_type = AV_PICTURE_TYPE_NONE;

  

AVPacket *outputPacket = av_packet_alloc();

if (!outputPacket) {

logger("could not allocate memory for output packet");

return -1;

}

// encode decoded frame to output packet with output codec

int response = avcodec_send_frame(outputVideoCodecContext, inputFrame);

while (response >= 0) {

response = avcodec_receive_packet(outputVideoCodecContext, outputPacket);

if (response == AVERROR(EAGAIN) || response == AVERROR_EOF) {

break;

} else  if (response < 0) {

logger("Error while receiving packet from encoder");//: %s", av_err2str(response));

return -1;

}

  

outputPacket->stream_index = inputVideoStreamIndex;

outputPacket->duration = outputVideoStream->time_base.den / outputVideoStream->time_base.num / inputVideoStream->avg_frame_rate.num * inputVideoStream->avg_frame_rate.den;

  

av_packet_rescale_ts(outputPacket, inputVideoStream->time_base, outputVideoStream->time_base);

response = av_interleaved_write_frame(outputFormatContext, outputPacket);

if (response != 0) {

logger("Error %d while receiving packet from decoder");//: %s", response, av_err2str(response));

return -1;

}

}// while receive_packet

av_packet_unref(outputPacket);

av_packet_free(&outputPacket);

return  0;

}

  

int  main(int  argc, char *argv[])

{

char* inputFile = "input.mov";

char* outputFile = "output.mp4";

char *videoOuputCodecName = "libx264";

  

char *videoOutputCodecPrivateKey = "x264-params";

char *videoOutputCodecPrivateValue = "keyint=60:min-keyint=60:scenecut=0:force-cfr=1";

char *muxerOptionsKey = NULL;

char *muxerOptionaValue = NULL;

int transcodeVideo = 1;

int operationResult = 0;

int inputVideoStreamIndex = 0;

  
  

AVFormatContext *inputFormatContext = NULL, *outputFormatContext = NULL;

AVCodecContext *inputCodecContext = NULL, *outputCodecContext = NULL;

AVStream *inputVideoStream, *outputVideoStream;

  

// av_log_set_level(AV_LOG_QUIET);// or we get debug messages from libx264

  

/* Open the input file for reading. */

// **format context, filename,format, options dict decoder priv options

operationResult= avformat_open_input(&inputFormatContext,inputFile,NULL,NULL);

if(operationResult < 0){

logger("Error opening input");

return -1;

}

  
  

/* Get information on the input file (number of streams etc.). */

if ((operationResult = avformat_find_stream_info(inputFormatContext, NULL)) < 0) {

logger("Could not open find stream info");// (error '%s')\ av_err2str(error));

goto cleanup;

}

  

/* Make sure that there is only one stream in the input file. */

if (inputFormatContext->nb_streams > 1) {

logger("Expected one audio input stream, but found more");//%d\n",

// (*input_format_context)->nb_streams);

goto cleanup;

}

// TODO: Loop through streams and extract index based on if (inputFormatContext->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {

inputVideoStream = inputFormatContext->streams[0];// video stream - only 1 stream

inputVideoStreamIndex = 0;

  

// Create codec and codec context to be used for decoding input

AVCodec* inputVideoCodec = avcodec_find_decoder(inputVideoStream->codecpar->codec_id);

if(!inputVideoCodec){

printf("Cant find codec for encoder codec id %d\n",inputVideoStream->codecpar->codec_id);

goto cleanup;

}

// create a codec context for input (decoder)

AVCodecContext* inputVideoCodecContext = avcodec_alloc_context3(inputVideoCodec);

if (!inputVideoCodecContext) {

logger("failed to alloc memory for inputVideoCodecContext");

goto cleanup;

}

// copy codec params from input stream to codec

if (operationResult = avcodec_parameters_to_context(inputVideoCodecContext, inputVideoStream->codecpar) < 0) {

logger("failed to fill inputVideoCodecContext ");

goto cleanup;

}

// Avcodeccontext, avcodec,options-codec params

if (operationResult=avcodec_open2(inputVideoCodecContext,inputVideoCodec, NULL) < 0) {

logger("failed to open inputVideoCodecContext");

goto cleanup;

}

  
  

// Create encoder with context for output

// *format context, AVOutputformat, formatname, filename

avformat_alloc_output_context2(&outputFormatContext, NULL, NULL, outputFile);

if (!outputFormatContext) {

logger("could not allocate memory for output format");

goto cleanup;

}

  

AVCodecContext* outputVideoCodecContext = NULL;

if(transcodeVideo){

// formatcontext,stream, frame

AVRational inputFramerate = av_guess_frame_rate(inputFormatContext,inputVideoStream, NULL);

// printf("input frame rate %d %d\n",inputFramerate.num, inputFramerate.den);

  

// create output stream

outputVideoStream = avformat_new_stream(outputFormatContext, NULL);

AVCodec* outputVideoCodec = avcodec_find_encoder_by_name(videoOuputCodecName);

if(!outputVideoCodec){

printf("Cant find codec by name %s\n",videoOuputCodecName);

goto cleanup;

}

  

outputVideoCodecContext = avcodec_alloc_context3(outputVideoCodec);

if(!outputVideoCodecContext){

logger("Cant allocate memory for outputVideoCodecContext");

goto cleanup;

}

// set output codec options (encoder)

av_opt_set(outputVideoCodecContext->priv_data, "preset", "ultrafast", 0);

if (videoOutputCodecPrivateKey && videoOutputCodecPrivateValue)

av_opt_set(outputVideoCodecContext->priv_data, videoOutputCodecPrivateKey, videoOutputCodecPrivateValue, 0);

  

// printf("wxh = %d x %d\n",inputVideoCodecContext->width,inputVideoCodecContext->height);

  

// Set outputcodec params from input

outputVideoCodecContext->height = inputVideoCodecContext->height;

outputVideoCodecContext->width = inputVideoCodecContext->width;

outputVideoCodecContext->sample_aspect_ratio = inputVideoCodecContext->sample_aspect_ratio;

if (outputVideoCodec->pix_fmts)

outputVideoCodecContext->pix_fmt = outputVideoCodec->pix_fmts[0];

else

outputVideoCodecContext->pix_fmt = inputVideoCodecContext->pix_fmt;

outputVideoCodecContext->bit_rate = 2 * 1000 * 1000;

outputVideoCodecContext->rc_buffer_size = 4 * 1000 * 1000;

outputVideoCodecContext->rc_max_rate = 2 * 1000 * 1000;

outputVideoCodecContext->rc_min_rate = 2.5 * 1000 * 1000;

  

outputVideoCodecContext->time_base = av_inv_q(inputFramerate);

outputVideoStream->time_base = outputVideoCodecContext->time_base;

  

if (avcodec_open2(outputVideoCodecContext, outputVideoCodec, NULL) < 0) {

logger("could not open the codec outputVideoCodec");

goto cleanup;

}

avcodec_parameters_from_context(outputVideoStream->codecpar, outputVideoCodecContext);

  

}// encode video stream

else {

//copy video stream

outputVideoStream = avformat_new_stream(outputFormatContext, NULL);

// copy codec params from input codec

avcodec_parameters_copy((outputVideoStream)->codecpar, inputVideoStream->codecpar);

}// copy video stream - dont encode

  

if (outputFormatContext->oformat->flags & AVFMT_GLOBALHEADER)

outputFormatContext->flags |= AV_CODEC_FLAG_GLOBAL_HEADER;

  

if (!(outputFormatContext->oformat->flags & AVFMT_NOFILE)) {

if (avio_open(&outputFormatContext->pb, outputFile, AVIO_FLAG_WRITE) < 0) {

logger("could not open the output file");

goto cleanup;

}

}

  

AVDictionary* muxerOptions = NULL;

if (muxerOptionsKey && muxerOptionaValue) {

av_dict_set(&muxerOptions,muxerOptionsKey,muxerOptionaValue, 0);

}

// Write header to output file before writing frame/packet data

if (avformat_write_header(outputFormatContext, &muxerOptions) < 0) {

logger("an error occurred writing header to output file");

goto cleanup;

}

  

AVFrame *inputFrame = av_frame_alloc();

if(!inputFrame){

logger("Failed to allocate memory for input frame");

goto cleanup;

}

AVPacket *inputPacket = av_packet_alloc();

if (!inputPacket) {

logger("failed to allocated memory for input packet");

goto cleanup;

}

  

while (av_read_frame(inputFormatContext, inputPacket) >= 0)

{

if (inputFormatContext->streams[inputPacket->stream_index]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {

if(transcodeVideo){

operationResult = avcodec_send_packet(inputVideoCodecContext, inputPacket);

if(operationResult < 0){

logger("Error while sending packet to decoder");

goto cleanup;

}

while(operationResult >=0){

operationResult = avcodec_receive_frame(inputVideoCodecContext, inputFrame);

if (operationResult == AVERROR(EAGAIN) || operationResult == AVERROR_EOF) {

break;

}

else  if (operationResult < 0) {

logger("Error while receiving frame from decoder");//, av_err2str(response));

goto cleanup;

}

if (operationResult >= 0) {

if(operationResult=encode_video(inputVideoStreamIndex,inputVideoStream,outputFormatContext,outputVideoStream,outputVideoCodecContext,inputFrame)) break;

}// if avcodec_receive_frame

av_frame_unref(inputFrame);

}// while operationResult > = 0

av_packet_unref(inputPacket);

}// if transcodeVideo

else{

av_packet_rescale_ts(inputPacket, inputVideoStream->time_base, outputVideoStream->time_base);

if (operationResult=av_interleaved_write_frame(outputFormatContext, inputPacket) < 0) {

logger("error while copying stream packet");

goto cleanup;

}

}// no transcodeVideo

}

}// while av_read_frame

  

// TODO: should I also flush the audio encoder?

if(operationResult=encode_video(inputVideoStreamIndex,inputVideoStream,outputFormatContext,outputVideoStream,outputVideoCodecContext,NULL)){

logger("Error trying to flush video enoder");

goto cleanup;

}

  

av_write_trailer(outputFormatContext);

  
  

goto cleanup;

cleanup:

if(inputFormatContext)

avformat_close_input(&inputFormatContext);

if(muxerOptions)

av_dict_free(&muxerOptions);

if(inputFrame)

av_frame_free(&inputFrame);

if(inputPacket)

av_packet_free(&inputPacket);

  

if(inputFormatContext){

avformat_close_input(&inputFormatContext);

avformat_free_context(inputFormatContext);

inputFormatContext = NULL;

}

if(outputFormatContext){

avformat_free_context(outputFormatContext);

outputFormatContext = NULL;

}

if(inputVideoCodecContext){

avcodec_free_context(&inputVideoCodecContext);

inputVideoCodecContext = NULL;

}

return operationResult <0 ? operationResult: AVERROR_EXIT;

}//main
```
## makefile
```
alltranscode: alltranscode.c

gcc -g -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib $^ -o $@ -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lz -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm -lvorbis -lvpx -lfdk-aac -lavfilter -lswscale -lpostproc -lass -lfreetype -lavutil -lavcodec -lvorbisenc -lx265
```

# For Constant frame rate

What you're looking for is fixed gop and fps! to achieve that just set stream  `avg_frame_rate`  and  `tune`  to  `zerolatency`, that's all.
*Source - Stackoverflow*

# Nice to know
-   **tbn**  = the time base in AVStream that has come from the container
-   **tbc**  = the time base in AVCodecContext for the codec used for a particular stream
-   **tbr**  = tbr is guessed from the video stream and is the value users want to see when they look for the video frame rate

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg5MDQyMjI1MywtNjA2OTMyMzI5LDE2OT
cwOTA5MTEsNjY5Mzg4NDksLTQ4OTQzOTcyNCw5MTEzOTIyODAs
LTI3MzI0OTkwNywxMjA4NDcwNjA1LDQ1MDUwOTM2NCwtNTgxNj
ExNDU4LC0xOTUzMDkzODk2LC0xNDQ4MDkwNDY4LC0xODcwNjAw
MjkzLC0xOTU2Mjc2ODQsMTA4ODYyNDkxNCwtOTQ4Njk3NywtMj
AzNTgxODc1LDEwNTc5MzQ2NjUsLTE4Mjg1MTEzOTNdfQ==
-->