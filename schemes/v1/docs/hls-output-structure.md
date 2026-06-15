# HLS 산출물 구성 가이드

이 문서는 VideoFly profile v1 값에 따라 생성되는 HLS 산출물의 구성을 정리한다.
FFmpeg 옵션 매핑은 `ffmpeg-option-mapping.md`에서 다루며, 이 문서는 `master.m3u8`, `playlist.m3u8` 구성을 다룬다.

## 기본 원칙

- HLS 산출물은 profile의 `rendition_policy`, `renditions`, `hls`, `playout`, `live`, `encrypt` 값을 기준으로 구성한다.
- 원본 이미지 또는 영상 asset URI를 media playlist에 직접 넣지 않는다.
- playlist에는 대상 rendition의 container 형식에 맞는 HLS segment URI만 기록한다.
- 구현 내부의 probe, timestamp 보정, segment 파일명 계산식은 이 문서에 포함하지 않는다.

## Master Playlist (`master.m3u8`)

`master.m3u8`은 profile의 rendition 목록을 variant stream으로 노출한다.

| Profile 값 | 산출물 구성 |
| --- | --- |
| `hls.version` | `EXT-X-VERSION` 산출 기준으로 사용한다. |
| `rendition_policy.mode` | variant rendition을 포함하거나 제외하는 기준으로 사용한다. |
| `rendition_policy.allow_upscale` | 원본보다 큰 해상도의 rendition을 master playlist에 포함할지 결정하는 기준으로 사용한다. |
| `renditions[]` | 각 rendition을 하나의 variant stream으로 구성한다. |
| `renditions[]` 배열 순서 | master playlist의 variant stream 출력 순서로 사용할 수 있다. |
| `renditions[].name` | `EXT-X-STREAM-INF` 다음 줄에 기록할 media playlist URI 산출 기준으로 사용한다. |
| `renditions[].container` | variant가 참조하는 media playlist의 segment container를 결정한다. |
| `renditions[].video.resolution` | `EXT-X-STREAM-INF`의 `RESOLUTION` 산출 기준으로 사용한다. |
| `renditions[].video.codec`, `renditions[].video.codec_options.profile`, `renditions[].video.codec_options.level`, `renditions[].audio.codec`, `renditions[].audio.codec_options.profile` | `EXT-X-STREAM-INF`의 `CODECS` 산출 기준으로 사용한다. |
| `renditions[].bandwidth` | `BANDWIDTH`를 동적으로 산출할 수 없을 때 fallback 값으로 사용한다. |
| `renditions[].score` | `EXT-X-STREAM-INF`의 `SCORE` 산출 기준으로 사용한다. |

`EXT-X-STREAM-INF`의 `BANDWIDTH`는 실제 출력 또는 probe 결과를 기준으로 동적으로 산출한다.
동적 산출이 불가능한 경우에만 `renditions[].bandwidth` 값을 사용한다.
`EXT-X-STREAM-INF`의 `SCORE`는 variant stream의 상대적인 재생 품질 점수로 사용한다.
`renditions[].score`가 있으면 해당 값을 우선 사용하고, 없으면 구현에서 산출한 값을 사용한다.
`SCORE`를 기록하는 경우 master playlist의 모든 variant stream에 기록되도록 구성한다.

`rendition_policy.mode` 값별 variant rendition 포함/제외 기준:

| `rendition_policy.mode` | 기준 |
| --- | --- |
| `always` | 항상 variant rendition에 포함한다. |
| `if_bitrate_exceeds` | 원본 codec/resolution이 rendition과 같고, 원본 bitrate가 rendition bitrate와 같거나 낮으면 variant rendition에 포함하지 않는다. |
| `if_spec_mismatch` | 원본 codec/resolution이 rendition과 같으면 variant rendition에 포함하지 않는다. |
| `if_codec_mismatch` | 원본 codec이 rendition과 같으면 variant rendition에 포함하지 않는다. |

