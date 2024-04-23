# FFMPEG Libav Book

Debian Docker instance

docker run -it --entrypoint=bash debian:unstable-slim

Install FFMPEG
https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu

Save container as new image
docker commit abcd345r debian-ffmpeg-libav

Compile and Link command
 gcc -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib -o trim trim.c -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm

https://www3.nd.edu/~zxu2/acms60212-40212/Makefile.pdf

Docker create volume in docker desktop called ffmpeg-code
docker run -v ffmpeg-code:/ffmpeg-code -it --entrypoint=bash debian-ffmpeg-libav



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

  avformat_close_input(&avInputFormatContext);

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
 * starts at 120 seconds n trims 1 min from there
*/
int main(int args,const char* argv[])
{
 cutFile();
}

```

Makefile
```
trim: trim.c
	gcc -I/root/ffmpeg_build/include -L/root/ffmpeg_build/lib -o trim trim.c -lavutil -lavformat -lavcodec -lz -lavutil -lm -lX11 -lvdpau -lva -lgnutls -lz -lx264 -lbz2 -lmp3lame -ldrm -lswresample -lva-x11 -lva-drm
```
