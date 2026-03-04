## [UE5.6] 언리얼 엔진에서 RTSP(CCTV) 실시간 재생하기 - VlcMedia 플러그인 완벽 가이드

> 언리얼 엔진 5.6에서 공공 CCTV RTSP 스트리밍을 구현하는 방법을 다룹니다. VlcMedia 플러그인을 활용한 실전 노하우와 최적화 팁을 공개합니다!

### 목차
1. [왜 UE5에서 RTSP가 필요한가?](#왜-ue5에서-rtsp가-필요한가)
2. [VlcMedia 플러그인 소개 및 설정](#vlcmedia-플러그인-소개-및-설정)
3. [실전 구현 - CCTVStreamActor 만들기](#실전-구현---cctvstreamactor-만들기)
4. [성능 최적화와 트러블슈팅](#성능-최적화와-트러블슈팅)
5. [배포 환경 고려사항](#배포-환경-고려사항)
6. [마무리하며](#마무리하며)

---

### VlcMedia 플러그인 소개 및 설정

### 🔧 플러그인 정보

**[ue4plugins/VlcMedia](https://github.com/ue4plugins/VlcMedia)**는 VideoLAN의 LibVLC를 UE5 Media Framework와 연동해주는 오픈소스 플러그인입니다.

- **별지수**: 282 stars, 147 forks
- **라이선스**: BSD-3-Clause
- **현재 버전**: 14.0 (UE5.6 호환성 커스터마이징 완료)
- **엔진 지원**: UE 4.19 기준 → UE 5.6 호환성 수정 적용

### UE5.6 호환성 수정

원본 플러그인은 UE 4.19 기준이라 UE5.6에서는 약간의 수정이 필요합니다:

```cpp
// Build.cs 수정 (실제 적용된 모듈 의존성)
PrivateDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "MediaUtils", "Projects", 
    "RenderCore", "VlcMediaFactory"
});

// DynamicallyLoadedModuleNames
DynamicallyLoadedModuleNames.AddRange(new string[] { "Media" });

// uplugin 엔진 버전 업데이트
"EngineVersion": "5.6.0"
"Version": "14.0"
```

### LibVLC 바이너리 설정

가장 중요한 부분입니다! 사용자가 VLC를 별도로 설치하지 않아도 되도록 플러그인에 LibVLC를 내장합니다.

**권장 버전**: LibVLC 3.0.20 (Win64, 현재 프로젝트 적용 완료)

```bash
# 파일 구조
Plugins/VlcMedia-master/
├── ThirdParty/vlc/Win64/
│   ├── libvlc.dll
│   ├── libvlccore.dll
│   └── plugins/
```

**주요 설정 포인트**:
- HW 디코더 비활성화: `--avcodec-hw=none`
- vmem 출력 강제: `--vout=vmem`
- 네트워크 안정성: `--network-caching=1000`

---

### 원본 VlcMedia 대비 수정 사항

원본 플러그인(ue4plugins/VlcMedia)을 그대로 쓰면 UE 5.6과 RTSP 재생에서 문제가 발생해 아래 부분을 커스터마이징했습니다.

#### 1) UE 5.6 호환 빌드 수정
- UE 5.6 API 변경사항 반영 (Build.cs, uplugin 등)
- `EngineVersion`을 `"5.6.0"`으로 업데이트, `Version`을 `"14.0"`으로 수정
- 모듈 의존성 최적화 (`VlcMediaFactory` 포함)

#### 2) LibVLC 초기화 옵션 강화
- HW 디코더 비활성화 (`--avcodec-hw=none`, `--no-hw-decoder`)
- vmem 출력 강제 (소프트웨어 프레임 콜백 경로 확보)
- 동영상 형식 설정 (RV32) 추가
- VLC_PLUGIN_PATH 환경 변수 설정 로직 추가
- 초기화 실패 시 재시도 구현

#### 3) RTSP 연결 안정성 개선
- TCP 전송 옵션 처리 (`?transport=tcp` 또는 미디어 옵션 추가)
- RTSP URL 파싱 및 미디어 옵션 추가 로직 구현
- 미디어 콜백 재초기화 로직 추가

#### 4) 에디터 에셋 우선 사용
- 에디터에서 생성한 MediaPlayer/MediaTexture를 우선 사용하도록 액터 로직 정리
- 런타임 생성 시 발생하는 연결 불일치 문제 회피
- 신규 미디어 에셋 추가 (NewMediaPlayer.uasset, NewMediaTexture_Mat.uasset 등)

#### 5) 최신 개선 사항 (2026년 1월)
- RTSP 테스트 레벨 초기 설정 변경 및 리소스 갱신
- 불필요한 플러그인 파일(libbluray-j2se-1.0.2.jar) 삭제
- 일부 경로 절대 경로 처리 개선

---

### 실전 구현 - CCTVStreamActor 만들기

### 기본 구조

먼저 CCTV를 표시할 액터를 만듭니다. 저희 프로젝트의 실제 코드를 기반으로 설명해드릴게요.

```cpp
// CCTVStreamActor.h
UCLASS()
class P1_API ACCTVStreamActor : public AActor
{
    GENERATED_BODY()

public:
    // 기본 컴포넌트
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "CCTV")
    USceneComponent* RootScene;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "CCTV")
    UStaticMeshComponent* ScreenMesh;

    // 미디어 관련
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "CCTV")
    UMediaPlayer* MediaPlayer;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "CCTV")
    UMediaTexture* MediaTexture;

    // 소리 재생용 컴포넌트
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "CCTV")
    UMediaSoundComponent* MediaSound;

    // 에디터 에셋 (권장)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "CCTV|Assets")
    UMediaPlayer* MediaPlayerAsset;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "CCTV|Assets")
    UMediaTexture* MediaTextureAsset;

    // RTSP 설정
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "CCTV")
    FString RtspUrl = TEXT("rtsp://210.99.70.120:1935/live/cctv001.stream");

    // 디버깅용: 특정 플레이어 강제 사용
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "CCTV|Debug")
    FName OverridePlayerName;

    // RTSP 연결 시 TCP 강제 사용 여부
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "CCTV|Debug")
    bool bUseTCP = false;

    // 영상에 적용할 기본 머티리얼
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "CCTV")
    UMaterialInterface* BaseMaterial;

private:
    // 이벤트 핸들러
    UFUNCTION()
    void HandleMediaOpened(FString Url);

    UFUNCTION()
    void HandleMediaOpenFailed(FString Url);
};
```

### 생성자 설정

```cpp
ACCTVStreamActor::ACCTVStreamActor()
{
    PrimaryActorTick.bCanEverTick = true;  // Tick 활성화
    OverridePlayerName = TEXT("VlcMedia");

    // 컴포넌트 생성
    RootScene = CreateDefaultSubobject<USceneComponent>(TEXT("RootScene"));
    RootComponent = RootScene;

    ScreenMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("ScreenMesh"));
    ScreenMesh->SetupAttachment(RootScene);

    // 미디어 플레이어 설정
    MediaPlayer = CreateDefaultSubobject<UMediaPlayer>(TEXT("MediaPlayer"));
    MediaPlayer->PlayOnOpen = true;  // 미디어 열릴 때 자동 재생
    MediaPlayer->SetLooping(false);
    
    // 미디어 텍스처 설정
    MediaTexture = CreateDefaultSubobject<UMediaTexture>(TEXT("MediaTexture"));
    MediaTexture->SetMediaPlayer(MediaPlayer);
    MediaTexture->AutoClear = false;  // 비디오가 없을 때 검은 화면 유지
    MediaTexture->NewStyleOutput = true;  // 최신 렌더링 파이프라인 사용
    MediaTexture->UpdateResource();

    // 소리 컴포넌트 설정
    MediaSound = CreateDefaultSubobject<UMediaSoundComponent>(TEXT("MediaSound"));
    MediaSound->SetupAttachment(RootScene);
    MediaSound->SetMediaPlayer(MediaPlayer);
}
```

### RTSP 연결 로직

핵심은 TCP 강제 연결과 플레이어 강제 설정입니다. 네트워크 환경에 따라 UDP보다 안정적이거든요.

```cpp
void ACCTVStreamActor::BeginPlay()
{
    Super::BeginPlay();

    // 에디터 에셋 우선 사용
    if (MediaPlayerAsset)
    {
        MediaPlayer = MediaPlayerAsset;
    }
    if (MediaTextureAsset)
    {
        MediaTexture = MediaTexture;
    }
    if (MediaTexture && MediaPlayer)
    {
        MediaTexture->SetMediaPlayer(MediaPlayer);
    }
    if (MediaSound && MediaPlayer)
    {
        MediaSound->SetMediaPlayer(MediaPlayer);
    }

    // 비디오 머티리얼 설정
    if (BaseMaterial && ScreenMesh)
    {
        VideoMaterial = UMaterialInstanceDynamic::Create(BaseMaterial, this);
        if (VideoMaterial && MediaTexture)
        {
            VideoMaterial->SetTextureParameterValue(FName("Texture"), MediaTexture);
            ScreenMesh->SetMaterial(0, VideoMaterial);
        }
    }

    // 이벤트 바인딩
    if (MediaPlayer)
    {
        MediaPlayer->OnMediaOpened.AddDynamic(this, &ACCTVStreamActor::HandleMediaOpened);
        MediaPlayer->OnMediaOpenFailed.AddDynamic(this, &ACCTVStreamActor::HandleMediaOpenFailed);

        // TCP 강제 옵션
        FString FinalUrl = RtspUrl;
        if (bUseTCP)
        {
            if (!FinalUrl.Contains(TEXT("?")))
            {
                FinalUrl += TEXT("?transport=tcp");
            }
            else if (!FinalUrl.Contains(TEXT("transport=tcp")))
            {
                FinalUrl += TEXT("&transport=tcp");
            }
        }

        // 미디어 소스 생성 및 재생
        UStreamMediaSource* StreamSource = NewObject<UStreamMediaSource>(this);
        StreamSource->StreamUrl = FinalUrl;
        
        // 플레이어 강제 설정
        FName PlayerToUse = OverridePlayerName;
        if (!PlayerToUse.IsNone())
        {
            if (FProperty* PlayerNameProp = StreamSource->GetClass()->FindPropertyByName(FName("PlayerName")))
            {
                if (FName* ValPtr = PlayerNameProp->ContainerPtrToValuePtr<FName>(StreamSource))
                {
                    *ValPtr = PlayerToUse;
                }
            }
        }

        MediaPlayer->OpenSource(StreamSource);
    }
}

// 이벤트 핸들러
void ACCTVStreamActor::HandleMediaOpened(FString Url)
{
    UE_LOG(LogTemp, Log, TEXT("CCTV Connected: %s"), *Url);
    if (MediaPlayer)
    {
        MediaPlayer->Play();
    }
}

void ACCTVStreamActor::HandleMediaOpenFailed(FString Url)
{
    UE_LOG(LogTemp, Error, TEXT("CCTV Connection Failed: %s"), *Url);
}
```

### 머티리얼 연동

MediaTexture를 3D 오브젝트에 표시하려면 Dynamic Material Instance가 필요합니다.

```cpp
// 머티리얼 설정
if (BaseMaterial && ScreenMesh)
{
    VideoMaterial = UMaterialInstanceDynamic::Create(BaseMaterial, this);
    if (VideoMaterial && MediaTexture)
    {
        VideoMaterial->SetTextureParameterValue(FName("Texture"), MediaTexture);
        ScreenMesh->SetMaterial(0, VideoMaterial);
    }
}
```

---

### 성능 최적화와 트러블슈팅

### 성능 최적화 팁

#### 다중 스트림 관리
```cpp
#define MAX_CONCURRENT_STREAMS 8

void UCCTVStreamActor::OptimizeResourceUsage()
{
    if (MediaPlayer && !bIsActive)
    {
        MediaPlayer->Pause();  // 사용하지 않을 때는 일시정지
    }
}
```

#### 메모리 최적화
- Media Texture 크기: 1080p 내에서 조정
- 프레임 버퍼: 필요시 30fps로 제한
- 불필요한 스트림: 즉시 정지 및 해제

#### 트러블슈팅 체크리스트

| 증상 | 원인 | 해결책 |
|---|---|---|
| 검은 화면 | vmem 출력 실패 | `--avcodec-hw=none` 추가 |
| 연결 타임아웃 | 방화벽/네트워크 | `?transport=tcp` 추가 |
| 소리만 나옴 | 비디오 출력 설정 | vmem 설정 재확인 |
| 자주 끊김 | 버퍼 크기 부족 | `--network-caching=1000` 증가 |

### 디버깅 명령어

UE5 콘솔에서 사용할 수 있는 유용한 명령어들:

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
// 스트림 상태 확인
UFUNCTION(BlueprintCallable)
bool IsStreamHealthy() const
{
    if (!MediaPlayer) return false;
    
    return MediaPlayer->IsPlaying() && 
           LastFrameTime > (FDateTime::Now() - FTimespan::FromSeconds(3.0));
}
```

---

### 배포 환경 고려사항

### 보안 설정

프로덕션 환경에서는 보안이 중요합니다.

```cpp
// 인증 처리 (보안 주의)
FString AuthenticatedUrl = FString::Printf(
    TEXT("rtsp://%s:%s@%s:%d%s"),
    *Username, *Password, *Host, Port, *Path
);

// 권장: 쿠키/토큰 기반 인증
FinalUrl += TEXT("?x-key=&x-cert=");
```

### 네트워크 설정

- **필수 포트**: 554 (RTSP), 1935 (RTMP)
- **방화벽**: TCP 대역 허용
- **TLS**: 안정성을 위해 `rtsps://` 사용 고려

### 배포 테스트 시나리오

| 시나리오 | 검증 항목 | 통과 기준 |
|---|---|---|
| 단일 스트림 24시간 | 안정성 | 크래시 없음 |
| 다중 스트림 (4개) | 성능 | 메모리 < 2GB |
| 네트워크 장애 | 복구력 | 5초 내 재연결 |

---

### 마무리하며

UE5.6에서 RTSP CCTV 스트리밍은 충분히 구현 가능합니다. VlcMedia 플러그인을 활용하면 안정적인 실시간 영상 처리가 가능합니다.

핵심은 **라이브러리 내장**, **TCP 강제 연결**, **리소스 관리** 세 가지입니다. 이 방법으로 다수의 CCTV를 안정적으로 운영할 수 있습니다.

#### 추가 팁

1. **에디터 에셋 우선**: 런타임 생성보다 에디터에서 만든 에셋 연결이 안정적
2. **메모리 모니터링**: 복수 스트림 시 메모리 사용량 실시간 확인 필요
3. **자동 복구**: 네트워크 장애 대비 재연결 로직 필수

#### 참고 자료

- [VlcMedia GitHub](https://github.com/ue4plugins/VlcMedia)
- [UE5 Media Framework 문서](https://docs.unrealengine.com/5.6/en-US/API/Runtime/Media/)
- [LibVLC 3.0 API 문서](https://www.videolan.org/developers/vlc/libvlc_group.html)

#### 다음 포스팅 예정

- **UE5 Niagara를 활용한 군중 시뮬레이션 완벽 가이드**
- **WebSocket 실시간 통신으로 구축하는 재난 대응 시스템**

---

**태그**: `#UE5` `#UnrealEngine` `#RTSP` `#CCTV` `#VlcMedia` `#게임엔진` `#미디어프레임워크` `#실시간스트리밍`