# Pipeline manipulation

이번 챕터는 어플리케이션에서 pipeline을 조작하는 많은 방법들에 대해 다룹니다. 아래 토픽들이 크게 다룰 점들입니다.

- 어떻게 어플리케이션에서 파이프라인으로 data를 넣는지
- 어떻게 pipeline에서 data를 읽어들일지
- 어떻게 pipeline의 속도, 길이, 시작점을 조작할지
- 어떻게 pipeline의 data 처리를 *들을지*
  
이번 챕터의 일부분은 low-level단으로 다뤄지기 때문에, 해당 파트를 이해하려면 프로그래밍 경험과 GStreamer를 잘 이해하고 있어야 합니다.

## Table of Contents

- [Pipeline manipulation](#pipeline-manipulation)
  - [Table of Contents](#table-of-contents)
  - [Using probes](#using-probes)
    - [Data probes](#data-probes)
    - [Play a section of a media file](#play-a-section-of-a-media-file)
  - [Manually adding or removing data from/to a pipeline](#manually-adding-or-removing-data-fromto-a-pipeline)
    - [Inserting data with appsrc](#inserting-data-with-appsrc)
    - [Grabbing data with appsink](#grabbing-data-with-appsink)
  - [Forcing a format](#forcing-a-format)
    - [Changing format in a PLAYING pipeline](#changing-format-in-a-playing-pipeline)
  - [Dynamically changing the pipeline](#dynamically-changing-the-pipeline)
    - [Changing elements in a pipeline](#changing-elements-in-a-pipeline)

## Using probes

Probing은 pad listener에 접근하기 위해서 제일 좋은 방법이라고 여겨집니다. 기술적으로, probe는 `gst_pad_add_probe()`를 이용해서 pad에 붙일 수 있는 callback 그 이상도 이하도 아닙니다. 해당 콜백을 제거하려면 `gst_pad_remove_probe()`를 이용하면 됩니다. 콜백을 붙이고 있는 동안은, probe는 당신에게 pad에서 발생하는 어떤 활동이건 알려줄 것입니다. 따라서 당신은 probe를 추가하면서 당신이 관심을 가지는 어떤 종류의 알림이건 정의해 사용할 수 있습니다.

Probe의 종류들 :

- Buffer가 push되거나 pull되는 경우. 사용자들은 probe를 등록할 때 `GST_PAD_PROBE_TYPE_BUFFER`를 확인하고 싶어할 것입니다. pad가 여러 가지 방식으로 계획될(?) 수 있기 때문인데, 필요한 경우 `GST_PAD_PROBE_TYPE_PUSH` 혹은 `GST_PAD_PROBE_TYPE_PULL` flag를 이용해서 필요한 scheduling mode만 특정할 수도 있습니다. Bufftr를 수정, drop, 확인을 위해 이 probe를 사용할 수도 있습니다. [Data probes](#data-probes)를 참고하세요.
- buffer list는 push될 수 있습니다. probe를 등록할 때 `GST_PAD_PROBE_TYPE_BUFFER_LIST`를 이용하세요.
- event는 pad를 통해 이동합니다. `GST_PAD_PROBE_TYPE_EVENT_DOWNSTREAM`이나 `GST_PAD_PROBE_TYPE_EVENT_UPSTREAM` flag를 통해 event의 방향을 선택하세요. 또한 양방향에 대한 이벤트를 알림받기 위해 `GST_PAD_PROBE_TYPE_EVENT_BOTH`도 지원됩니다. 기본적으로는 flush event는 알림을 발생시키지 않습니다. 따라서 flush event에서의 callback을 받기 위해서는 `GST_PAD_PROBE_TYPE_EVENT_FLUSH`를 통해 직접 알림을 켜 줘야 합니다. 이벤트들은 언제나 push mode에서만 알림이 옵니다. 이 타입의 probe를 이벤트를 확인, 수정, drop하기 위해 사용할 수 있습니다.
- query는 pad를 통해 이동합니다. `GST_PAD_PROBE_TYPE_QUREY_DOWNSTREAM`과 `GST_PAD_PROBE_TYPE_QUERY_UPSTREAM` flag를 이용해 downstream/upstream query를 선택할 수 있습니다. 양방향을 모두 선택하기 위해서  `GST_PAD_PROBE_TYPE_QUERY_BOTH`를 사용할 수 있습니다. Query probe는 두 번 알림을 받는데요, query가 upstream 혹은 downstream으로 전송되었을 때와 query의 결과값이 돌아왔을 때입니다. 사용자는 `GST_PAD_PROBE_TYPE_PUSH`나 `GST_PAD_PROBE_TYPE_PULL`를 가지고 어떤 단계에서 callback이 호출될지 결정할 수 있습니다; 각각 query가 수행될 때 혹은 query의 결과가 돌아왔을 때에요.
- dataflow에 대해 알림을 주는 것 뿐만 아니라, 사용자는 probe에게 callback이 돌아올 때 dataflow를 block 처리해달라고 요청할 수 있습니다. 이 요청을 blocking probe라고 부르는데, `GST_PAD_PROBE_TYPE_BLOCK` flag를 이용해서 활성화시킬 수 있으며, 선택된 활동에 대해서만 dataflow를 block하기 위해 다른 flag와 함께 사용할 수 있습니다. pad는 해당 probe가 삭제되거나, `GST_PAD_PROBE_REMOVE`를 해당 callback으로 return했을 때 unblock처리됩니다. `GST_PAD_PROBE_PASS`를 이용해서 현재 block 되어있는 아이템만 통과시킬수만 있으며, 해당 pad는 다음 아이템에 대해서는 다시 block됩니다.
probe를 blocking하는 건 잠시 동안 pad를 block하는 용도로 주로 쓰이는데, 해당 pad가 unlink되었거나 사용자가 해당 pad를 unlink시킬 예정이기 때문입니다. 만약 dataflow가 block되지 않는다면, data가 unlinked된 pad에 들어가게 되기 때문에 pipeline이 error 상태가 되게 됩니다. 어떻게 부분적으로 pipeline을 preroll시키기 위해 blocking probe를 사용하는지 확인하겠습니다. [Play a section of a media file](#play-a-section-of-a-media-file)을 확인하세요.
- pad에서 아무런 일도 일어나지 않을 때 알림을 받는 용도로도 사용합니다. `GST_PAD_PROBE_TYPE_IDLE` flag를 이용해 이 probe를 사용할 수 있습니다. 사용자는 `GST_PAD_PROBE_TYPE_PUSH`와 함께/혹은 `GST_PAD_PROBE_TYPE_PULL`을 이용해 pad scheduling mode에 따라 알림을 받을 수 있습니다. IDLE probe는 해당 probe가 설치되어 있는 한 어떤 data도 pad를 지나가지 못하게 한다는 점에서 blocking probe라고 할 수 있습니다.
사용자는 또한 "이상적인" probe를 pad를 동적으로 재연결할 때에 사용할 수 있게 합니다. 어떻게 idleb probe를 pipeline의 element를 대체하는 용도로 쓰이게 되는지 확인하도록 하겠습니다. [Dynamically changing the pipeline](#dynamically-changing-the-pipeline)을 확인하세요.

### Data probes

Data probe는 data가 pad를 통해 흐를 때 사용자에게 알림을 줍니다. 해당 종류의 probe를 생성하려면 `GST_PAD_PROBE_TYPE_BUFFER` 및/또는 `GST_PAD_PROBE_TYPE_BUFFER_LIST`를 `gst_pad_add_probe()`를 통해 보내십시오. `_chain()` 함수에서 수행할 수 있는 element들이 할 수 있는 가장 흔한 buffer 동작은 probe callback들 안에서 처리될 수 있습니다.

Data probe들은 pipeline의 streaming thread context에서 흐르게 되는데, 따라서 콜백들은 blocking과 이상한(weird한) 행동을 수행하는 것을 피하려고 노력해야 합니다. 해당 동작들은 pipeline의 동작성에 부정적인 영향을 줄 수 있으며, bug의 상황에서는, deadlock이나 crash를 유발할 수 있습니다. 더 상세하게 다루자면, 일반적으로 probe callback 안에서는 GUI와 연관된 함수를 호출하지 말아야 하며, pipeline의 상태를 변경하려고 하면 안 됩니다. 어플리케이션은 pipeline의 bus를 통해 custom message를 발행해서 main application의 쓰레드와 소통하거나, pipeline을 멈추는 등의 행동을 해야 합니다.

아래의 예시는 data probe를 사용하는 법에 대해 보여줍니다. 뭘 봐야 할지 모르겠다면, 이 프로그램의 결과를 `gst-launch-1.0 videotestsrc ! xvimagesink`와 비교해 보세요.

```c
#include <gst/gst.h>

static GstPadProbeReturn
cb_have_data (GstPad          *pad,
              GstPadProbeInfo *info,
              gpointer         user_data)
{
  gint x, y;
  GstMapInfo map;
  guint16 *ptr, t;
  GstBuffer *buffer;

  buffer = GST_PAD_PROBE_INFO_BUFFER (info);

  buffer = gst_buffer_make_writable (buffer);

  /* Making a buffer writable can fail (for example if it
   * cannot be copied and is used more than once)
   */
  if (buffer == NULL)
    return GST_PAD_PROBE_OK;

  /* Mapping a buffer can fail (non-writable) */
  if (gst_buffer_map (buffer, &map, GST_MAP_WRITE)) {
    ptr = (guint16 *) map.data;
    /* invert data */
    for (y = 0; y < 288; y++) {
      for (x = 0; x < 384 / 2; x++) {
        t = ptr[384 - 1 - x];
        ptr[384 - 1 - x] = ptr[x];
        ptr[x] = t;
      }
      ptr += 384;
    }
    gst_buffer_unmap (buffer, &map);
  }

  GST_PAD_PROBE_INFO_DATA (info) = buffer;

  return GST_PAD_PROBE_OK;
}

gint
main (gint   argc,
      gchar *argv[])
{
  GMainLoop *loop;
  GstElement *pipeline, *src, *sink, *filter, *csp;
  GstCaps *filtercaps;
  GstPad *pad;

  /* init GStreamer */
  gst_init (&argc, &argv);
  loop = g_main_loop_new (NULL, FALSE);

  /* build */
  pipeline = gst_pipeline_new ("my-pipeline");
  src = gst_element_factory_make ("videotestsrc", "src");
  if (src == NULL)
    g_error ("Could not create 'videotestsrc' element");

  filter = gst_element_factory_make ("capsfilter", "filter");
  g_assert (filter != NULL); /* should always exist */

  csp = gst_element_factory_make ("videoconvert", "csp");
  if (csp == NULL)
    g_error ("Could not create 'videoconvert' element");

  sink = gst_element_factory_make ("xvimagesink", "sink");
  if (sink == NULL) {
    sink = gst_element_factory_make ("ximagesink", "sink");
    if (sink == NULL)
      g_error ("Could not create neither 'xvimagesink' nor 'ximagesink' element");
  }

  gst_bin_add_many (GST_BIN (pipeline), src, filter, csp, sink, NULL);
  gst_element_link_many (src, filter, csp, sink, NULL);
  filtercaps = gst_caps_new_simple ("video/x-raw",
               "format", G_TYPE_STRING, "RGB16",
               "width", G_TYPE_INT, 384,
               "height", G_TYPE_INT, 288,
               "framerate", GST_TYPE_FRACTION, 25, 1,
               NULL);
  g_object_set (G_OBJECT (filter), "caps", filtercaps, NULL);
  gst_caps_unref (filtercaps);

  pad = gst_element_get_static_pad (src, "src");
  gst_pad_add_probe (pad, GST_PAD_PROBE_TYPE_BUFFER,
      (GstPadProbeCallback) cb_have_data, NULL, NULL);
  gst_object_unref (pad);

  /* run */
  gst_element_set_state (pipeline, GST_STATE_PLAYING);

  /* wait until it's up and running or failed */
  if (gst_element_get_state (pipeline, NULL, NULL, -1) == GST_STATE_CHANGE_FAILURE) {
    g_error ("Failed to go into PLAYING state");
  }

  g_print ("Running ...\n");
  g_main_loop_run (loop);

  /* exit */
  gst_element_set_state (pipeline, GST_STATE_NULL);
  gst_object_unref (pipeline);

  return 0;
}
```

확실하게 말하자면, pad probe callback은 buffer가 변경을 허용할 때만(writable할 때만)수정할 수 있습니다. 해당 buffer가 writable한지 아닌지는 pipeline과 속한 element와 연관되어 있습니다. 일반적으로 buffer는 writable하지만 가끔 아니기도 합니다. 그리고 아닌 경우에는 data나 metadata의 예상치 못한 수정은 디버깅하고 추적하기 매우 어려운 버그를 만들어낼 수 있습니다. 해당 buffer가 writable한지는 `gst_buffer_is_writable`을 통해 알 수 있습니다. 전달된 buffer와는 다른 buffer를 다시 전달할 수 있으므로, `gst_buffer_make_writable`을 callback 함수에서 사용해서 해당 buffer가 writable한지 확인하는 것이 좋은 아이디어일 것입니다.
Pad probe들은 파이프라인을 통과하는 데이터를 확인하는 데에 최적화되어있습니다. 만약 데이터 수정을 필요로 한다면, GStreamer element를 직접 작성하는 것이 더 나을 것입니다. `GstAudioFilter`, `GstVideoFilter` 혹은 `GstBaseTransform`과 같은 base class들은 이런 작업들을 더 쉽게 만들어 줍니다.
만약 사용자가 하고 싶은 것이 pipeline을 흐르는 buffer를 확인하는 것이라면, pad probe들을 세팅할 필요조차 없습니다. 사용자는 identity element를 pipeline에 단순히 삽입시켜 그것을 "handoff" signal에 연결시킬 수 있습니다. identity element는 또한 `dump`, `last-message`와 같은 유용한 디버깅 툴을 제공합니다. 후자(`last-message`)의 경우에는 '-v' swtich를 `gst-launch`로 전달하고, identity의 `silent` 요소를 `FALSE`로 설정하면 활성화됩니다.

### Play a section of a media file

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## Manually adding or removing data from/to a pipeline

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Inserting data with appsrc

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Grabbing data with appsink

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## Forcing a format

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Changing format in a PLAYING pipeline

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## Dynamically changing the pipeline

이번 섹션에서는 동적으로 파이프라인을 수정하는 기술들에 대해 다루겠습니다. 여기서는 특히 data flow에 대한 침범 없이 `PLAYING` 상태인 pipeline을 수정하는 방법에 대해 이야기하겠습니다.

우선, 동적인 파이프라인을 생성하는 데에 고려해야 하는 몇 가지 중요한 요소들이 있습니다.

- 파이프라인에서 element들을 삭제할 때, 치명적인 pipeline 에러를 피하기 위하여 unlinked된 pad에 dataflow가 없음을 확실히 해야 합니다. [Changing elements in a pipeline](#changing-elements-in-a-pipeline)를 확인하세요.
- 파이프라인에 element를 추가할 때, data flow를 파이프라인에 흘리기 전에 올바른 state로 해당 element를 설정하는 것 - 일반적으로 parent(이 경우에는 파이프라인의 element?)와 동일하게 설정합니다 - 을 잊지 마세요. element가 처음 생성될 때에 element는 `NULL` state로 설정되며, 이 상태일 때 data를 받으면 error를 return합니다. [Changing elements in a pipeline](#changing-elements-in-a-pipeline)를 확인하세요.
- 파이프라인에 element들을 추가할 때, GStreamer는 생성된 element의 clock과 base-time 값을 기본적으로 pipeline의 현재 값으로 설정합니다. 이것은 해당 element가 파이프라인에 있는 다른 element들과 같은 파이프라인 running-time을 구성할 수 있다는 것을 의미합니다. 또, 이 사실은 해당 (새로 생성된) element의 sink가 pipeline에 존재하는 다른 sink들과 마찬가지로 buffer에 동기화가 가능하며, 해당 source가 다른 source들과 일치하는 running-time을 가지고 있는 buffer를 생성할 수 있다는 것을 의미합니다.
- upstream chain에서 element의 연결을 제거할 때, 꼭 element의 sink pad로 `EOS` event를 내려보내고 `EOS`가 element를 떠났는지 확인해서(event probes 확인을 통해) element의 queued data가 완전히 flush되었는지 확인하세요.
만약 flush를 수행하지 않는다면, unlinked element에 buffered 되어있던 data를 잃게 됩니다. Data 손실은 simple frame loss(적은 수의 video frame 혹은 audio의 몇 miliseconds 정도)를 일으킬 수 있는데, 만약 muxer나 encoder, 혹은 이와 유사한 element들을 삭제하게 된다면, 재생을 제대로 수행할 수 없는 깨진 파일이 생기게 될 위험성이 생기게 됩니다. 이는 해당 element와 관련있는 metadata(헤더, seek/index table, 내부 sync tag들)이 정상적으로 저장되거나 업데이트되지 않을 수 있기 때문입니다. [Changing elements in a pipeline](#changing-elements-in-a-pipeline)를 확인하세요.
- Live source는 pipeline의 현재 `running-time`과 같은 `running-time`을 가진 buffer를 생성하게 됩니다.
live source가 아닌 파이프라인은 `running-time`이 0으로 시작하는 buffer를 생성하게 되고, 마찬가지로, flushing seek을 수행하면, 이러한 파이프라인들은 `running-time`을 0으로 초기화합니다.
`running-time`은 `gst_pad_set_offset()`을 통해 수정할 수 있습니다. 파이프라인을 구성하는 element들의 `running-time`이 동기화를 유지하기 위한 것이라는 사실을 명심하세요.
- element들을 추가하는 건 파이프라인의 상태를 변화시킬 수 있습니다. 예를 들어 non-prerolled sink를 추가하는 것은 pipeline을 다시 prerolling 상태로 되돌립니다. 또 예를 들면, non-prerolled sink를 지우는 것은, pipeline을 PAUSED와 PLAYING 상태로 변경시킵니다.
Live source를 추가하는 것은 preroll 단계를 취소하고 pipeline을 playing 상태로 변경합니다. live element를 추가하는 것도 pipeline의 latency를 변경시킬 수 있습니다.
파이프라인의 element들을 추가/삭제 하는 건 pipeline의 clock 선정까지 바꿀 수 있습니다. 만약 새로 추가된 element가 clock을 가지고 있다면, pipeline이 새 clock을 사용하도록 하는 게 나을 것입니다. 반면에 파이프라인에 clock을 제공하던 element가 삭제된다면, 다른 새 clock을 선택해야 할 것입니다.
- element들을 추가 혹은 삭제하는 것은 upstream 아니면 downstream element들로 하여금 cap과/혹은 할당자들을 다시 설정하도록 하게 할 수 있습니다. 어플리케이션 단계에서 뭔가를 새로 할 필요는 없지만, 플러그인이 그들 스스로를 format과 할당 방식을 최적화시키기 위해 새로운 파이프라인의 구성방식에 적용시킬 것입니다.
정말로 중요한 것은 element를 추가/삭제 혹은 변경을 수행할 때, pipeline이 새로운 format으로 변경할 수 있고 이 과정을 실패할 수 있다는 것입니다. 일반적으로 사용자는 이 fail을 필요한 곳에 올바른 converter element를 추가함으로써 고칠 수 있습니다. [Changing elements in a pipeline](#changing-elements-in-a-pipeline)를 확인하세요.

GStreamer는 거의 모든 동적인 pipeline 수정을 지원하지만, 사용자는 파이프라인 에러 발생 없이 이 작업을 해낼 수 있도록 몇가지 detail을 알아야 할 필요가 있습니다. 이 다음 섹션에서 몇가지 수정에 대한 use-case들을 확인해 보겠습니다.

### Changing elements in a pipeline

이 예시에서, 우리는 아래와 같은 element chain을 가지고 있다고 가정하겠습니다.

```C
   - ----.      .----------.      .---- -
element1 |      | element2 |      | element3
       src -> sink       src -> sink
   - ----'      '----------'      '---- -

```

파이프라인이 Playing 상태일 때 element2를 element4로 대체하고 싶다고 가정해 봅시다. element2는 가시화되어있으며 단지 파이프라인의 가시화 상태를 변경하고 싶다고 가정합니다.
우리는 단순히 element2의 sinkpad를 element1의 source pad에서 제거할 수는 없는데, 왜냐하면 해당 과정이 element1의 source pad를 unlinked 상태로 내버려둠으로써 source pad를 통해 data가 들어오는 상황에서 streaming error를 일으킬 수 있기 때문입니다. 이 문제를 해결하는 테크닉은 element2를 element4로 대체하기 전에 element1의 source pad에서 들어오는 dataflow를 blocking 처리하고, 아래 기술된 stop을 바탕으로 dataflow의 흐름을 재개하는 것입니다.

1. element1의 source pad를 'blocking pad probe'를 통해 Block 처리합니다. pad가 block되면, probe callback이 호출될 것입니다.
2. Block callback 내부에서는 element1과 element2 사이에 아무것도 흐르지 않게 되며, block이 풀리기 전까지는 해당 상태가 유지될 것입니다.
3. element1과 element2를 unlink처리하세요.
4. element2의 data가 flush되도록 하세요. 몇몇 element들은 내부적으로 data를 남겨둘 수 있기 때문에, element2 밖으로 데이터를 강제로 내보내서 데이터 손실이 일어나지 않도록 해야 합니다. 아래와 같이 `EOS` event를 element2로 보내서 해당 job을 수행할 수 있습니다.
  - element2의 source pad로 event probe를 집어넣습니다.
  - `EOS`를 element2의 sink pad로 보냅니다. 이 과정을 통해 element2의 내부에 있는 데이터를 강제로 내보낼 수 있습니다.
  - `EOS` event가 element2의 source pad에서 나타날 때까지 기다립니다. `EOS` 이벤트가 수신된다면, 해당 이벤트를 drop하고 event probe를 지웁니다.
5. element2와 element3을 unlink 처리합니다. 이제 파이프라인에서 element2를 삭제할 수 있으며, 해당 상태를 NULL로 변경할 수 있습니다.
6. 아직 추가되지 않았다면, 파이프라인에 element4를 추가합니다. element4와 element3을 연결하고, element1과 element4를 연결하세요.
7. element4가 element4를 제외한 파이프라인의 다른 element들과 같은 state인지 확인하세요. buffer와 event를 수신할 수 있게 되기 전에 적어도 `PAUSED` state는 되어야 합니다.
8. element1의 source pad probe를 unblock 처리하세요. 이 조치가 element4로 새로운 데이터가 들어가게 하고, streaming을 다시 시작하게 할 것입니다.

상기 알고리즘은 source pad가 blocked 상태일 때, 그러니까 dataflow가 pipeline에 존재할 때 동작합니다. 만약 dataflow가 없다면, 아직 element를 변경할 필요가 없으므로 이 알고리즘은 `PAUSED` state에서 사용할 수 있게 됩니다.

이 예시는 매 초마다 한번씩 간단한 pipeline의 video effect를 변경하는 방법을 보여줍니다.

```C
#include <gst/gst.h>

static gchar *opt_effects = NULL;

#define DEFAULT_EFFECTS "identity,exclusion,navigationtest," \
    "agingtv,videoflip,vertigotv,gaussianblur,shagadelictv,edgetv"

static GstPad *blockpad;
static GstElement *conv_before;
static GstElement *conv_after;
static GstElement *cur_effect;
static GstElement *pipeline;

static GQueue effects = G_QUEUE_INIT;

static GstPadProbeReturn
event_probe_cb (GstPad * pad, GstPadProbeInfo * info, gpointer user_data)
{
  GMainLoop *loop = user_data;
  GstElement *next;

  if (GST_EVENT_TYPE (GST_PAD_PROBE_INFO_DATA (info)) != GST_EVENT_EOS)
    return GST_PAD_PROBE_PASS;

  gst_pad_remove_probe (pad, GST_PAD_PROBE_INFO_ID (info));

  /* push current effect back into the queue */
  g_queue_push_tail (&effects, gst_object_ref (cur_effect));
  /* take next effect from the queue */
  next = g_queue_pop_head (&effects);
  if (next == NULL) {
    GST_DEBUG_OBJECT (pad, "no more effects");
    g_main_loop_quit (loop);
    return GST_PAD_PROBE_DROP;
  }

  g_print ("Switching from '%s' to '%s'..\n", GST_OBJECT_NAME (cur_effect),
      GST_OBJECT_NAME (next));

  gst_element_set_state (cur_effect, GST_STATE_NULL);

  /* remove unlinks automatically */
  GST_DEBUG_OBJECT (pipeline, "removing %" GST_PTR_FORMAT, cur_effect);
  gst_bin_remove (GST_BIN (pipeline), cur_effect);

  GST_DEBUG_OBJECT (pipeline, "adding   %" GST_PTR_FORMAT, next);
  gst_bin_add (GST_BIN (pipeline), next);

  GST_DEBUG_OBJECT (pipeline, "linking..");
  gst_element_link_many (conv_before, next, conv_after, NULL);

  gst_element_set_state (next, GST_STATE_PLAYING);

  cur_effect = next;
  GST_DEBUG_OBJECT (pipeline, "done");

  return GST_PAD_PROBE_DROP;
}

static GstPadProbeReturn
pad_probe_cb (GstPad * pad, GstPadProbeInfo * info, gpointer user_data)
{
  GstPad *srcpad, *sinkpad;

  GST_DEBUG_OBJECT (pad, "pad is blocked now");

  /* remove the probe first */
  gst_pad_remove_probe (pad, GST_PAD_PROBE_INFO_ID (info));

  /* install new probe for EOS */
  srcpad = gst_element_get_static_pad (cur_effect, "src");
  gst_pad_add_probe (srcpad, GST_PAD_PROBE_TYPE_BLOCK |
      GST_PAD_PROBE_TYPE_EVENT_DOWNSTREAM, event_probe_cb, user_data, NULL);
  gst_object_unref (srcpad);

  /* push EOS into the element, the probe will be fired when the
   * EOS leaves the effect and it has thus drained all of its data */
  sinkpad = gst_element_get_static_pad (cur_effect, "sink");
  gst_pad_send_event (sinkpad, gst_event_new_eos ());
  gst_object_unref (sinkpad);

  return GST_PAD_PROBE_OK;
}

static gboolean
timeout_cb (gpointer user_data)
{
  gst_pad_add_probe (blockpad, GST_PAD_PROBE_TYPE_BLOCK_DOWNSTREAM,
      pad_probe_cb, user_data, NULL);

  return TRUE;
}

static gboolean
bus_cb (GstBus * bus, GstMessage * msg, gpointer user_data)
{
  GMainLoop *loop = user_data;

  switch (GST_MESSAGE_TYPE (msg)) {
    case GST_MESSAGE_ERROR:{
      GError *err = NULL;
      gchar *dbg;

      gst_message_parse_error (msg, &err, &dbg);
      gst_object_default_error (msg->src, err, dbg);
      g_clear_error (&err);
      g_free (dbg);
      g_main_loop_quit (loop);
      break;
    }
    default:
      break;
  }
  return TRUE;
}

int
main (int argc, char **argv)
{
  GOptionEntry options[] = {
    {"effects", 'e', 0, G_OPTION_ARG_STRING, &opt_effects,
        "Effects to use (comma-separated list of element names)", NULL},
    {NULL}
  };
  GOptionContext *ctx;
  GError *err = NULL;
  GMainLoop *loop;
  GstElement *src, *q1, *q2, *effect, *filter1, *filter2, *sink;
  gchar **effect_names, **e;

  ctx = g_option_context_new ("");
  g_option_context_add_main_entries (ctx, options, NULL);
  g_option_context_add_group (ctx, gst_init_get_option_group ());
  if (!g_option_context_parse (ctx, &argc, &argv, &err)) {
    g_print ("Error initializing: %s\n", err->message);
    g_clear_error (&amp;err);
    g_option_context_free (ctx);
    return 1;
  }
  g_option_context_free (ctx);

  if (opt_effects != NULL)
    effect_names = g_strsplit (opt_effects, ",", -1);
  else
    effect_names = g_strsplit (DEFAULT_EFFECTS, ",", -1);

  for (e = effect_names; e != NULL && *e != NULL; ++e) {
    GstElement *el;

    el = gst_element_factory_make (*e, NULL);
    if (el) {
      g_print ("Adding effect '%s'\n", *e);
      g_queue_push_tail (&effects, el);
    }
  }

  pipeline = gst_pipeline_new ("pipeline");

  src = gst_element_factory_make ("videotestsrc", NULL);
  g_object_set (src, "is-live", TRUE, NULL);

  filter1 = gst_element_factory_make ("capsfilter", NULL);
  gst_util_set_object_arg (G_OBJECT (filter1), "caps",
      "video/x-raw, width=320, height=240, "
      "format={ I420, YV12, YUY2, UYVY, AYUV, Y41B, Y42B, "
      "YVYU, Y444, v210, v216, NV12, NV21, UYVP, A420, YUV9, YVU9, IYU1 }");

  q1 = gst_element_factory_make ("queue", NULL);

  blockpad = gst_element_get_static_pad (q1, "src");

  conv_before = gst_element_factory_make ("videoconvert", NULL);

  effect = g_queue_pop_head (&effects);
  cur_effect = effect;

  conv_after = gst_element_factory_make ("videoconvert", NULL);

  q2 = gst_element_factory_make ("queue", NULL);

  filter2 = gst_element_factory_make ("capsfilter", NULL);
  gst_util_set_object_arg (G_OBJECT (filter2), "caps",
      "video/x-raw, width=320, height=240, "
      "format={ RGBx, BGRx, xRGB, xBGR, RGBA, BGRA, ARGB, ABGR, RGB, BGR }");

  sink = gst_element_factory_make ("ximagesink", NULL);

  gst_bin_add_many (GST_BIN (pipeline), src, filter1, q1, conv_before, effect,
      conv_after, q2, sink, NULL);

  gst_element_link_many (src, filter1, q1, conv_before, effect, conv_after,
      q2, sink, NULL);

  gst_element_set_state (pipeline, GST_STATE_PLAYING);

  loop = g_main_loop_new (NULL, FALSE);

  gst_bus_add_watch (GST_ELEMENT_BUS (pipeline), bus_cb, loop);

  g_timeout_add_seconds (1, timeout_cb, loop);

  g_main_loop_run (loop);

  gst_element_set_state (pipeline, GST_STATE_NULL);
  gst_object_unref (pipeline);

  return 0;
}
```

우리가 해당 효과의 전과 후에 어떻게 videoconvert element를 추가했는지를 주목해 주세요. 이 조치는 몇몇 element들이 다른 colorspaces에서 동작할 수 있기 때문에 필요합니다. Conversion element를 추가함에 따라 적합한 format이 적용될 수 있음을 확신할 수 있습니다.
