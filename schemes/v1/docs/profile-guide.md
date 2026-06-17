# VideoFly Profile 설정 가이드

이 문서는 VideoFly profile v1을 직접 작성하는 사용자를 위한 설정 가이드이다.
Profile은 하나의 입력 영상을 어떤 HLS 출력으로 제공할지 정의한다.
출력 variant, HLS playlist, 오디오/비디오 인코딩, playout, live, encrypt 같은 동작은 profile 값으로 조정한다.

## Profile 기본 구조

Profile은 공통 설정과 선택적 `renditions`로 구성한다.
`renditions`는 실제로 제공할 출력 목록이며, 배열 순서는 `master.m3u8`의 variant stream 순서로 사용할 수 있다.
`renditions`를 생략하거나 빈 배열로 지정하면 원본 codec, resolution, bitrate를 유지하는 무변환 정책으로 처리한다.

```json
{
  "schema_version": "v1",
  "name": "_default",
  "description": "기본 VideoFly profile",
  "rendition_policy": {
    "mode": "if_spec_mismatch",
    "allow_upscale": false
  },
  "hls": {
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
        "resolution": "1280x720",
        "sizing_policy": "fit_to_orientation_no_upscale"
      }
    }
  ]
}
```

| 설정 | 설명 |
| --- | --- |
| `schema_version` | profile schema 버전이다. v1 profile은 `"v1"`을 사용한다. |
| `name` | profile 이름이다. 기본 profile은 `_default`, `_youtube`, `_balance`처럼 `_`로 시작할 수 있다. |
| `description` | profile 설명이다. 운영에는 영향을 주지 않는다. |
| `rendition_policy` | 원본이 이미 조건을 만족할 때 variant rendition을 포함하지 않을지 결정한다. |
| `hls` | HLS playlist와 segment 생성 방식을 설정한다. |
| `renditions` | 제공할 출력 rendition 목록이다. 생략하거나 빈 배열이면 원본 codec, resolution, bitrate를 유지한다. |

## 기본값 파일

`schemes/v1/examples/profile.default.json`은 profile v1의 기본값 기준 파일이다.
새 profile을 작성할 때는 이 파일을 기준으로 필요한 값을 변경한다.

비활성 기능은 `enable: false`로 명시되어 있으며, 실제 운영 profile에서는 필요한 기능만 활성화한다.

## Rendition 포함 정책

`rendition_policy`는 원본 영상을 기준으로 어떤 rendition을 variant로 포함할지 결정한다.
각 mode는 rendition별로 적용된다.

| 값 | 설명 |
| --- | --- |
| `always` | 항상 variant rendition에 포함한다. |
| `if_bitrate_exceeds` | 원본 codec/resolution이 rendition과 같고, 원본 bitrate가 rendition bitrate와 같거나 낮으면 포함하지 않는다. |
| `if_spec_mismatch` | 원본 codec/resolution이 rendition과 같으면 포함하지 않는다. |
| `if_codec_mismatch` | 원본 codec이 rendition과 같으면 포함하지 않는다. |

`allow_upscale`은 원본보다 큰 해상도의 rendition을 허용할지 결정한다.
`false`이면 원본보다 큰 해상도의 rendition은 포함하지 않는다.

## HLS 설정

`hls`는 모든 rendition이 공유하는 HLS playlist 설정이다.

| 설정 | 값 | 설명 |
| --- | --- | --- |
| `version` | integer | playlist에 기록할 `EXT-X-VERSION` 값이다. fMP4를 사용하면 보통 `7`을 사용한다. |
| `path` | `abs`, `rel` | playlist 안에 기록되는 segment URI 형식이다. |
| `split` | `loose`, `balance`, `strict` | segment 분할 방식이다. |
| `segment_duration` | number | 목표 segment 길이(초)이다. |

`path` 값별 동작:

| 값 | 동작 |
| --- | --- |
| `abs` | 절대 경로 또는 절대 URL 형태로 segment URI를 기록한다. |
| `rel` | playlist 위치 기준 상대 경로로 segment URI를 기록한다. |

`split` 값별 동작:

| 값 | 동작 |
| --- | --- |
| `loose` | 원본 키프레임을 위배하지 않는 범위에서 분할한다. |
| `balance` | `segment_duration` 기준 +/- 1초 범위에서 가능한 적게 분할한다. |
| `strict` | `segment_duration` 단위에 맞춰 분할한다. |

대표적인 fMP4 VOD playlist는 다음 형태로 생성된다.
아래 예시는 URI 구조를 단순화하기 위해 상대 경로로 표기한다.

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:2
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD

#EXT-X-MAP:URI="init.mp4"

#EXTINF:2.000000,
0.m4s
#EXTINF:2.000000,
1.m4s

