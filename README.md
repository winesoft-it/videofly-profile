# videofly-profile

VideoFly에서 사용하는 비디오 프로파일 scheme을 관리하는 저장소입니다.

향후 프로파일 정책과 버전별 JSON Schema, 예제 파일을 함께 관리합니다.

## 구조

```text
schemes/
  v1/
    schemas/
      profile.schema.json
      rendition.schema.json
    examples/
      profile.default.json
      profile.sample.json
    docs/
      profile-guide.md
```

`{version}`은 `v1`, `v2`처럼 표기합니다.

## 구성

- `schemas/`: 해당 버전의 JSON Schema 파일을 둡니다.
- `examples/`: 해당 버전의 기본값 및 샘플 profile 파일을 둡니다.
- `docs/`: 해당 버전의 profile 작성 가이드 문서를 둡니다.

## 주요 개념

- `profile`: VideoFly에서 사용할 비디오 출력 설정 묶음입니다.
- `rendition`: profile 안에 포함되는 개별 출력 단위입니다. 해상도, 컨테이너, 오디오/비디오 설정 등을 가집니다.

## 버전 관리

scheme은 `schemes/{version}` 형태로 확장하며, 이 구조를 VideoFly profile의 버전 정책 기준으로 사용합니다.
