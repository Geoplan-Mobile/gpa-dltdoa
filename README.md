# gpa-dltdoa

안드로이드용 **DL-TDoA(Downlink Time Difference of Arrival) UWB 측위 라이브러리**.
UWB 레인징 측정값을 입력받아 태그의 `(x, y, z)` 좌표를 산출합니다.

> 이 저장소는 **배포 전용**입니다. 컴파일된 AAR(난독화 release)과 문서만 제공하며, 소스는 비공개입니다.

## 설치

저장소(raw URL)와 의존성을 추가합니다. **인증 불필요(익명 pull)**.

### `settings.gradle` (또는 모듈 `build.gradle` 의 repositories)

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()   // transitive 의존성(commons-math3, jts-core, slf4j-api) 자동 해결
        maven { url "https://raw.githubusercontent.com/Geoplan-Mobile/gpa-dltdoa/main/repository" }
    }
}
```

### 모듈 `build.gradle`

```gradle
dependencies {
    implementation 'kr.geoplan.android.lib:gpa-dltdoa:1.1.1'
}
```

## 사용법

```java
// 1) 결과 콜백 구현
PositionCallback callback = new PositionCallback() {
    @Override public void onPosition(double x, double y, double z) {
        // 측위 결과 좌표
    }
    @Override public void onInvalidated(String error) {
        // 측위 무효 / 오류
    }
};

// 2) Positioner 생성 + 앵커 정보 적용
DlTdoaPositioner positioner = new DlTdoaPositioner(callback);
positioner.applyAnchorInfo(AnchorInfo.fromJson(anchorJson));  // 앵커 좌표/세션 정보
positioner.setMinRssi(-85);                                   // (선택) 최소 RSSI 임계값

// 3) UWB 레인징 측정값 주입 → 좌표 산출 시 callback.onPosition 호출
positioner.update(peer, measurement);

// 4) 상태 초기화
positioner.reset();
```

## 공개 API

| 타입 | 설명 |
|---|---|
| `DlTdoaPositioner` | 측위 엔진 진입점 — 생성 → `applyAnchorInfo` → `update` |
| `PositionCallback` | 결과 콜백 — `onPosition(x, y, z)` / `onInvalidated(error)` |
| `AnchorInfo` | 앵커 좌표·세션 정보 — `AnchorInfo.fromJson(json)` 으로 생성 |

> 그 외 내부 구현 타입은 비공개(난독화)이며 외부 API 표면에 노출되지 않습니다.

## 버전

- 최신: **1.1.1**
- 좌표: `kr.geoplan.android.lib:gpa-dltdoa:1.1.1`
- 전체 버전 목록: [`repository/.../maven-metadata.xml`](repository/kr/geoplan/android/lib/gpa-dltdoa/maven-metadata.xml)