`rendition_policy.allow_upscale = false`이면 원본 해상도보다 큰 rendition은 master playlist variant에서 제외한다.

`master.m3u8` 산출 예시:

예시의 media playlist URI는 설명용이며, 실제 경로 규칙을 정의하지 않는다.

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720,CODECS="av01.0.05M.08,mp4a.40.2",SCORE=2
fmp4_720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=900000,RESOLUTION=640x360,CODECS="av01.0.04M.08,mp4a.40.2",SCORE=1
fmp4_360p/playlist.m3u8
```

## Media Playlist (`playlist.m3u8`)

Media playlist는 각 rendition마다 생성한다.
`playout`이 있으면 before 항목, 본편 항목, after 항목 순서로 segment URI를 구성한다.

| Profile 값 | 산출물 구성 |
| --- | --- |
| `hls.version` | `EXT-X-VERSION` 산출 기준으로 사용한다. |
| `hls.segment_duration` | 목표 segment 길이로 사용한다. `EXT-X-TARGETDURATION`은 실제 segment 길이들을 기준으로 산출하고, `EXTINF`에는 각 segment의 실제 재생 길이를 기록한다. |
| `hls.path = abs` | media playlist에 절대 경로 또는 절대 URL 형태의 segment URI를 기록한다. |
| `hls.path = rel` | media playlist에 상대 경로 형태의 segment URI를 기록한다. |
| `hls.split` | 본편 segment의 분할 방식으로 사용한다. `EXTINF` 값을 직접 지정하지 않는다. |
| `renditions[].container = fmp4` | media playlist에 fMP4 init segment와 media segment를 기록한다. |
| `renditions[].container = m2ts` | media playlist에 MPEG-TS segment를 기록한다. |
| `encrypt.enable = true` | media playlist에 `EXT-X-KEY`를 기록하고 media segment 암호화를 적용한다. |
| `encrypt.key_file_name` | `EXT-X-KEY`의 key URI 산출 기준으로 사용한다. |
| `encrypt.iv` | `EXT-X-KEY`의 IV 산출 기준으로 사용한다. |
| `live.enable = true` | 요청 URL의 live `start`/`end` 입력값과 현재 시각 기준으로 노출 가능한 segment만 media playlist에 기록한다. |
| `playout.items[].position = before` | 본편 첫 segment 앞에 playout segment를 배열 순서대로 기록한다. |
| `playout.items[].position = after` | 본편 마지막 segment 뒤에 playout segment를 배열 순서대로 기록한다. |

`playlist.m3u8` 대표 산출 예시:

아래 예시는 VOD/fMP4 media playlist 형태를 보여준다. URI는 설명용이며, 실제 경로 규칙을 정의하지 않는다.

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

`hls.path` 값별 segment URI 산출 기준:

| `hls.path` | media playlist segment URI |
| --- | --- |
| `abs` | 절대 경로 또는 절대 URL 형태로 기록한다. 예: `/hls/v1/720p/seg_000.m4s`, `https://example.com/hls/720p/seg_000.m4s` |
| `rel` | media playlist 위치 기준 상대 경로로 기록한다. 예: `seg_000.m4s`, `../common/key.key` |

`hls.split` 값별 segment 분할 기준:

| `hls.split` | segment 분할 기준 |
| --- | --- |
| `loose` | 원본 키프레임을 위배하지 않는 범위에서 segment를 분할한다. |
| `balance` | `hls.segment_duration` 기준 +/- 1초 범위에서 가능한 적게 segment를 분할한다. |
| `strict` | `hls.segment_duration` 단위에 맞춰 segment를 분할한다. |

`container` 값별 media playlist 산출 기준:

| `renditions[].container` | media playlist 구성 |
| --- | --- |
| `fmp4` | `EXT-X-MAP`으로 init segment를 기록하고 fMP4 media segment URI를 기록한다. |
| `m2ts` | MPEG-TS media segment URI를 기록한다. fMP4 init segment는 사용하지 않는다. |

`encrypt` 값별 산출 기준:

