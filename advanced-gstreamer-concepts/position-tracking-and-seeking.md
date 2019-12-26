# Position tracking and seeking

지금까지 미디어 처리를 위한 파이프라인의 생성 방법과 해당 파이프라인을 어떻게 동작시킬 수 있는지에 대해 확인해 왔습니다. 대부분의 어플리케이션 개발자라면 이것들 뿐만이 아니라 사용자에게 미디어의 상태에 대한 정보를 제공하는 것에 대해서도 관심이 있을 것입니다. 예를 들어서 미디어 플레이어의 경우 노래의 재생 상태에 대한 상태 바(progress bar)나 전체 stream의 길이를 나타내고 싶어할 것입니다. 트랜스코딩 어플리케이션의 경우라면 현재 작업이 어디까지 수행되었는지에 대한 상태 바를 표시하고 싶어하겠죠. GStreamer는 이 모든 것을 *querying*이라고 알려진 개념을 이용해 built-in 서포트를 제공합니다. Seeking이라는 개념도 Querying과 매우 유사하기 때문에, 해당 개념도 여기에서 함께 다루겠습니다. Seeking은 *events*라는 개념을 이용해 수행됩니다.

## Table of Contents

- [Position tracking and seeking](#position-tracking-and-seeking)
  - [Table of Contents](#table-of-contents)
  - [Querying: getting the position or length of a stream](#querying-getting-the-position-or-length-of-a-stream)
  - [Events: seeking (and more)](#events-seeking-and-more)

## Querying: getting the position or length of a stream

Querying은 상태 추적과 관련된 특정한 stream의 요소를 요청하는 것이라고 정의되고 있습니다. 이는 (가능하다면) stream의 길이나 현재의 위치를 얻어오는 것을 포함하고 있습니다. 이러한 stream의 요소들은 시간, 오디오 샘플, 비디오 프레임 혹은 bytes와 같은 다양한 형태로 표현될 수 있습니다. Querying을 위해 가장 흔하게 사용되는 함수는 `gst_element_query()`이지만, 그 외의 다른 편리한 wrapper 함수들도 존재합니다.(ex. `gst_element_query_position()`이나 `gst_element_query_duration()`) Stream에 대해 직접적으로 querying을 수행할 수 있으며, 해당 결과물은 직접적으로 내부적인 디테일을 표현해 줄 것입니다.

내부적으로 쿼리들은 sink에 전송되며, 어느 한 element가 해당 쿼리를 관리할 수 있을 때까지 뒤로 퍼뜨려집니다. 쿼리의 결과는 다시 함수를 호출한 개체에게 돌아갑니다. 일반적으로 demuxer가 핸들러의 역할을 수행하며, webcam과 같은 live source의 경우에는 해당 source 자체가 그 역할을 수행합니다.

```c
#include <gst/gst.h>

static gboolean
cb_print_position (GstElement *pipeline)
{
    gint64 pos, len;

    if (gst_element_query_position (pipeline, GST_FORMAT_TIME, &pos)
        && gst_element_query_duration (pipeline, GST_FORMAT_TIME, &len)) {
        g_print ("Time: %" GST_TIME_FORMAT " / %" GST_TIME_FORMAT "\r",
        GST_TIME_ARGS (pos), GST_TIME_ARGS (len));
    }

    /* call me again */
    return TRUE;
}

gint
main (gint   argc,
      gchar *argv[])
{
    GstElement *pipeline;

    [..]

    /* run pipeline */
    g_timeout_add (200, (GSourceFunc) cb_print_position, pipeline);
    g_main_loop_run (loop);

    [..]
}
```

## Events: seeking (and more)

Events는 쿼리와 매우 유사한 방식으로 동작합니다. 예를 들어서 "Dispatching"의 경우에는 events와 완벽하게 동일하게 동작하며, 같은 제약사항을 가지고 있습니다. 또한 toplevel의 pipeline로 전송되며(?), 사용자가 알고자 하는 모든 것을 나타내 줄 것이라는 점에서도 비슷합니다. 물론 Event를 이용하여 어플리케이션과 element가 상호작용할 수 있는 많은 방법들이 있지만, 여기에서는 "seeking"만을 보고자 합니다. *Seeking*은 seek-event를 통해 수행됩니다. Seek-event는 playback rate, seek offset format(시간, 오디오 샘플들, 비디오의 frames 혹은 byte값들과 같은 offset들의 유닛들), 옵션적으로 seeking-related flags들의 set들(ex. 내부 버퍼들이 flush되어야 하는지에 대한 여부), seek method(offset이 주어졌는지에 대해 관련된 것을 보여주는), 그리고 seek offset들을 포함합니다. 첫 번째 offset인 cur은 찾아야 할 새로운 포지션을 나타내며, 두 번째 offset인 stop은 선택적이고, stream이 어디서 멈춰야 할 지에 대한 위치를 나타냅니다. 일반적으로 end_method와 end offset을 GST_SEEK_TYPE_NONE 혹은 -1로 표현합니다. seek의 작동 방식은 `gst_element_seek()`에 표현되어 있습니다.

```c
static void
seek_to_time (GstElement *pipeline,
          gint64      time_nanoseconds)
{
  if (!gst_element_seek (pipeline, 1.0, GST_FORMAT_TIME, GST_SEEK_FLAG_FLUSH,
                         GST_SEEK_TYPE_SET, time_nanoseconds,
                         GST_SEEK_TYPE_NONE, GST_CLOCK_TIME_NONE)) {
    g_print ("Seek failed!\n");
  }
}
```

GST_SEEK_FLAG_FLUSH flag가 이용된 seek은 파이프라인이 PAUSED 혹은 PLAYING 상태일 때에만 수행되어야 합니다. "Seek" 이후에 새로운 데이터로 인해 파이프라인이 다시 preroll 될 때까지 파이프라인은 자동으로 preroll 상태가 됩니다.파이프라인이 preroll 된 이후에는 seek이 수행되었던 상태(PAUSED 혹은 PLAYING)로 되돌아갑니다. `gst_element_get_state()`를 이용하거나 bus에서 ASYNC_DONE 메세지가 나타날 때까지 기다리는 방법을 통해 seek이 완료될 때까지 기다릴 수 있습니다.

GST_SEEK_FLAG_FLUSH를 사용하지 않은 "Seek"은 파이프라인이 PLAYING 상태일 때만 수행 가능합니다. PAUSED 상태에서 non-flushing seek을 실행할 경우에는 파이프라인 streaming 쓰레드가 sink에서 blocked 상태가 될 수 있기 때문에 데드락 상태에 걸릴 수 있습니다.

Seek은 `gst_element_seek()`이 리턴될 때 완료되기 때문에, 해당 작업이 즉시 완료되는 것이 아니라는 것을 깨닫는 것은 중요합니다. 연관된 특정 element들에 따라, 실제 검색은 다른 쓰레드(streaming thread)에서 추후에 수행될 수 있고, 이 경우에는 새로운 검색 위치의 버퍼가 sink와 같은 downstream element에 도달할 때까지 짧은 시간이 소요될 수 있습니다.(만약 non-flushing seek일 경우, 더 긴 시간이 소요됩니다.)

slider의 움직임에 대한 직접적인 반응을 위해 짧은 시간차를 두고 다중-seek을 수행하는 것도 가능합니다. 내부적으로는 seek을 수행한 후 파이프라인이 (playing 상태였을 때에만) paused 상태로 전환되고, 위치가 다시 설정되며, demuxer와 decocder가 새로운 위치에서 decode를 다시 수행합니다. 그리고 이 일련의 과정들은 모든 sink들이 다시 데이터를 가질 때까지 계속됩니다. 만약 원래 playing 상태였다면, 다시 playing 상태로 전환될 것입니다. 새로운 위치가 video output에서 즉각적으로 사용 가능하므로, 만약 pipeline이 Playing 상태가 아니더라도 새로운 frame을 볼 수 있을 것입니다.
