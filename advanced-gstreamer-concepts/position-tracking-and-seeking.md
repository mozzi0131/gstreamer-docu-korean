# Position tracking and seeking

지금까지 미디어 처리를 위한 파이프라인의 생성 방법과 해당 파이프라인을 어떻게 동작시킬 수 있는지에 대해 확인해 왔습니다. 대부분의 어플리케이션 개발자라면 이것들 뿐만이 아니라 사용자에게 미디어의 상태에 대한 정보를 제공하는 것에 대해서도 관심이 있을 것입니다. 예를 들어서 미디어 플레이어의 경우 노래의 재생 상태에 대한 상태 바(progress bar)나 전체 stream의 길이를 나타내고 싶어할 것입니다. 트랜스코딩 어플리케이션의 경우라면 현재 작업이 어디까지 수행되었는지에 대한 상태 바를 표시하고 싶어하겠죠. GStreamer는 이 모든 것을 *querying*이라고 알려진 개념을 이용해 built-in 서포트를 제공합니다. Seeking이라는 개념도 Querying과 매우 유사하기 때문에, 해당 개념도 여기에서 함께 다루겠습니다. Seeking은 *events*라는 개념을 이용해 수행됩니다.

## Table of Contents

- [Position tracking and seeking](#position-tracking-and-seeking)
  - [Table of Contents](#table-of-contents)
  - [Querying: getting the position or length of a stream](#querying-getting-the-position-or-length-of-a-stream)
  - [Events: seeking (and more)](#events-seeking-and-more)

## Querying: getting the position or length of a stream

Querying은 상태 추적과 관련된 특정한 stream의 요소를 요청하는 것이라고 정의되고 있습니다. 이는 (가능하다면) stream의 길이나 현재의 위치를 얻어오는 것을 포함하고 있습니다. 이러한 stream의 요소들은 시간, 오디오 샘플, 비디오 프레임 혹은 bytes와 같은 다양한 형태로 표현될 수 있습니다. Querying을 위해 가장 흔하게 사용되는 함수는 `gst_element_query()`이지만, 그 외의 다른 편리한 wrapper 함수들도 존재합니다.(예를 들어 `gst_element_query_position()`이나 `gst_element_query_duration()`) Stream에 대해 직접적으로 querying을 수행할 수 있으며, 해당 결과물은 직접적으로 내부적인 디테일을 표현해 줄 것입니다.

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

Events는 쿼리와 매우 유사한 방식으로 동작합니다. 예를 들어서 "Dispatching"의 경우에는 events와 완벽하게 동일하게 동작하며, 같은 제약사항을 가지고 있습니다. 또한 toplevel의 pipeline로 보내지며, 사용자가 알고자 하는 모든 것을 나타내 줄 것이라는 점에서도 비슷합니다. 물론 Event를 이용하여 어플리케이션과 element가 상호작용할 수 있는 많은 방법들이 있지만, 여기에서는 "seeking"만을 보고자 합니다. *Seeking*은 seek-event를 통해 수행됩니다. Seek-event는 playback rate, seek offset format(시간, 오디오 샘플들, 비디오의 frames 혹은 byte값들과 같은 offset들의 유닛들), seek method(offset이 )