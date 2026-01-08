# SistasRTC SDK

WebRTC tabanlı gerçek zamanlı video ve ses iletişimi için cross-platform SDK. SIP protokolü üzerinden WebSocket kullanarak güvenli ve stabil bağlantılar sağlar.

## Desteklenen Platformlar

| Platform | Minimum Versiyon |
|----------|------------------|
| **Android** | API 24 (Android 7.0) |
| **iOS** | iOS 13+ |

## Özellikler

- WebRTC tabanlı video/audio çağrı
- SIP protokolü desteği
- WebSocket üzerinden sinyal iletimi
- Otomatik yeniden bağlanma mekanizması
- Bluetooth kulaklık desteği
- Kamera geçişi (ön/arka)
- Mikrofon kontrolü
- Dinamik video çözünürlük ayarları
- Network değişikliği algılama
- ICE restart desteği

---

# Android Entegrasyonu

## Gereksinimler

- **Minimum SDK:** 24 (Android 7.0)
- **Target SDK:** 35
- **Compile SDK:** 35
- **Java:** 11+
- **Desteklenen ABI:** armeabi-v7a, arm64-v8a

## Kurulum

### 1. Gradle Bağımlılıkları

`settings.gradle` dosyanıza ekleyin:

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        flatDir {
            dirs 'sistaswebrtcsdk/libs'
        }
    }
}
```

`build.gradle` (app modülü):

```gradle
dependencies {
    implementation project(':sistaswebrtcsdk')
}
```

### 2. Manifest İzinleri

`AndroidManifest.xml` dosyanıza gerekli izinleri ekleyin:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

## Kullanım

### 1. CallManager Oluşturma

```java
import com.sistas.sistaswebrtcsdk.CallManager;
import com.sistas.sistaswebrtcsdk.CallEventListener;

CallManager callManager = new CallManager(context);
```

### 2. Callback Dinleyici Ekleme

```java
callManager.setCallEventListener(new CallEventListener() {
    @Override
    public void onCallStateChanged(String callState, String reason) {
        switch (callState) {
            case "CONNECTED":
                // Çağrı bağlandı
                break;
            case "ENDED":
                // Çağrı sonlandı (reason: 200, NO_CONNECTION, 4xx, 5xx vs.)
                break;
            case "INFO":
                // Bilgi mesajı (SWITCHCAMERA, PHOTO vs.)
                break;
            case "NETWORK_CHANGE":
                // Network değişikliği algılandı
                break;
        }
    }
});
```

### 3. Çağrı Başlatma

```java
SurfaceViewRenderer localView = findViewById(R.id.local_view);
SurfaceViewRenderer remoteView = findViewById(R.id.remote_view);

Map<String, String> customHeaders = new HashMap<>();
customHeaders.put("X-Custom-Header", "value");

callManager.startCall(
    localView,           // Yerel kamera görünümü
    remoteView,          // Uzak kamera görünümü
    "sip.example.com",   // SIP sunucu adresi
    8089,                // SIP sunucu portu
    "1234",              // Aranacak numara
    "5678",              // Arayan kullanıcı
    customHeaders,       // Özel SIP header'ları (nullable)
    callId -> {
        // Call ID hazır olduğunda tetiklenir
        Log.d("CallID", callId);
    }
);
```

### 4. Çağrı Kontrolleri

```java
// Kamera değiştir
callManager.switchCamera();

// Mikrofon aç/kapat
boolean isMuted = callManager.toggleMicrophone();

// Yerel kamerayı kapat/aç
callManager.disableLocalCamera();
callManager.enableLocalCamera();

// Çağrıyı sonlandır
callManager.endCall(true); // true: BYE mesajı gönder
```

### 5. Opsiyonel Ayarlar

```java
// Video çözünürlüğü (varsayılan: 640x480@30fps)
callManager.setVideoConstraints(1280, 720, 30);

// Yeniden bağlanma ayarları (varsayılan: 10 deneme, 3000ms aralık)
callManager.setReconnectOptions(5, 2000);
```

## Android Önemli Uygulama Notları

### Ekran Döndürme (Configuration Changes)

Çağrı ekranında ekran döndürüldüğünde Activity'nin yeniden oluşturulmasını engellemek için **AndroidManifest.xml** dosyasında `configChanges` ayarını eklemeniz gerekir:

```xml
<activity
    android:name=".CallActivity"
    android:configChanges="orientation|screenSize|screenLayout|smallestScreenSize|keyboardHidden"
    android:screenOrientation="unspecified" />
