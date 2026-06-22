# VideoFly Profile 설정 가이드

VideoFly profile v1 규약에 대해 기술한다.
Profile은 원본 입력의 출력을 기술한다.

- HLS(Http Live Streaming)
- 멀티 Playlist (Multi BitRate Streaming)
- Multivariant 포맷 (fMP4, MPEG2-TS)

그 외 세부 오디오/비디오 인코딩, encrypt, composition 같은 세부 사항이 포함된다.

## 기본 구조

메타정보, ``hls`` 공통설정, `renditions` 세 부분으로 구성한다.
`renditions`는 해상도별 상세 출력을 정의하며, 순서대로 `master.m3u8` 에 노출된다.

```json
{
  "schema_version": "v1",
  "description": "기본 VideoFly profile",
  "hls": {
    "transcode": {
      "policy": "if_spec_mismatch",
      "upscale": false
    },
    "version": 7,
    "path": "abs",
    "split": "loose",
    "segment_duration": 2
  },
  "renditions": [
    {
      "name": "fmp4_720p",
      "container": "fmp4",
      "audio": {
        "codec": "aac",
        "channels": 2
      },
      "video": {
        "codec": "av1",
        "resolution": "1280x720"
      }
    }
  ]
}
```

| 설정 | 설명 |
| --- | --- |
| `schema_version` | 현재 schema 버전은 반드시 `"v1"` 이어야한다. |
| `description` | 설명. 기능적 정의는 없다. |
| `hls` | HLS 제공 형태를 정의한다. |
| `renditions` | Multivariant/rendition 출력에 대한 정의이다. |

Profile 이름은 JSON 내부 필드가 아니라 파일명으로 결정한다.
예를 들어 `_default.json` 파일의 profile 이름은 `_default`이다.

## 기본 Profile 상속

모든 ``v1`` 의 기본 Profile은 `schemes/v1/profile.default.json` 로 사용자 Profile은 이 설정을 오바라이드 하는 형태로 정의된다.

-  사용자 profile의 값은 기본 Profile보다 우선한다.
-  기본 Profile에 정의되지 않은 설정은 인식되지 않는다.
-  트랜스코딩시 입력되는 Profile은 반드시 기본 Profile을 베이스로 사용자 Profile을 결합한 형태로 정의해야한다.

Profile 작성 예시는 `schemes/v1/examples/profile.sample.json`을 참고한다.

## HLS 설정

`hls`는 모든 Playlist의 공통설정이다.

| 설정 | 값 | 설명 |
| --- | --- | --- |
| `transcode.policy` | `always`, `if_bitrate_exceeds`, `if_spec_mismatch`, `if_codec_mismatch` | 트랜스코딩 수행정책 |
| `transcode.upscale` | boolean | 원본이 출력해상도보다 작을 경우 트랜스코딩 수행여부 |
| `version` | integer | playlist에 기록할 `EXT-X-VERSION` 값. fMP4 호환시 기본 값은 `7` 이다. |
| `path` | `abs`, `rel` | m3u8내 URL 형식으로 ``abs`` 절대경로, ``rel`` 은 상태경로이다. |
| `split` | `loose`, `balance`, `strict` | segment 분할 방식 |
| `segment_duration` | number | 목표 segment 길이(초) |

### `transcode.policy` 상세

원본 조건별 트랜스코딩 수행정책.

| 값 | 설명 |
| --- | --- |
| `always` | 항상 트랜스코딩. |
| `if_bitrate_exceeds` | 원본 codec/resolution이 rendition과 같고, 원본 bitrate가 rendition bitrate와 같거나 낮으면 트랜스코딩하지 않는다. |
| `if_spec_mismatch` | 원본 codec/resolution이 rendition과 같으면 트랜스코딩하지 않는다. |
| `if_codec_mismatch` | 원본 codec이 rendition과 같으면 트랜스코딩하지 않는다. |

