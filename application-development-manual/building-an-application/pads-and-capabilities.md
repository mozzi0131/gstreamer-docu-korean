# Pads and capabilities

저희가 지금까지 [Elements](https://gstreamer.freedesktop.org/documentation/application-development/basics/elements.html?gi-language=c)에서 확인해온 바와 같이, Pads는 외부와 통신하는 element의 interface입니다. Data는 한 element의 source pad에서 다른 element의 sink pad로 흐르게 됩니다. 나중에 이 챕터에서 capabilities에 대해 이야기하겠습니다. ([Capabilities of a pad](#capabilities-of-a-pad)를 확인하세요.)

## Table of Contents

- [Pads and capabilities](#pads-and-capabilities)
  - [Table of Contents](#table-of-contents)
  - [Pads](#pads)
    - [Dynamic (or sometimes) pads](#dynamic-or-sometimes-pads)
    - [Request pads](#request-pads)
  - [Capabilities of a pad](#capabilities-of-a-pad)
    - [Dissecting capabilities](#dissecting-capabilities)
    - [Properties and values](#properties-and-values)
  - [What capabilities are used for](#what-capabilities-are-used-for)
    - [Using capabilities for metadata](#using-capabilities-for-metadata)
    - [Creating capabilities for filtering](#creating-capabilities-for-filtering)
  - [Ghost pads](#ghost-pads)

## Pads

Pad의 종류는 방향과 가용성이라는 두 가지 요소를 바탕으로 정의됩니다. 이전에 이야기했다시피, GStreamer는 source pad와 sink pad라는 두 종류의 방향을 가지고 있습니다. 이 개념은 element에서의 시점을 바탕으로 정의된 것입니다. element들은 sink pad에서 data를 전송받고, source pad에서 data를 생성합니다. 구조적으로 sink pad들은 element의 왼쪽에 그려지며 source pad는 오른쪽에 그려집니다. 따라서 그래프상으로 데이터는 왼쪽에서 오른쪽으로 흐르게 됩니다.

Pad의 방향들은 Pad의 가용성에 비하면 매우 단순합니다. Pad는 가용성의 세 가지 상태인 always, sometimes, on request 중 그 어떤 것이든 가질 수 있습니다. 이 세 가지 타입들의 의미는 상태의 이름과 완전히 동일합니다. Always의 경우에는 언제든지 pad가 존재한다는 의미이며, sometimes의 의미는 특정 케이스에서만 pad가 존재하며 랜덤하게 사라질 수 있기도 하다는 의미입니다. 그리고 on-request의 의미는 어플리케이션에서 요청이 들어왔을 때에만 pad가 존재한다는 의미입니다.

### Dynamic (or sometimes) pads

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Request pads

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## Capabilities of a pad

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Dissecting capabilities

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Properties and values

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## What capabilities are used for

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

### Using capabilities for metadata

Pad는 하나 혹은 그 이상의 가용성을 가질 수 있습니다. Capability들은(`GstCaps`) 하나 이상의 `GstStructure` 배열의 형태로 표현되며, 각각의 `GstStructure`들은 field명의 문자열(ex. "width")과 type을 가진 value들(ex. `G_TYPE_INT` 혹은 `GST_TYPE_INT_RANGE`)를 포함한 field의 배열입니다.

*possible*한 패드의 가용성과 *allowed*된 패드의 가용성, 그리고 *negotiated*된 패드의 가용성 사이에는 큰 차이가 있다는 것을 기억해 주세요. *Possible*한 패드의 가용성은 gst-inspect 상에서 볼 수 있는 pad 템플릿의 cap들이고, *Allowed*된 패드의 가용성은 패드의 구성요소인 cap들 혹은 해당 cap들의 부분집합들입니다. *Allowed*된 패드의 cap들은 peer pad의 possible cap에 대한 의존성이 존재합니다. 마지막으로 *Negotiated* cap은 stream/buffer의 정확한 형태를 나타내고, range나 list와 같은 가변적인 변수 bit 없이 정확히 하나의 구조만 포함합니다.(ex. fixed caps) (?)

capability들의 집합에서 요소들의 값을 구조체의 개별적인 요소에 querying 수행함으로써 얻어올 수 있습니다. `gst_caps_get_structure()`를 이용해서 cap의 구조체를 얻어올 수 있고, `gst_caps_get_size()`를 통해 `GstCaps`에서 가지고 있는 구조체의 개수를 얻어올 수 있습니다.

Caps는 하나의 구조체만 가지고 있을 때 'Simple caps'라고 불려지며, 가변적인 필드값 없이(가능한 값들의 range 혹은 list들 없이) 하나의 구조만 가지고 있는 경우 *fixed caps*라고 불립니다. 이것 외의 또 다른 두 가지 특별한 타입의 cap은 *ANY* cap과 *empty* cap입니다.

아래 예제 코드를 통해 어떻게 fixed video cap들의 집합에서 width와 height를 추출하는지 확인하실 수 있습니다.

```c
static void
read_video_props (GstCaps *caps)
{
  gint width, height;
  const GstStructure *str;

  g_return_if_fail (gst_caps_is_fixed (caps));

  str = gst_caps_get_structure (caps, 0);
  if (!gst_structure_get_int (str, "width", &width) ||
      !gst_structure_get_int (str, "height", &height)) {
    g_print ("No width/height available\n");
    return;
  }

  g_print ("The video size of this set of capabilities is %dx%d\n",
       width, height);
}
```

### Creating capabilities for filtering

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## Ghost pads

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