```

### Geri Tuşu ve Geri Kaydırma (Back Navigation)

Kullanıcı çağrı ekranındayken geri tuşuna bastığında veya geri kaydırma yaptığında çağrının düzgün şekilde sonlandırılması için `onBackPressed()` metodunu override etmeniz gerekir:

```java
@Override
public void onBackPressed() {
    if (callManager != null) {
        callManager.endCall(true);
        callManager = null;
    }
    super.onBackPressed();
}
```

### Activity Lifecycle Yönetimi

Activity yok edildiğinde çağrının temizlenmesi için `onDestroy()` metodunu implement edin:

```java
@Override
protected void onDestroy() {
    if (callManager != null) {
        callManager.endCall(true);
        callManager = null;
    }
    super.onDestroy();
}
```

### Aktif Çağrı Kontrolü

Aynı anda birden fazla çağrı başlatılmasını engellemek için çağrı durumunu takip eden bir flag kullanın:

```java
public class CallActivity extends AppCompatActivity {
    public static boolean isCallActive = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        isCallActive = true;
        // ...
    }

    @Override
    protected void onDestroy() {
        isCallActive = false;
        // ...
    }
}

// MainActivity'de çağrı başlatmadan önce kontrol edin:
private void openCallActivity() {
    if (CallActivity.isCallActive) {
        Toast.makeText(this, "Zaten aktif bir çağrı var!", Toast.LENGTH_SHORT).show();
        return;
    }
    startActivity(new Intent(this, CallActivity.class));
}
```

## Android Layout Örneği

```xml
<org.webrtc.SurfaceViewRenderer
    android:id="@+id/remote_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />

<org.webrtc.SurfaceViewRenderer
    android:id="@+id/local_view"
    android:layout_width="120dp"
    android:layout_height="160dp"
    android:layout_gravity="top|end"
    android:layout_margin="16dp" />
```

---

# iOS Entegrasyonu

## Gereksinimler

- **iOS 13** veya üstü
- **WebRTC** kütüphanesi

## Info.plist Konfigürasyonu

```xml
<key>NSCameraUsageDescription</key>
<string>Görüntülü görüşme için kameranıza erişmemiz gerekiyor.</string>

<key>NSMicrophoneUsageDescription</key>
<string>Görüntülü görüşme için mikrofonunuza erişmemiz gerekiyor.</string>
```

## Kullanım

### 1. Video View'ların Hazırlanması

```swift
private let remoteView = RTCMTLVideoView(frame: .zero)
private let localView  = RTCMTLVideoView(frame: .zero)

private func setupVideoViews() {
    remoteView.translatesAutoresizingMaskIntoConstraints = false
    localView.translatesAutoresizingMaskIntoConstraints = false

    // Remote video (arka plan, tam ekran)
    remoteView.videoContentMode = .scaleAspectFit
    remoteView.contentMode = .scaleAspectFill

    // Local video (küçük pencere, köşede)
    localView.videoContentMode  = .scaleAspectFill
    localView.layer.cornerRadius = 10
    localView.clipsToBounds = true

    view.addSubview(remoteView)
    view.addSubview(localView)
}
```

### 2. CallManager ve Delegate Ayarlama

```swift
final class CallViewController: UIViewController, SistasCallStateListener {
    private let callManager = CallManager()

    override func viewDidLoad() {
        super.viewDidLoad()
        callManager.stateDelegate = self
    }

    func callManager(_ manager: CallManager, didChangeState state: CallState, reason: CallReason) {
        switch state {
        case .ended:
            closeCallScreen()

        case .connected:
            print("Çağrı Bağlandı")

        case .callId:
            if case .text(let callId) = reason {
                print("Call ID: \(callId)")
            }

        default:
            break
        }
    }
}
```

### 3. Çağrı Başlatma

```swift
let extra: [String: String] = [
    "X-Device-Model": model,
    "X-OS-Version": ios,
    "X-Client-Version": "iOS-Demo-1.0"
]

callManager.startCall(
    localview: localView,
    remoteview: remoteView,
    domain: "sip.example.com",
    port: 8089,
    callTo: "1234",
    fromUser: "5678",
    extraheaders: extra
)
```

### 4. Çağrı Kontrolleri

```swift
// Mikrofonu aç/kapat
let isMicMuted = callManager.toggleMic()

// Kamera değiştir
callManager.switchCamera()

