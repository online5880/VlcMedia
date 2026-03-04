# [UE5.6] 언리얼 엔진에서 RTSP(CCTV) 실시간 재생하기 (VlcMedia 플러그인 설정 가이드)

## 들어가며
언리얼 엔진 5.6 환경에서 공공 CCTV 데이터와 같은 RTSP 스트림을 실시간으로 재생해야 하는 경우가 있습니다.
기본 Media Framework만으로는 RTSP 처리가 불안정할 수 있어, VlcMedia 플러그인을 커스터마이징하여 통합하는 방법을 정리했습니다.

이 글은 LibVLC 바이너리를 플러그인에 내장하여 별도의 설치 없이 배포 가능한 구성을 기준으로 작성되었습니다.

---

## 1. 개발 환경 및 준비물

### 개발 환경
- Engine Version: Unreal Engine 5.6.x
- IDE: Visual Studio (MSVC Toolchain)
- Plugin: VlcMedia (GitHub)

참고: 원본 리포지토리를 클론 후 프로젝트 플러그인(Plugins/) 폴더에 포함하여 커스터마이징 사용

### 테스트용 RTSP 주소
- URL: `rtsp://210.99.70.120:1935/live/cctv001.stream`
- 출처: [공공데이터포털 - 충청남도 천안시_교통정보 CCTV](https://www.data.go.kr/data/15063717/fileData.do)

---

## 2. LibVLC 라이브러리 구성 (핵심)

사용자가 VLC 플레이어를 별도로 설치하지 않아도 되도록, 플러그인 내부에 바이너리를 심는 방식입니다.

- LibVLC 다운로드: 3.0.20 (Win64 .zip) 권장
- 경로 배치: `Plugins/VlcMedia-master/ThirdParty/vlc/Win64`

### 설정 포인트
- UE5 렌더링 파이프라인과의 호환성을 위해 vmem 출력 강제
- 안정성을 위해 HW 디코더 비활성화

---

## 3. 에디터 내 에셋 구성

C++ 코드에서 모든 것을 생성하기보다, 에디터에서 생성한 에셋을 레퍼런스로 사용하는 것이 관리 측면에서 유리합니다.

1) Media Player 생성  
2) 생성 시 "Video Output Media Texture asset" 체크  
3) Material 생성  
   - Media Texture를 사용하는 머티리얼  
   - Texture Sample 파라미터 이름을 `Texture`로 지정

---

## 4. 액터(CCTVStreamActor) 구현 및 설정

### 주요 프로퍼티

| 프로퍼티 | 설명 | 예시 값 |
| --- | --- | --- |
| RtspUrl | 재생할 스트림 주소 | `rtsp://...` |
| OverridePlayerName | 플레이어 모듈 이름 | `VlcMedia` |
| bUseTCP | TCP 연결 강제 여부 | `false` |

### TCP 연결 강제 (bUseTCP)
RTSP는 기본적으로 UDP를 시도하지만, 방화벽이나 네트워크 환경에 따라 패킷 손실이 발생할 수 있습니다.
URL 뒤에 옵션을 붙여 TCP로 강제하는 로직을 추가하는 것이 좋습니다.

```cpp
FString FinalUrl = RtspUrl;
if (bUseTCP)
{
    // VlcMedia가 인식하는 옵션 포맷에 맞춰 추가 (?transport=tcp)
    FinalUrl += TEXT("?transport=tcp");
}
```

### 액터 세팅 순서
1) 월드에 CCTVStreamActor 배치  
2) MediaPlayerAsset 할당 (앞서 만든 에셋)  
3) MediaTextureAsset 할당 (앞서 만든 에셋)  
4) BaseMaterial 할당 (Texture 파라미터가 있는 머티리얼)

---

## 5. 트러블슈팅 (화면이 안 나올 때)

소리는 들리거나 플레이어 상태는 Playing인데 머티리얼이 검정색일 때 체크리스트입니다.

1) 에셋 연결 확인  
   - MediaTexture가 현재 재생 중인 MediaPlayer와 올바르게 연결되어 있는지 확인
