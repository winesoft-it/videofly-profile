# Profile 옵션 변환 가이드

이 문서는 VideoFly profile v1의 profile/rendition 값이 이 프로젝트에서 변환 옵션과 HLS 처리 정책으로 해석되는 기준을 정리한다.
목표는 FFmpeg 전체 옵션 목록을 만드는 것이 아니라, schema에 노출된 값이 어떤 FFmpeg 옵션 또는 비FFmpeg 처리로 이어지는지 정의하는 것이다.

## 기본 원칙

- Profile schema는 작성자가 직접 제어할 값만 가진다.
- 이 문서는 실제 구현의 코드 구조나 함수 호출 순서를 정의하지 않는다.
- Profile에 없는 동적 옵션은 변환 구현이 원본 probe, job/segment 정보, codec/container, 실행 장비 기준으로 계산한다.
- Profile 계약 제외 항목의 세부 계산식은 이 문서에 나열하지 않는다. Profile 계약에 영향을 주는 경우에만 이 문서에 추가한다.

단위 규칙:

| Profile 값 | 단위 | 변환 처리 |
| --- | --- | --- |
| `bit_rate` | bps | kbps 정수로 변환 |
| `max_bitrate`, `min_bitrate` | bps | kbps 정수로 변환 |
| `sample_rate` | Hz | 그대로 전달 |
| `frame_rate`, `max_frame_rate` | fps | 그대로 전달 |
| `hls.segment_duration` | sec | HLS segment 시간 정책으로 사용 |

## Profile 처리 정책

아래 항목은 FFmpeg option으로 직접 전달하지 않고, 출력 생성과 playlist 구성 과정의 처리 기준으로 사용한다.
rendition 생성 조건, HLS segment/playlist 정책, live 재생 정책, 암호화 정책이 여기에 해당한다.
`schema_version`, `name`, `description`은 profile 식별/문서화용 메타데이터이며 FFmpeg option으로 전달하지 않는다.
`renditions[]`는 출력 목록이며, 각 항목에 아래 HLS/Audio/Video/Sizing 매핑을 적용한다.

| Profile field | 처리 |
| --- | --- |
| `rendition_policy.mode` | 어떤 rendition을 만들지 결정하는 변환 정책이다. FFmpeg option으로 직접 전달하지 않는다. |
| `rendition_policy.allow_upscale` | rendition 생성 여부 또는 sizing policy 선택에 영향을 줄 수 있다. 직접 FFmpeg option은 아니다. |
| `live.*` | live 요청 정책이다. FFmpeg option으로 직접 전달하지 않는다. |
| `encrypt.*` | HLS `EXT-X-KEY`/playlist 암호화 정책이다. FFmpeg 실행 옵션이 아니라 playlist 생성 단계에서 처리한다. |
| `event.*` | 이벤트/정규화 정책이다. FFmpeg option으로 직접 전달하지 않는다. |

## HLS 기본 처리

HLS 출력은 rendition의 `container`와 profile의 `hls` 정책을 기준으로 구성한다.
아래 항목은 profile 값과 구현 기본값이 FFmpeg HLS muxer 옵션 또는 playlist/segment 처리로 이어지는 방식을 설명한다.

| Profile 기준 | FFmpeg option | 처리 |
| --- | --- | --- |
| HLS rendition 생성 | `-f hls` | HLS muxer를 사용한다. |
| `container = fmp4` | `-hls_segment_type fmp4` | fMP4 segment를 생성한다. |
| `container = m2ts` | `-hls_segment_type mpegts` | MPEG-TS segment를 생성한다. |
| `container = fmp4` | `-hls_fmp4_init_filename <init 파일명>` | fMP4 init 파일명을 구성한다. |
| `hls.version` | 없음 | playlist 버전 또는 HLS 호환성 정책으로 사용한다. |
| `hls.segment_duration` | 없음 | 목표 segment 길이 정책으로 사용한다. 실제 segment 길이는 서비스의 segment 처리에서 맞춘다. |
| `hls.path` | 없음 | playlist에 기록되는 segment 경로 표현 방식을 결정한다. |
| `hls.split` | 없음 | segment 분할 정책으로 사용한다. |
| HLS muxer segment time | `-hls_time 36000` | 기본 FFmpeg 출력은 단일 요청 output 생성을 기준으로 구성한다. profile 작성자가 직접 지정하는 값은 아니다. |

타임스탬프 보정, stream mapping, mux 지연, B-frame 같은 세부 FFmpeg 옵션은 profile 계약으로 고정하지 않는다.
이 값들은 구현이 입력 원본, 출력 container, 서비스 정책에 맞게 결정한다.

## Audio 매핑

기준 경로: `profile.renditions[].audio`

| Profile field | FFmpeg option |
| --- | --- |
| `enable = false` | `-an` |
| `codec = aac` | `-acodec aac` |
| `codec = ac3` | `-acodec ac3` |
| `codec = eac3` | `-acodec eac3` |
| `codec = opus` | `-acodec libopus` |
| `codec_options.profile = aac_low` | `-aprofile aac_low` |
| `codec_options.profile = aac_he` | `-aprofile aac_he` |
| `codec_options.profile = aac_he_v2` | `-aprofile aac_he_v2` |
| `codec_options.profile = auto` | 생략 |
| `sample_rate` | `-ar <Hz>` |
| `bit_rate` | `-b:a <kbps>k` |
| `channels` | `-ac <channels>` |