#EXT-X-ENDLIST
```

## Rendition 설정

`renditions[]`의 각 항목은 하나의 출력 variant를 의미한다.
예를 들어 `fmp4_720p`, `fmp4_360p`, `m2ts_720p`처럼 container와 해상도 단위로 구성할 수 있다.
`renditions`를 생략하거나 빈 배열로 지정하면 별도 rendition을 만들지 않고 원본 codec, resolution, bitrate 기준으로 출력한다.

| 설정 | 설명 |
| --- | --- |
| `name` | rendition 이름이다. `{container}_{name}` 형태를 사용한다. 예: `fmp4_720p`. |
| `container` | HLS segment container이다. `fmp4` 또는 `m2ts`를 사용한다. |
| `bandwidth` | `master.m3u8`의 `BANDWIDTH` fallback 값이다. 보통은 생략한다. |
| `score` | `master.m3u8`의 `SCORE` 값이다. 명시하지 않으면 내부 산출 값을 사용한다. |
| `audio` | 오디오 출력 설정이다. |
| `video` | 비디오 출력 설정이다. |

`bandwidth`는 실제 출력 또는 분석 결과로 `BANDWIDTH`를 얻을 수 없을 때만 사용하는 fallback이다.
`score`는 variant stream의 상대적인 재생 품질 점수이다. `SCORE`를 기록하는 경우 모든 variant stream에 함께 기록되도록 구성한다.

`master.m3u8`는 다음과 같은 형태로 생성된다.

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720,CODECS="av01.0.05M.08,mp4a.40.2",SCORE=2
fmp4_720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=900000,RESOLUTION=640x360,CODECS="av01.0.04M.08,mp4a.40.2",SCORE=1
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

## Sizing/Padding 설정

`sizing_policy`와 `padding_policy`는 원본 영상을 목표 해상도에 맞추는 방법을 결정한다.
기본 운영 profile은 `fit_to_orientation_no_upscale`을 사용한다.
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

## Playout 설정

`playout`은 원본 앞 또는 뒤에 이미지/영상 asset을 삽입할 때 사용한다.
광고, 안내 slate, intro/outro 영상처럼 본편과 함께 하나의 HLS playlist로 제공해야 하는 asset을 정의할 수 있다.

| 설정 | 설명 |
| --- | --- |
| `enable` | playout 사용 여부이다. 생략하면 `false`로 처리한다. |
| `items` | 삽입할 asset 목록이다. 같은 `position`에서는 배열 순서를 유지한다. |
| `items[].name` | playout item 이름이다. 예: `intro`, `slate`, `outro`. |
| `items[].position` | `before`이면 본편 앞, `after`이면 본편 뒤에 배치한다. |
| `items[].source.type` | `image` 또는 `video`를 사용한다. |
| `items[].source.uri` | 원본 asset 위치이다. playlist에는 이 URI를 직접 기록하지 않는다. |
| `items[].duration` | image 재생 길이 또는 video 사용 길이이다. |

Playout asset은 대상 rendition과 같은 container, codec, audio, resolution/sizing 조건에 맞춰 HLS segment로 제공된다.
fMP4에서 playout 구간과 본편 구간의 init segment가 다르면 `EXT-X-MAP`을 다시 기록하고, 구간 경계에는 `EXT-X-DISCONTINUITY`가 기록될 수 있다.

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

## Live 설정

`live`는 원본 파일을 live stream처럼 제공할 때 사용한다.
Live 노출 시작/종료 시각은 profile에 저장하지 않고 요청 URL의 `start`, `end` 입력값으로 전달된다.

| 설정 | 설명 |
| --- | --- |
| `enable` | live 방식 제공 여부이다. 생략하면 `false`로 처리한다. |
| `buffer` | 현재 시각 기준으로 playlist에 남겨둘 segment window 범위이다. |
| `denial_code` | 허용된 시간 범위를 벗어난 요청에 사용할 HTTP 응답 상태 코드이다. |

Live playlist는 현재 시각과 요청 URL의 `start`, `end` 값을 기준으로 노출 가능한 일부 segment만 기록한다.
진행 중인 live playlist는 VOD playlist와 달리 `EXT-X-ENDLIST`를 기록하지 않을 수 있다.

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

## Event 설정

`event`는 원본 비디오가 서비스될 때 발생시킬 profile-level 이벤트를 정의한다.
현재는 원본 포맷 표준화를 요청하는 `normalize` 값을 사용할 수 있다.

| 설정 | 값 | 설명 |
| --- | --- | --- |
| `normalize` | `to_h264_aac` | 원본을 H.264/AAC 기준으로 표준화하도록 요청한다. |

## 설정 예시

Profile 작성 예시는 `schemes/v1/examples/profile.sample.json`을 참고한다.
기본값 기준 profile은 `schemes/v1/examples/profile.default.json`을 참고한다.