`upscale`은 원본보다 큰 해상도의 rendition을 허용할지 결정한다.
`false`이면 원본보다 큰 해상도로 트랜스코딩하지 않는다.


### Split 값별 동작

| 값 | 동작 | 예시 |
| --- | --- | --- |
| `loose` | 원본 키프레임을 위배하지 않는 범위에서 분할한다. | ``segment_duration`` 보다 큰 원본 세그먼트는 그대로 유지되며, 작은 경우 ``segment_duration`` 에 맞추어 병합된다. |
| `balance` | `segment_duration` 기준 +/- 1초 범위에서 가능한 적게 분할한다. | 가능한 ``segment_duration`` 과 적은 오차를 가지도록 원본 세그먼트를 분할한다. 예를 들어 원본 세그먼트가 10초, 출력이 2초라면 5개로 분할된다.  |
| `strict` | `segment_duration` 단위에 맞춰 분할한다. | 정확히 목표 `segment_duration` 로 강제분할한다. |

다음은 분할 예제이다.

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:2
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD

#EXT-X-MAP:URI="init.mp4"

#EXTINF:2.012,
0.m4s
#EXTINF:1.992,
1.m4s

#EXT-X-ENDLIST
```

## Rendition 설정

`renditions[]`의 각 항목은 하나의 출력 설정을 의미한다.
예를 들어 `fmp4_720p`, `fmp4_360p`, `m2ts_720p`처럼 container와 해상도 단위로 구성할 수 있다.

| 설정 | 설명 |
| --- | --- |
| `name` | 개별 rendition 명. `{container}_{name}` 형태를 사용한다. 예: `fmp4_720p`. |
| `container` | HLS segment container이다. `fmp4` 또는 `m2ts`를 사용한다. |
| `bandwidth` | 출력비디오로부터 대역폭을 추정할 수 없을 경우에만 이 설정값이 표기된다. |
| `audio` | 오디오 출력 설정 |
| `video` | 비디오 출력 설정 |


```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720,CODECS="av01.0.05M.08,mp4a.40.2"
fmp4_720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=900000,RESOLUTION=640x360,CODECS="av01.0.04M.08,mp4a.40.2"
fmp4_360p/playlist.m3u8
```

## Audio 설정

`audio`는 rendition의 오디오 출력 조건을 설정한다.

| 설정 | 값 | 설명 | FFmpeg 참고 |
| --- | --- | --- | --- |
| `enable` | boolean | `false`이면 오디오를 제외한다. | `-an` |
| `codec` | `aac`, `ac3`, `eac3`, `opus` | 오디오 코덱이다. | `-acodec` |
| `codec_options.profile` | `aac_low`, `aac_he`, `aac_he_v2`, `auto` | 오디오 코덱 profile이다. | `-aprofile` |
| `sample_rate` | integer | 오디오 sample rate(Hz)이다. | `-ar` |
| `bit_rate` | integer | 오디오 bitrate(bps)이다. | `-b:a` |
| `channels` | integer | 오디오 channel 수이다. | `-ac` |

AAC-LC는 `aac_low`를 사용한다.
`codec_options.profile = auto`이면 별도 profile 옵션을 지정하지 않는다.

## Video 설정

`video`는 rendition의 비디오 출력 조건을 설정한다.

| 설정 | 설명 | FFmpeg 참고 |
| --- | --- | --- |
| `enable` | `false`이면 비디오를 제외한다. | `-vn` |
| `codec` | 출력 비디오 코덱이다. `h264`, `av1`을 사용할 수 있다. | `-vcodec` |
| `gpu` | GPU encoder preset과 quality 설정이다. | `-preset`, `-cq` |
| `cpu` | CPU encoder preset과 quality 설정이다. | `-preset`, `-crf` |
| `codec_options.profile` | 코덱 profile이다. 예: `main`, `high`. | `-vprofile` |
| `codec_options.level` | 코덱 level이다. 예: `4.0`, `4.1`. | `-vlevel` |
| `codec_options.max_bitrate` | 최대 video bitrate(bps)이다. | `-maxrate`, `-bufsize` |
| `codec_options.min_bitrate` | 최소 video bitrate(bps)이다. | `-minrate` |
| `codec_options.chroma_subsampling` | pixel format이다. 예: `yuv420p`, `nv12`, `auto`. | `-pix_fmt` |
| `codec_options.interlace_mode` | interlace 처리 방식이다. 예: `progressive`, `top_field_first`. | `-field_order` |
| `bit_rate` | 목표 video bitrate(bps)이다. | `-b:v` |
| `frame_rate` | 출력 frame rate(fps)이다. | `fps` filter |
| `max_frame_rate` | 최대 출력 frame rate(fps)이다. | `-fpsmax` |
| `resolution` | 목표 해상도이다. `{width}x{height}` 형식을 사용한다. | scale/crop/pad filter |
| `sizing_policy` | 원본을 목표 해상도에 맞추는 방식이다. | scale/crop/pad filter |
| `padding_policy` | 남는 영역을 padding할지 결정한다. | pad filter |

CPU/GPU 실행 환경에 따라 실제 encoder는 달라질 수 있다.
예를 들어 H.264는 CPU에서 `libx264`, GPU에서 `h264_nvenc`를 사용할 수 있고, AV1은 CPU에서 `libsvtav1`, GPU에서 `av1_nvenc`를 사용할 수 있다.

Quality 값은 bitrate보다 우선하는 운영 기본 정책이다.
`bit_rate`, `max_bitrate`, `min_bitrate`는 필요한 경우에만 사용하는 override 성격의 optional 값이다.

## AWS MediaConvert 유사 옵션 처리 기준

AWS MediaConvert에는 유사 옵션이 있으나, VideoFly profile v1에서는 다음 항목을 고객 설정값으로 제공하지 않는다.

| AWS 옵션 | 유사 개념 | Profile v1 처리 |
| --- | --- | --- |
| `H264Settings.NumberReferenceFrames` | reference frame 수 | 인코더 세부 튜닝 값으로 보고 Profile에 포함하지 않는다. |
| `H264Settings.EntropyEncoding` | CABAC/CAVLC 선택 | 인코더 정책으로 처리하고 Profile에 포함하지 않는다. |
| `ColorCorrector.ColorSpaceConversion` | 색공간/컬러 매트릭스 변환 | codec option이 아니므로 Profile v1에서는 제공하지 않는다. |
| fixed GOP, keyframe max distance | 키프레임/GOP 제어 | 별도 설정으로 제공하지 않고 `hls.split`, `segment_duration` 기준으로 처리한다. |

색공간 변환은 출력 codec과 별개로 BT.709, BT.2020 같은 색공간을 강제로 맞춰야 할 때 사용한다.
예를 들어 H.265 출력에서 SDR 호환을 위해 BT.709로 변환하거나, H.264 출력에서 원본 BT.2020 색공간을 유지/명시해야 하는 경우가 여기에 해당한다.
H.264/H.265에서는 색공간 자체를 codec profile처럼 선택하는 것이 아니라, 인코딩 전 필터로 픽셀 값을 변환하고 bitstream에는 color primaries, transfer characteristics, matrix coefficients 같은 색상 메타데이터가 기록되는 형태로 사용된다.

## Sizing/Padding 설정

`sizing_policy`와 `padding_policy`는 원본 영상을 목표 해상도에 맞추는 방법을 결정한다.
`sizing_policy`를 생략하면 `fit_to_orientation_no_upscale`로 처리한다.
`padding_policy`는 생략하면 `no_pad` 동작을 기본으로 본다.

| `sizing_policy` | 설명 |
| --- | --- |
| `fit` | 원본 비율을 유지하면서 목표 박스 안에 맞춘다. |
| `fit_to_orientation` | 원본이 세로이면 width/height 기준을 바꾼 뒤 `fit`을 적용한다. |
| `fill` | 원본 비율을 유지하면서 목표 박스를 채우고 중앙을 기준으로 crop한다. |
| `stretch` | 비율을 유지하지 않고 목표 해상도로 늘린다. |
| `center_crop` | scale 없이 목표 박스 범위로 중앙 crop한다. |
| `fit_no_upscale` | 원본보다 키우지 않고 목표 박스 안에 맞춘다. |
| `fill_no_upscale` | 원본보다 키우지 않고 목표 박스를 채운 뒤 중앙 crop한다. |
| `fit_to_orientation_no_upscale` | 원본 방향을 반영하고, 원본보다 키우지 않는 `fit`을 적용한다. |

| `padding_policy` | 설명 |
| --- | --- |
| `no_pad` | padding을 추가하지 않는다. |
| `pad` | 목표 해상도까지 letterbox padding을 추가한다. |

## Encrypt 설정

`encrypt`는 HLS AES-128 암호화 정보를 설정한다.

| 설정 | 설명 |
| --- | --- |
| `enable` | 암호화 사용 여부이다. 생략하면 `false`로 처리한다. |
| `key_file_name` | playlist에 기록할 key URI 기준 값이다. |
| `token` | key file 또는 key material 생성을 위한 값이다. 운영용 비밀값은 공유 profile에 저장하지 않는 것을 권장한다. |
| `token_type` | `raw`는 token을 그대로 사용하고, `enc`는 복호화 후 사용한다. |
| `iv` | 선택적 AES-128 IV이다. `0x`로 시작하는 16바이트 16진수 문자열을 사용한다. |

`token`, `token_type`은 key material을 만들 때 사용하며 media playlist에 직접 기록하지 않는다.
암호화를 사용하면 media playlist에는 다음과 같은 `EXT-X-KEY`가 기록된다.

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="enc.key",IV=0x0123456789abcdef0123456789abcdef
```