`sample_rate`가 없을 때 별도 기본값이 필요하면 구현 기본값으로 `48000`을 사용할 수 있다.

## Video 매핑

기준 경로: `profile.renditions[].video`

| Profile field | FFmpeg option |
| --- | --- |
| `enable = false` | `-vn` |
| `codec = h264` + CPU 인코더 | `-vcodec libx264` |
| `codec = h264` + GPU 인코더 | `-vcodec h264_nvenc` |
| `codec = av1` + CPU 인코더 | `-vcodec libsvtav1` |
| `codec = av1` + GPU 인코더 | `-vcodec av1_nvenc` |
| `codec_options.profile` | `-vprofile <value>` |
| `codec_options.level` | `-vlevel <value>` |
| `codec_options.max_bitrate` | `-maxrate <kbps>k`, `-bufsize <kbps>k` |
| `codec_options.min_bitrate` | `-minrate <kbps>k` |
| `codec_options.chroma_subsampling = auto` | 생략 |
| `codec_options.chroma_subsampling` | `-pix_fmt <value>` |
| `bit_rate` | `-b:v <kbps>k` |
| `frame_rate` | video filter graph에 `fps=<value>` 추가 |
| `max_frame_rate` | `-fpsmax <value>` |
| `resolution` | `sizing_policy`와 함께 Sizing/Padding filter를 구성한다. |

`codec`은 출력 코덱을 의미하며, 실제 encoder 라이브러리는 CPU/GPU 실행 경로에 따라 선택한다.
GPU 인코더는 현재 `gpu.preset`의 `p1`-`p7` 체계를 기준으로 NVENC 계열 encoder를 사용한다.

`frame_rate`, `resolution`, `sizing_policy`, `padding_policy`처럼 video filter graph에 영향을 주는 값은 하나의 `-vf` filter chain으로 조합한다.
동일 출력에 `-vf`를 여러 번 생성해 마지막 값만 적용되는 형태로 만들지 않는다.

`gpu`와 `cpu` 블록은 변환 구현이 실행 환경에 따라 하나를 선택한 뒤 FFmpeg 옵션으로 변환한다.

| Profile field | 변환 처리 |
| --- | --- |
| `gpu.preset` | GPU encoder 사용 시 `-preset <value>`로 변환한다. |
| `gpu.quality` | GPU encoder 사용 시 `-cq <value>` 또는 codec별 quality 옵션으로 변환한다. |
| `cpu.preset` | CPU encoder 사용 시 `-preset <value>`로 변환한다. |
| `cpu.quality` | CPU encoder 사용 시 `-crf <value>`로 변환한다. |

Quality 값은 bitrate보다 우선하는 운영 기본 정책이다.
`bit_rate`, `max_bitrate`, `min_bitrate`는 필요한 경우에만 사용하는 override 성격의 optional 값이다.

## Sizing/Padding 매핑

기준 경로: `profile.renditions[].video`

`resolution`은 `{width}x{height}` 형식이며, `sizing_policy`와 함께 scale/crop/pad filter를 결정한다.
운영 기본 profile은 `sizing_policy = fit_to_orientation_no_upscale`을 사용한다.
CPU 인코더 경로는 `scale`을 사용하고, GPU 인코더 경로는 `scale_npp`를 사용한다.
`CPU filter`, `GPU filter`는 sizing/padding 단계에서 사용할 filter chain이다.
`W`, `H`는 `resolution`의 목표 width/height이다.
`OW`, `OH`는 원본 orientation을 반영한 목표값이며, 원본이 세로이면 `OW=H`, `OH=W`, 아니면 `OW=W`, `OH=H`로 계산한다.
`padding_policy`의 기본 동작은 `no_pad`이며, `pad`일 때만 `pad_filter`를 마지막에 추가한다.

