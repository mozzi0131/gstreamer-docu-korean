# Metadata

GStreamer는 지원하는 두 가지의 메타데이터(Stream tags, Stream info)를 명확하게 구분하고 있습니다. Stream tag는 stream의 요소들을 기술적이지 않은 방법으로 표현하며, Stream info는 stream의 요소들을 기술적으로 표현합니다. 예를 들어 Stream tag는 작곡가, 노래의 제목 혹은 앨범의 이름 등을 포함하며 Stream info는 video의 크기, audio samplerate, 사용된 codec등을 포함합니다.

Tag들은 GStreamer의 태깅 시스템을 통해 관리됩니다. Stream-info는 GstPad의 현재 GstCaps를 가져와서 검색할 수 있습니다.

## Table of Contents

- [Metadata](#metadata)
  - [Table of Contents](#table-of-contents)
  - [Metadata reading](#metadata-reading)
  - [Tag writing](#tag-writing)

## Metadata reading

Stream information은 GstPad에서 얻으면 가장 쉽게 얻을 수 있습니다. 하지만 이 방법을 사용하려면 stream information을 얻고자 하는 모든 pad에 접근해야 합니다. 이 방법은 이미 [Using capabilities for metadata](https://gstreamer.freedesktop.org/documentation/application-development/basics/pads.html?gi-language=c#using-capabilities-for-metadata)에서 논의되었기 때문에, 여기에서는 생략하도록 하겠습니다.

Tag를 읽는 것은 GStreamer의 bus를 통해 수행됩니다. `GST_MESSAGE_TAG` 메세지를 통해 Tag를 알 수 있고, 원하는 대로 관리할 수 있습니다. 이것 또한 [Bus](https://gstreamer.freedesktop.org/documentation/application-development/basics/bus.html?gi-language=c) 에서 이전에 이야기되었었습니다.

하지만 `GST_MESSAGE_TAG` 메세지는 여러 번 나타날 수 있으며, 태그를 일관성 있게 집계하고 표시하는 것은 어플리케이션의 책임이라는 것을 유의해 주십시오. 해당 작업은 `gst_tag_list_merge()`를 통해 수행할 수 있지만, 새로운 노래를 loading 할 때마다, 혹은 인터넷으로 라디오를 들을 때 몇 분마다 캐시를 비워주는 것을 잊지 마세요. 또한 나중에 들어오게 되는 새로운 제목이 오래된 제목보다 우선권을 가질 수 있도록 `GST_TAG_MERGE_PREPEND`를 병합 모드로 이용해 주세요.

아래의 예제 코드는 어떻게 파일에서 태그를 추출하고 출력하는지에 대해 보여줍니다.

```c
/* compile with:
 * gcc -o tags tags.c `pkg-config --cflags --libs gstreamer-1.0` */
#include <gst/gst.h>

static void
print_one_tag (const GstTagList * list, const gchar * tag, gpointer user_data)
{
  int i, num;

  num = gst_tag_list_get_tag_size (list, tag);
  for (i = 0; i < num; ++i) {
    const GValue *val;

    /* Note: when looking for specific tags, use the gst_tag_list_get_xyz() API,
     * we only use the GValue approach here because it is more generic */
    val = gst_tag_list_get_value_index (list, tag, i);
    if (G_VALUE_HOLDS_STRING (val)) {
      g_print ("\t%20s : %s\n", tag, g_value_get_string (val));
    } else if (G_VALUE_HOLDS_UINT (val)) {
      g_print ("\t%20s : %u\n", tag, g_value_get_uint (val));
    } else if (G_VALUE_HOLDS_DOUBLE (val)) {
      g_print ("\t%20s : %g\n", tag, g_value_get_double (val));
    } else if (G_VALUE_HOLDS_BOOLEAN (val)) {
      g_print ("\t%20s : %s\n", tag,
          (g_value_get_boolean (val)) ? "true" : "false");
    } else if (GST_VALUE_HOLDS_BUFFER (val)) {
      GstBuffer *buf = gst_value_get_buffer (val);
      guint buffer_size = gst_buffer_get_size (buf);

      g_print ("\t%20s : buffer of size %u\n", tag, buffer_size);
    } else if (GST_VALUE_HOLDS_DATE_TIME (val)) {
      GstDateTime *dt = g_value_get_boxed (val);
      gchar *dt_str = gst_date_time_to_iso8601_string (dt);

      g_print ("\t%20s : %s\n", tag, dt_str);
      g_free (dt_str);
    } else {
      g_print ("\t%20s : tag of type '%s'\n", tag, G_VALUE_TYPE_NAME (val));
    }
  }
}

static void
on_new_pad (GstElement * dec, GstPad * pad, GstElement * fakesink)
{
  GstPad *sinkpad;

  sinkpad = gst_element_get_static_pad (fakesink, "sink");
  if (!gst_pad_is_linked (sinkpad)) {
    if (gst_pad_link (pad, sinkpad) != GST_PAD_LINK_OK)
      g_error ("Failed to link pads!");
  }
  gst_object_unref (sinkpad);
}

int
main (int argc, char ** argv)
{
  GstElement *pipe, *dec, *sink;
  GstMessage *msg;
  gchar *uri;

  gst_init (&argc, &argv);

  if (argc < 2)
    g_error ("Usage: %s FILE or URI", argv[0]);

  if (gst_uri_is_valid (argv[1])) {
    uri = g_strdup (argv[1]);
  } else {
    uri = gst_filename_to_uri (argv[1], NULL);
  }

  pipe = gst_pipeline_new ("pipeline");

  dec = gst_element_factory_make ("uridecodebin", NULL);
  g_object_set (dec, "uri", uri, NULL);
  gst_bin_add (GST_BIN (pipe), dec);

  sink = gst_element_factory_make ("fakesink", NULL);
  gst_bin_add (GST_BIN (pipe), sink);

  g_signal_connect (dec, "pad-added", G_CALLBACK (on_new_pad), sink);

  gst_element_set_state (pipe, GST_STATE_PAUSED);

  while (TRUE) {
    GstTagList *tags = NULL;

    msg = gst_bus_timed_pop_filtered (GST_ELEMENT_BUS (pipe),
        GST_CLOCK_TIME_NONE,
        GST_MESSAGE_ASYNC_DONE | GST_MESSAGE_TAG | GST_MESSAGE_ERROR);

    if (GST_MESSAGE_TYPE (msg) != GST_MESSAGE_TAG) /* error or async_done */
      break;

    gst_message_parse_tag (msg, &tags);

    g_print ("Got tags from element %s:\n", GST_OBJECT_NAME (msg->src));
    gst_tag_list_foreach (tags, print_one_tag, NULL);
    g_print ("\n");
    gst_tag_list_unref (tags);

    gst_message_unref (msg);
  }

  if (GST_MESSAGE_TYPE (msg) == GST_MESSAGE_ERROR) {
    GError *err = NULL;

    gst_message_parse_error (msg, &err, NULL);
    g_printerr ("Got error: %s\n", err->message);
    g_error_free (err);
  }

  gst_message_unref (msg);
  gst_element_set_state (pipe, GST_STATE_NULL);
  gst_object_unref (pipe);
  g_free (uri);
  return 0;
}
```

## Tag writing

Tag 작성은 `GstTagSetter` 인터페이스를 통해 수행할 수 있습니다. 이를 위해 필요한 것은 해당 파이프라인에 tag-set을 지원하는 element를 포함하는 것 뿐입니다.

파이프라인에 tag 작성을 지원하는 element가 존재하는지 확인하기 위해서 `gst_bin_iterate_all_by_interface(pipeline, GST_TYPE_TAG_SETTER)`을 이용할 수 있습니다. 결과로 추출된 element(일반적으로 encoder 혹은 muxer)에서 taglist와 함께 `gst_tag_setter_merge_tags()`, 혹은 개별적인 tag와 함께 `gst_tag_setter_add_tags()`를 이용해 tag를 설정할 수 있습니다.

GStreamer의 tag 지원에서 근사한 점(^^;) 중 하나는, tag가 파이프라인에 보존된다는 것입니다. 즉, 태그가 포함된 파일 하나를 다른 미디어 타입(마찬가지로 태그를 지원해야 합니다)으로 변환할 경우, 태그는 데이터 stream의 일부처럼 처리되어 새로 작성된 media file에 병합될 것입니다.