2) 파라미터 이름 확인  
   - 머티리얼의 텍스처 파라미터 이름이 코드에서 호출하는 이름(`Texture`)과 일치하는지 확인
3) 콜백 로그 확인  
   - `StaticVideoDisplayCallback` 로그가 찍히지 않으면 디코딩 실패 또는 vmem 설정 문제 가능성

---

## 6. 원본 VlcMedia 대비 수정 사항

원본 플러그인(ue4plugins/VlcMedia)을 그대로 쓰면 UE 5.6과 RTSP 재생에서 문제가 발생해 아래 부분을 커스터마이징했습니다.

### 1) UE 5.6 호환 빌드 수정
- UE 5.6 API 변경사항 반영 (Build.cs, uplugin 등)
- 최신 인터페이스 요구사항에 맞춰 일부 함수 시그니처 정리
- 기존 플러그인 uplugin의 `EngineVersion` 값은 `"4.20.0"` 기준

### 2) LibVLC 초기화 옵션 강화
- HW 디코더 비활성화 (`--avcodec-hw=none`, `--no-hw-decoder`)
- vmem 출력 강제 (소프트웨어 프레임 콜백 경로 확보)
- 네트워크 캐싱/클록 관련 안정성 옵션 추가

### 3) RTSP 연결 안정성 개선
- TCP 전송 옵션 처리 (`?transport=tcp` 또는 미디어 옵션 추가)
- URL 옵션 처리 로직 보완

### 4) 에디터 에셋 우선 사용
- 에디터에서 생성한 MediaPlayer/MediaTexture를 우선 사용하도록 액터 로직 정리
- 런타임 생성 시 발생하는 연결 불일치 문제 회피

---

## 7. 버전 호환성

| LibVLC 버전 | UE 5.6 호환성 | 비고 |
| --- | --- | --- |
| 3.0.20 | ✅ 완전 지원 | 현재 프로젝트 버전 |
| 3.0.18-19 | ✅ 지원 가능 | vmem API 호환 |
| 4.x | ❌ 미지원 | vmem API 변경으로 인해 수정 필요 |

**참고**: LibVLC 4.0부터는 video memory callback 방식이 변경되어 현재 플러그인 코드 수정 필요

---

## 8. 성능 최적화 팁

### 다중 스트림 관리
```cpp
// 스트림 개수 제한 (메모리 사용량 기준)
#define MAX_CONCURRENT_STREAMS 8

// 재생하지 않는 스트림 일시정지
void CCTVStreamActor::OptimizeResourceUsage()
{
    if (MediaPlayer && !bIsActive)
    {
        MediaPlayer->Pause();
    }
}
```

### Media Texture 최적화
- **해상도**: 1080p 권장 (4K는 과도한 메모리 사용)
- **포맷**: RGBA8 사용 (압축 없이 직접 표현)
- **업데이트 주기**: 필요시 30fps로 제한

### 메모리 관리
```cpp
// 프레임 버퍼 크기 조정
--vmem-width=1920 --vmem-height=1080
```

---

## 9. 보안 고려사항

### 인증 처리
```cpp
// RTSP URL에 인증 정보 포함 (테스트용만 권장)
FString AuthenticatedUrl = FString::Printf(
    TEXT("rtsp://%s:%s@%s:%d%s"),
    *Username, *Password, *Host, Port, *Path
);

// 보안: 프로덕션에서는 쿠키/토큰 방식 사용 권장
```

### 네트워크 방화벽 설정
- **필수 포트**: 554 (RTSP 기본), 1935 (RTMP)
- **TCP 대역**: 보안 정책에 따른 포트 범위 허용
- **TLS**: 안정성을 위해 `rtsps://` 사용 고려

### 데이터 암호화
```cpp
// TLS/SSL 적용 시
FinalUrl += TEXT("?transport=tcp&x-key=&x-cert=");
```

---

## 10. 디버깅 및 모니터링