| Profile 값 | 산출물 구성 |
| --- | --- |
| `encrypt.enable = false` | `EXT-X-KEY`를 기록하지 않고 암호화되지 않은 segment를 사용한다. |
| `encrypt.enable = true` | `EXT-X-KEY`를 기록하고 암호화된 segment를 사용한다. |
| `encrypt.key_file_name` | `EXT-X-KEY`의 `URI` 산출 기준으로 사용한다. |
| `encrypt.token`, `encrypt.token_type` | key 파일 또는 key material 산출 기준으로 사용한다. media playlist에는 직접 기록하지 않는다. |
| `encrypt.iv` | 값이 있으면 `EXT-X-KEY`의 `IV` 산출 기준으로 사용한다. |

예: `#EXT-X-KEY:METHOD=AES-128,URI="enc.key",IV=0x0123456789abcdef0123456789abcdef`

`live` 값별 산출 기준:

| 기준 값 | 산출물 구성 |
| --- | --- |
| `live.enable = false` | 전체 본편 segment를 media playlist에 기록한다. |
| `live.enable = true` | 요청 URL의 live `start`/`end` 입력값과 현재 시각을 기준으로 노출 가능한 일부 segment만 media playlist에 기록한다. |
| 요청 URL `start` | live 노출 시작 시각이다. 현재 시각과 비교해 본편에서 사용할 시작 위치를 산출한다. |
| 요청 URL `end` | live 노출 종료 시각이다. 현재 시각이 종료 범위를 벗어난 경우 media playlist 산출 대상에서 제외할 수 있다. |
| `live.buffer` | 현재 시각 기준으로 media playlist에 남겨둘 segment window 범위 산출 기준으로 사용한다. |
| `live.denial_code` | HLS playlist 내용에는 기록하지 않는다. live 요청 거절 응답 정책으로 사용한다. |

`playout` 값별 media playlist 산출 기준:

| Profile 값 | 산출물 구성 |
| --- | --- |
| `playout.enable = false` | playout segment를 기록하지 않는다. |
| `playout.items[].position = before` | 같은 position의 배열 순서대로 본편 앞에 기록한다. |
| `playout.items[].position = after` | 같은 position의 배열 순서대로 본편 뒤에 기록한다. |
| `playout.items[].source.type = image` | image asset을 대상 rendition과 동일한 container, codec, audio, resolution/sizing 조건의 정지 영상 segment로 변환해 기록한다. |
| `playout.items[].source.type = video` | video asset을 대상 rendition과 동일한 container, codec, audio, resolution/sizing 조건의 video segment로 변환해 기록한다. |
| `playout.items[].source.uri` | 원본 asset 입력으로만 사용한다. media playlist에는 이 URI를 직접 기록하지 않는다. |
| `playout.items[].duration` | image segment 길이 또는 video asset 사용 길이의 기준으로 사용한다. |

`playout` 적용 `playlist.m3u8` 산출 예시:

playout이 before/main/after로 구성된 경우 media playlist는 `intro` 변환 segment, 본편 segment, `outro` 변환 segment 순서로 기록한다.
아래 예시는 fMP4에서 각 구간이 별도 init segment를 갖는 경우이다.
예시의 segment 이름은 설명용이며, 실제 파일명 규칙을 정의하지 않는다.

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
#EXTINF:2.000,
main_001.m4s

#EXT-X-DISCONTINUITY
#EXT-X-MAP:URI="outro_init.mp4"
#EXTINF:4.000,
outro_000.m4s

#EXT-X-ENDLIST
```

fMP4 media playlist에서 init segment가 바뀌는 구간에는 `EXT-X-MAP`을 다시 기록한다.
playout 구간과 본편 구간 사이에 codec, track, timestamp, init segment가 달라지는 경우에는 해당 경계에 `EXT-X-DISCONTINUITY`를 기록한다.
첫 segment 앞에는 이전 segment가 없으므로 `EXT-X-DISCONTINUITY`를 기록하지 않는다.
