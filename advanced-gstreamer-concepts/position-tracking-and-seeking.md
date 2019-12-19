Position tracking and seeking
=====

지금까지 미디어 처리를 위한 파이프라인의 생성 방법과 해당 파이프라인을 어떻게 동작시킬 수 있는지에 대해 확인해 왔습니다. 대부분의 어플리케이션 개발자라면 이것들 뿐만이 아니라 사용자에게 미디어의 상태에 대한 정보를 제공하는 것에 대해서도 관심이 있을 것입니다. 예를 들어서 미디어 플레이어의 경우 노래의 재생 상태에 대한 상태 바(progress bar)나 전체 stream의 길이를 나타내고 싶어할 것입니다. 트랜스코딩 어플리케이션의 경우라면 현재 작업이 어디까지 수행되었는지에 대한 상태 바를 표시하고 싶어하겠죠. GStreamer는 이 모든 것을 *querying*이라고 알려진 개념을 이용해 built-in 서포트를 제공합니다. Seeking이라는 개념도 Querying과 매우 유사하기 때문에, 해당 개념도 여기에서 함께 다루겠습니다. Seeking은 *events*라는 개념을 이용해 수행됩니다.

Table of Contents
-----

- [Position tracking and seeking](#position-tracking-and-seeking)
  - [Table of Contents](#table-of-contents)
  - [Querying: getting the position or length of a stream](#querying-getting-the-position-or-length-of-a-stream)
  - [Events: seeking (and more)](#events-seeking-and-more)

Querying: getting the position or length of a stream
-----

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

Events: seeking (and more)
-----

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
