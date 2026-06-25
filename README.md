# gpa-dltdoa (Android Library)

본 저장소는 `gpa-dltdoa` 측위 코어 알고리즘의 외부 연동 장착을 위한 **배포 전용 릴리즈 저장소(Release Repository)**이다.
사전 컴파일(Pre-compiled) 및 난독화(R8)가 완료된 정적 `AAR` 바이너리를, 정적 Maven 저장소(raw URL) 형태로 **인증 없이(익명)** 독립 제공한다.
좌표 `kr.geoplan.android.lib:gpa-dltdoa` 로 의존성에 추가하며, 루트 `repository/.../maven-metadata.xml` 로 배포 버전을 확인할 수 있다.

> **💡 엔진 코어 역량 요약**
> Android의 **Ranging (UWB, `android.ranging`)** 인프라 기반 동작.
> UWB 앵커 상호 간의 DL-TDoA(Downlink TDoA) 측정 데이터를 분석하여 기기 단말(Device)의 실시간 (X, Y, Z) 실내 좌표를 역산.

> **⚠️ 동작 요구사항**
> 측위 호출(`DlTdoaPositioner`)은 Android의 **`android.ranging` Ranging API** 를 사용하며 매우 최신 API 레벨을 요구한다(`@RequiresApi`).
> 라이브러리 의존성 추가 자체는 `minSdk 27` 부터 가능하나, 측위 객체 호출은 런타임 `Build.VERSION.SDK_INT` 가드 하에서만 수행해야 한다.

---

## 프로젝트 연동 및 사용 방법 (Usage)

### 1. Gradle 저장소 및 의존성 연동

저장소(raw URL)와 의존성을 추가한다. **인증 토큰이 필요 없다.**

```gradle
// settings.gradle  (dependencyResolutionManagement) 또는 모듈 build.gradle 의 repositories
repositories {
    google()
    mavenCentral()   // transitive 의존성(commons-math3, jts-core, slf4j-api) 자동 해결
    maven { url "https://raw.githubusercontent.com/Geoplan-Mobile/gpa-dltdoa/main/repository" }
}
```

```gradle
// 모듈 build.gradle
dependencies {
    implementation 'kr.geoplan.android.lib:gpa-dltdoa:1.1.1'
}
```

### 2. DlTdoaPositioner 생성 및 측정값 주입

`DlTdoaPositioner` 인스턴스를 생성하고, UWB 세션(`android.ranging`)에서 DL-TDoA 측정 콜백이 도착할 때마다 **측정 한 건씩** `update()` 로 주입한다. 세션의 생성/수명주기는 호출자(앱)의 책임이며, 라이브러리는 측정값을 받아 좌표만 산출한다.

```java
import android.ranging.DlTdoaMeasurement;
import android.ranging.RangingDevice;
import kr.geoplan.android.lib.dltdoa.*;

// 1) 결과 콜백 구현
PositionCallback callback = new PositionCallback() {
    @Override public void onPosition(double x, double y, double z) {
        // 측위 결과 좌표 (단위: m)
    }
    @Override public void onInvalidated(String error) {
        // 측위 무효 / 오류
    }
};

// 2) 엔진 생성 + 앵커 좌표 주입 (update 전에 1회)
DlTdoaPositioner positioner = new DlTdoaPositioner(callback);
positioner.applyAnchorInfo(AnchorInfo.fromJson(anchorInfoJson));
positioner.setMinRssi(-90);   // (선택) 최소 RSSI 임계값. 기본 -90 dBm

// 3) UWB 측정 단건마다 주입 → 라운드 완성 시 callback.onPosition 호출
//    (android.ranging 콜백 내부에서 호출)
positioner.update(peer, measurement);   // peer: RangingDevice, measurement: DlTdoaMeasurement

// 4) 세션 중지/재시작 시 누적 라운드 상태 초기화
positioner.reset();
```

> **측위 흐름**
> `update()` 는 측정값을 **한 건씩** 받아 내부에서 라운드(블록) 단위로 누적한다. 한 라운드가 완성되면 좌표를 산출해 `onPosition(x, y, z)` 로 통지한다. 패킷 부족·신호 손실 등으로 좌표를 낼 수 없으면 `onInvalidated(error)` 가 호출된다.

### 3. 앵커 위치(좌표) 주입 — `AnchorInfo`

정밀 측위를 위해 각 UWB 앵커의 실제 공간 좌표를 엔진에 고지해야 한다. Edge 의 `anchor_info` JSON 을 `AnchorInfo.fromJson()` 으로 파싱해 주입한다.

```jsonc
// anchor_info JSON 스키마
{
  "sessionId": 4444,
  "anchors": [
    { "mac": "5555", "location": [1.0, 2.5, 2.0], "offsetNs": 0.0 },   // mac 은 16진수 문자열
    { "mac": "B4BF", "location": [5.0, 5.5, 2.0], "offsetNs": 0.0 }
  ]
}
```

```java
AnchorInfo info = AnchorInfo.fromJson(anchorInfoJson);  // 파싱 실패 시 null 반환
if (info != null) {
    positioner.applyAnchorInfo(info);
}
```

---

## API 레퍼런스 (API Reference)

대외적으로 개방(public)된 핵심 타입의 기술 명세이다. 그 외 내부 구현 타입은 비공개(난독화)이며 API 표면에 노출되지 않는다.

### 클래스: `DlTdoaPositioner`
측위 연산을 주관하는 메인 진입점(facade).

* **`DlTdoaPositioner(PositionCallback callback)`**
  엔진 인스턴스를 생성한다. `callback` 으로 좌표/에러가 통지된다.
* **`void applyAnchorInfo(AnchorInfo info)`**
  앵커 좌표를 주입한다. `update()` 호출 전에 수행한다.
* **`void setMinRssi(int minRssi)`**
  연산에 반영할 최소 수신 신호 강도(dBm) 임계값. **기본값 `-90`**. 이보다 약한 앵커 측정값은 제외된다.
* **`void update(RangingDevice peer, DlTdoaMeasurement m)`**
  UWB 측정 **한 건**을 입력한다. 내부에서 라운드를 누적하고, 라운드가 완성되면 좌표를 산출해 `PositionCallback.onPosition` 을 호출한다. (측정 콜백마다 호출)
* **`void reset()`**
  누적 중인 라운드 상태를 비운다. 세션 중지/재시작 시 호출.

### 인터페이스: `PositionCallback`
측위 결과 콜백.

* **`void onPosition(double x, double y, double z)`** — 산출된 (x, y, z) 좌표 (단위: m).
* **`void onInvalidated(String error)`** — 측위 무효/오류 통지.

### 클래스: `AnchorInfo`
DL-TDoA 앵커 셋업 정보.

* **`static final int DEFAULT_SESSION_ID = 4444`** — 세션 ID 기본값.
* **`final int sessionId`** — 세션 ID.
* **`final Map<Integer, double[]> coordinates`** — 앵커 MAC(16-bit) → 좌표 `[x, y, z]` (m).
* **`final Map<Integer, Double> offsetNs`** — 앵커 MAC → 캘리브레이션 오프셋(ns). *현재 측위 로직 미사용(형식 호환용 보관).*
* **`AnchorInfo(int sessionId, Map<Integer, double[]> coordinates, Map<Integer, Double> offsetNs)`** — 직접 생성자.
* **`static AnchorInfo fromJson(String json)`** — Edge `anchor_info` JSON 파싱. 실패 시 `null` 반환.