| `sizing_policy` | 처리 | CPU filter | GPU filter | `pad_filter` |
| --- | --- | --- | --- | --- |
| `fit` | 비율 유지, 목표 박스 안에 맞춤. | `scale=W:H:force_original_aspect_ratio=decrease,crop=trunc(iw/2)*2:trunc(ih/2)*2` | `scale_npp=w=W:h=H:force_original_aspect_ratio=decrease:interp_algo=super:format=nv12,scale_npp=w=trunc(iw/2)*2:h=trunc(ih/2)*2:format=nv12` | `pad=W:H:(ow-iw)/2:(oh-ih)/2` |
| `fit_to_orientation` | 원본이 세로이면 width/height 기준을 바꾼 뒤 `fit` 적용. | `scale=OW:OH:force_original_aspect_ratio=decrease,crop=trunc(iw/2)*2:trunc(ih/2)*2` | `scale_npp=w=OW:h=OH:force_original_aspect_ratio=decrease:interp_algo=super:format=nv12,scale_npp=w=trunc(iw/2)*2:h=trunc(ih/2)*2:format=nv12` | `pad=OW:OH:(ow-iw)/2:(oh-ih)/2` |
| `fill` | 비율 유지, 목표 박스를 채운 뒤 중앙 crop. | `scale=W:H:force_original_aspect_ratio=increase,crop=W:H:(iw-W)/2:(ih-H)/2` | `scale_npp=w=W:h=H:force_original_aspect_ratio=increase:interp_algo=super:format=nv12,crop=W:H:(iw-W)/2:(ih-H)/2` | 없음 |
| `stretch` | 비율 유지 없이 목표 해상도로 scale. | `scale=W:H` | `scale_npp=w=W:h=H:interp_algo=super:format=nv12` | 없음 |
| `center_crop` | scale 없이 목표 박스 범위로 중앙 crop. | `crop='min(iw,W)':'min(ih,H)':(iw-min(iw,W))/2:(ih-min(ih,H))/2` | `crop='min(iw,W)':'min(ih,H)':(iw-min(iw,W))/2:(ih-min(ih,H))/2,scale_npp=w=trunc(iw/2)*2:h=trunc(ih/2)*2:format=nv12` | `pad=W:H:(ow-iw)/2:(oh-ih)/2` |
| `fit_no_upscale` | 원본보다 키우지 않고 목표 박스 안에 맞춤. | `scale='min(W,iw)':'min(H,ih)':force_original_aspect_ratio=decrease,crop=trunc(iw/2)*2:trunc(ih/2)*2` | `scale_npp=w='min(W,iw)':h='min(H,ih)':force_original_aspect_ratio=decrease:interp_algo=super:format=nv12,scale_npp=w=trunc(iw/2)*2:h=trunc(ih/2)*2:format=nv12` | `pad=W:H:(ow-iw)/2:(oh-ih)/2` |
| `fill_no_upscale` | 원본보다 키우지 않고 목표 박스를 채운 뒤 중앙 crop. | `scale='trunc(iw*min(max(W/iw,H/ih),1)/2)*2':'trunc(ih*min(max(W/iw,H/ih),1)/2)*2',crop='min(W,iw)':'min(H,ih)':(iw-min(W,iw))/2:(ih-min(H,ih))/2` | `scale_npp=w='trunc(iw*min(max(W/iw,H/ih),1)/2)*2':h='trunc(ih*min(max(W/iw,H/ih),1)/2)*2':interp_algo=super:format=nv12,crop='min(W,iw)':'min(H,ih)':(iw-min(W,iw))/2:(ih-min(H,ih))/2` | `pad=W:H:(ow-iw)/2:(oh-ih)/2` |
| `fit_to_orientation_no_upscale` | 원본이 세로이면 width/height 기준을 바꾸고, 원본보다 키우지 않는 `fit` 적용. | `scale='min(OW,iw)':'min(OH,ih)':force_original_aspect_ratio=decrease,crop=trunc(iw/2)*2:trunc(ih/2)*2` | `scale_npp=w='min(OW,iw)':h='min(OH,ih)':force_original_aspect_ratio=decrease:interp_algo=super:format=nv12,scale_npp=w=trunc(iw/2)*2:h=trunc(ih/2)*2:format=nv12` | `pad=OW:OH:(ow-iw)/2:(oh-ih)/2` |

## Profile 계약 제외 항목

다음 항목은 profile/rendition schema에 포함하지 않는다.
profile 작성자가 직접 지정하는 값이 아니라, 변환 구현이 실행 시점에 결정하는 값이다.

| 제외 항목 |
| --- |
| segment trim 범위 |
| PTS offset/보정 |
| source timestamp/timebase 보정 |
| AAC frame/sample 보정 |
| fMP4 init/segment patch |
| MPEG-TS segment patch |
| GPU hardware frame/CUDA 입력 구성 |
| tile-columns |
| conditional CBR |
| keyframe/GOP 보정 |

위 항목을 profile에서 직접 제어해야 하는 요구사항이 생기면 schema 필드 추가 여부를 먼저 검토한 뒤 이 문서에 매핑을 추가한다.

## 옵션 추가 규칙

새 FFmpeg 옵션을 추가할 때는 다음 기준으로 관리한다.

| 상황 | 처리 |
| --- | --- |
| profile 작성자가 직접 제어해야 함 | schema에 필드를 추가하고 이 문서에 FFmpeg 매핑을 추가한다. |
| 항상 변환 구현이 계산해야 함 | schema에 추가하지 않고 이 문서의 Profile 계약 제외 항목 또는 기본 동작에만 언급한다. |
| 내부 구현 전용임 | schema와 이 문서에 추가하지 않는다. 코드 또는 구현 문서에서 관리한다. |
| sample/default에 항상 필요한 운영값임 | 관련 운영 profile과 example 파일에 일괄 반영한다. |
| optional이며 대부분 생략함 | schema에는 열어두고 관련 sample/default에는 넣지 않는다. 생략 시 동작만 이 문서에 적는다. |