## Composition 설정

`composition`은 원본 앞 또는 뒤에 이미지/영상 asset을 붙여 하나의 HLS 출력처럼 재생할 때 사용한다.
광고, 안내 slate, intro/outro 영상처럼 본편과 함께 하나의 HLS playlist로 제공해야 하는 asset을 정의할 수 있다.

| 설정 | 설명 |
| --- | --- |
| `enable` | composition 사용 여부이다. 생략하면 `false`로 처리한다. |
| `items` | 붙일 asset 목록이다. 같은 `position`에서는 배열 순서를 유지한다. |
| `items[].name` | composition item 이름이다. 예: `intro`, `slate`, `outro`. |
| `items[].position` | `before`이면 본편 앞, `after`이면 본편 뒤에 배치한다. |
| `items[].source.type` | `image` 또는 `video`를 사용한다. |
| `items[].source.uri` | 원본 asset 위치이다. playlist에는 이 URI를 직접 기록하지 않는다. |
| `items[].duration` | image 재생 길이 또는 video 사용 길이이다. |

Composition asset은 대상 rendition과 같은 container, codec, audio, resolution/sizing 조건에 맞춰 HLS segment로 제공된다.
fMP4에서 composition 구간과 본편 구간의 init segment가 다르면 `EXT-X-MAP`을 다시 기록하고, 구간 경계에는 `EXT-X-DISCONTINUITY`가 기록될 수 있다.

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:4

#EXT-X-MAP:URI="intro_init.mp4"
#EXTINF:3.000,
intro_000.m4s

#EXT-X-DISCONTINUITY
#EXT-X-MAP:URI="main_init.mp4"
#EXTINF:2.000,
main_000.m4s

#EXT-X-DISCONTINUITY
#EXT-X-MAP:URI="outro_init.mp4"
#EXTINF:4.000,
outro_000.m4s

#EXT-X-ENDLIST
```