### UE5 콘솔 변수
```bash
# VlcMedia 디버깅
vt.VlcMedia.LogLevel 4
vt.VlcMedia.ShowStats 1

# Media Framework 디버깅
Media.LogLevel Verbose
Media.Player.ShowTime 1
```

### 스트림 상태 모니터링
```cpp
// CCTVStreamActor.h
UFUNCTION(BlueprintCallable)
bool IsStreamHealthy() const;

UFUNCTION(BlueprintCallable)
float GetCurrentBitrate() const;

// CCTVStreamActor.cpp
bool UCCTVStreamActor::IsStreamHealthy() const
{
    if (!MediaPlayer) return false;
    
    return MediaPlayer->IsPlaying() && 
           LastFrameTime > (FDateTime::Now() - FTimespan::FromSeconds(3.0));
}
```

### 상세 로깅
```cpp
// 스트림 상태 변화 감지
void UCCTVStreamActor::OnMediaOpened(FString OpenedUrl)
{
    UE_LOG(LogTemp, Log, TEXT("RTSP Stream Opened: %s"), *OpenedUrl);
}

void UCCTVStreamActor::OnMediaEvent(EMediaEvent Event)
{
    switch (Event)
    {
        case EMediaEvent::PlaybackBuffering:
            UE_LOG(LogTemp, Warning, TEXT("RTSP Buffering..."));
            break;
        case EMediaEvent::PlaybackEnd:
            UE_LOG(LogTemp, Log, TEXT("RTSP Stream Ended"));
            break;
    }
}
```

---

## 11. 배포 환경 테스트 케이스

### 기본 테스트 시나리오
1. **단일 스트림**: 1개 RTSP URL 24시간 연속 재생
2. **다중 스트림**: 4개 스트림 동시 재생 (메모리/성능)
3. **네트워크 장애**: 연결 끊김 후 자동 복구 테스트
4. **리소스 제약**: 저사양 머신에서의 동작 확인

### 검증 항목 체크리스트
- [ ] 스트림 연결 시간 < 3초
- [ ] 재생 중 크래시 없음
- [ ] 메모리 누수 없음 (24시간 테스트)
- [ ] 네트워크 재연결 정상 동작
- [ ] 패키징 후 정상 동작

---

## 12. 에러 코드별 대응표

| 증상 | 에러 코드 | 원인 | 해결책 |
| --- | --- | --- | --- |
| 검은 화면 | VLC_EGENERIC | vmem 출력 실패 | `--avcodec-hw=none` 추가 |
| 연결 타임아웃 | VLC_ETIMEDOUT | 방화벽/네트워크 | `?transport=tcp` 추가 |
| 디코딩 실패 | VLC_EVIDEO_DECODER | 코덱 부족 | `--codec=avcodec` 추가 |
| 소리만 나옴 | VLC_EVIDEO_OUTPUT | 비디오 출력 설정 | vmem 설정 재확인 |
| 자주 끊김 | VLC_ENOITEM | 버퍼 크기 부족 | `--network-caching=1000` 증가 |

---

## 13. 권장 라이브러리 옵션

```cpp
// 최종 권장 LibVLC 초기화 옵션
FString VlcOptions = TEXT(
    "--no-xlib "
    "--quiet "
    "--avcodec-hw=none "
    "--no-hw-decoder "
    "--vout=vmem "
    "--vmem-width=1920 "
    "--vmem-height=1080 "
    "--network-caching=1000 "
    "--clock-jitter=50 "
    "--clock-synchro=1 "
);
```

---

## 마무리

UE 5.6에서도 VlcMedia를 활용하면 RTSP CCTV 스트림을 안정적으로 가져올 수 있습니다.
특히 LibVLC 바이너리 포함과 TCP 강제 옵션은 배포 시 발생할 수 있는 많은 문제를 예방해 줍니다.

추가된 최적화, 보안, 모니터링 기능들을 활용하면 실제 운영 환경에서도 안정적인 CCTV 스트리밍을 구축할 수 있습니다.