// Çağrıyı sonlandır
callManager.endCall()
```

### 5. Opsiyonel Ayarlar

```swift
// Video çözünürlüğü (varsayılan: 640x480@30fps)
callManager.setVideoConstraints(width: 1280, height: 720, fps: 30)

// Yeniden bağlanma ayarları (varsayılan: 10 deneme, 3000ms aralık)
callManager.setReconnectOptions(maxReconnectAttempts: 5, reconnectDelayMs: 2000)
```

## iOS CallState Değerleri

| State | Açıklama |
|-------|----------|
| `.callId` | Çağrı başarıyla başlatıldı, Call ID döndürüldü |
| `.connected` | Çağrı bağlantısı kuruldu |
| `.disconnected` | Bağlantı kesildi |
| `.info` | Bilgilendirme mesajı |
| `.networkChange` | Ağ değişikliği algılandı |
| `.ended` | Çağrı sonlandırıldı |
| `.notAcceptableHere` | SDP uyuşmazlığı (488) |
| `.serverInternalError` | Sunucu hatası (500) |

---

# Ortak API Referansı

## CallManager Metodları

| Metod | Android | iOS | Açıklama |
|-------|---------|-----|----------|
| Çağrı başlat | `startCall(...)` | `startCall(...)` | Yeni çağrı başlatır |
| Çağrı sonlandır | `endCall(boolean)` | `endCall()` | Çağrıyı sonlandırır |
| Kamera değiştir | `switchCamera()` | `switchCamera()` | Ön/arka kamera geçişi |
| Mikrofon toggle | `toggleMicrophone()` | `toggleMic()` | Mikrofonu açar/kapatır |
| Video ayarları | `setVideoConstraints(w,h,fps)` | `setVideoConstraints(w,h,fps)` | Video parametreleri |
| Reconnect ayarları | `setReconnectOptions(max,delay)` | `setReconnectOptions(max,delay)` | Yeniden bağlanma |

## SIP Hata Kodları

| Kod | Anlamı |
|-----|--------|
| 200 | OK - Başarılı |
| 400 | Bad Request - Hatalı istek |
| 403 | Forbidden - Yasaklandı |
| 404 | Not Found - Bulunamadı |
| 408 | Request Timeout - Zaman aşımı |
| 480 | Temporarily Unavailable - Geçici olarak kullanılamıyor |
| 486 | Busy Here - Meşgul |
| 487 | Request Terminated - Sonlandırıldı |
| 488 | Not Acceptable Here - SDP sorunu |
| 500 | Server Internal Error - Sunucu hatası |
| 503 | Service Unavailable - Servis kullanılamıyor |
| 603 | Decline - Reddedildi |

---

# Sorun Giderme

## Ortak Sorunlar

| Sorun | Çözüm |
|-------|-------|
| Kamera açılmıyor | Kamera izinlerini kontrol edin |
| Ses duyulmuyor | Mikrofon izinlerini ve Bluetooth bağlantısını kontrol edin |
| Bağlantı kurulamıyor | SIP sunucu adresi/port ve internet bağlantısını kontrol edin |
| Sık kopuyor | `setReconnectOptions` ile deneme sayısını artırın |

## Android Spesifik

- Ekran döndüğünde çağrı yeniden başlıyorsa `configChanges` manifest ayarını ekleyin
- Geri gidince çağrı kapanmıyorsa `onBackPressed()` ve `onDestroy()` override edin

## iOS Spesifik

- Info.plist izinlerini kontrol edin
- iOS 13+ kullandığınızdan emin olun

---

# Teknik Detaylar

## Kullanılan Teknolojiler

| Bileşen | Android | iOS |
|---------|---------|-----|
| WebRTC | libwebrtc_m136.7103.0.0 | WebRTC.framework |
| WebSocket | OkHttp3 | Native URLSession |
| Protokol | SIP over WebSocket (RFC 7118) | SIP over WebSocket (RFC 7118) |
| SDP | Unified Plan | Unified Plan |

## Varsayılan Değerler

| Ayar | Değer |
|------|-------|
| Video Çözünürlük | 640x480 |
| FPS | 30 |
| Max Reconnect | 10 |
| Reconnect Delay | 3000ms |
| Keepalive | 30 saniye (OPTIONS) |

---

# Destek

Sorularınız ve sorunlarınız için:

- Email: genesysinfo@sistas.com.tr
- Web: https://www.sistas.com.tr

---

**SDK Versiyonu:** Android 1.0.2 / iOS 1.0.2
**Son Güncelleme:** 2025
