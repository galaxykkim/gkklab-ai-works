---
skill: flutter-test-init-webview
description: 테스트용 WebView 화면을 구현합니다. 화면 전체 영역에 TestWebView를 배치하고, 상단에 URL 입력 필드와 Load 버튼을 표시하여 런타임에 원하는 URL을 로드할 수 있게 합니다. 만약 URL을 입력하지 않은 경우에는 기본으로 설정된 html source를 로드합니다.
---

> **전제 패키지** : `flutter_riverpod`, `go_router`, `webview_flutter`  
> **전제 클래스** : `JsWebView`, `JsWebViewController`, `JsInterface`, `JsMessage` (`lib/core/jswebview/` 하위)

---

## Instructions

Flutter 프로젝트에 테스트용 WebView 화면을 구성하는 파일들을 아래 구조와 규칙에 따라 생성합니다.
만약 생성할 경로에 동일한 이름의 폴더나 파일이 존재할 경우, 사용자에게 직접 경로를 입력받도록 합니다.

### 생성 파일 목록

```
lib/
└── test/
    └── webview/
        ├── test_interface.dart
        ├── test_webview.dart
        ├── test_webview_intent.dart
        ├── test_webview_state.dart
        ├── test_webview_viewmodel.dart
        └── test_webview_screen.dart
```

---

### 1. `test_interface.dart`

JS 브릿지 인터페이스(`CommonInterface`)를 정의합니다.
`JsInterface`를 상속하며 `showToast` 액션을 처리합니다.

```dart
// lib/test/webview/test_interface.dart

import '../../core/jswebview/js_config.dart';
import '../../core/jswebview/js_interface.dart';
import '../../core/jswebview/js_message.dart';

/// 공통 UI 동작 JS Interface.
class CommonInterface extends JsInterface {
  /// Toast 메시지 표시 콜백.
  final void Function(String message) showToast;

  CommonInterface({required this.showToast});

  @override
  String get name => 'common';

  @override
  Future<void> onReceived(JsMessage message) async {
    switch (message.action) {
      case CommonAction.showToast:
        _showToast(message);
    }
  }

  void _showToast(JsMessage request) async {
    final toastMessage = request.data[CommonDataKey.message] as String? ?? '';
    showToast(toastMessage);

    // 비동기 로직 테스트.
    await Future.delayed(const Duration(seconds: 3));

    // callbackId가 있으면 Native에서 응답 전송
    successCallback(request, {'result': 'success'});
  }
}

class CommonAction {
  static const String showToast = 'showToast';
}

class CommonDataKey {
  static const String message = 'message';
}
```

---

### 2. `test_webview.dart`

`JsWebView`를 래핑한 테스트용 WebView 위젯을 정의합니다.
- `CommonInterface`를 기본으로 등록합니다.
- `onControllerReady` 콜백을 외부로 노출하여 `test_webview_screen.dart`에서 컨트롤러를 획득할 수 있도록 합니다.
- URL을 입력하지 않았을 때를 대비해 JS 브릿지 테스트용 기본 HTML source를 초기값으로 사용합니다.

```dart
// lib/test/webview/test_webview.dart

import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';
import '../../core/jswebview/js_interface.dart';
import '../../core/jswebview/js_webview.dart';
import '../../core/jswebview/js_webview_controller.dart';
import 'test_interface.dart';

class TestWebView extends StatefulWidget {
  final void Function(JsWebViewController)? onControllerReady;

  const TestWebView({super.key, this.onControllerReady});

  @override
  State<TestWebView> createState() => _TestWebViewState();
}

class _TestWebViewState extends State<TestWebView> {
  late final List<JsInterface> _interfaces;
  late final String _htmlSource;

  @override
  void initState() {
    super.initState();
    _interfaces = [
      CommonInterface(
        showToast: (msg) {
          debugPrint('>>> TestWebView > showToast: $msg');
        },
      ),
    ];
    _htmlSource = '''
      <!DOCTYPE html>
      <html>
        <head>
          <meta name="viewport" content="width=device-width, initial-scale=1.0">
          <style>
            body { font-family: sans-serif; text-align: center; padding: 20px; background-color: #f5f5f5; }
            button { 
              padding: 12px 24px; font-size: 16px; cursor: pointer; 
              background-color: #007AFF; color: white; border: none; border-radius: 8px;
              box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            }
            #display { 
              margin-top: 20px; padding: 15px; border-radius: 8px; 
              background-color: white; min-height: 50px; border: 1px solid #ddd;
            }
          </style>
        </head>
        <body>
          <h3>WebView <-> Native Bridge</h3>
          <div id="display">Native 메시지를 기다리는 중...</div>
          <br/>
          <button onclick="callToastWithCallback()">Toast 호출 (Callback 확인)</button>
  
          <script>
            // 1. 콜백 함수들을 저장할 객체
            window.callbacks = {};

            // 2. Native → WebView: 단방향 데이터 수신
            window.nativeSend = function(message) {
              const displayElement = document.getElementById('display');
              displayElement.innerText = "[Native 전송] " + JSON.stringify(message.data);
              displayElement.style.color = "#34C759";
            };

            // 3. Native → WebView: 비동기 요청 수신 (응답 필요)
            window.nativeSendAsync = function(message) {
              const displayElement = document.getElementById('display');
              displayElement.innerText = "[Native 요청] action=" + message.action + " (3초 후 응답)";
              displayElement.style.color = "#FF9500";

              // 3초 지연 후 응답 전송
              setTimeout(function() {
                const response = {
                  type: "response",
                  callbackId: message.callbackId,
                  action: message.action,
                  data: { result: "WebView 처리 완료" }
                };
                if (window.common && window.common.postMessage) {
                  window.common.postMessage(JSON.stringify(response));
                }
                displayElement.innerText = "[Native 요청] 응답 완료: " + message.action;
                displayElement.style.color = "#34C759";
              }, 3000);
            };

            // 4. Native → WebView: WebView 요청에 대한 Native 응답 수신
            window.resolveCallback = function(message) {
              const callbackId = message.callbackId;
              console.log("Received response from Native:", message);
              if (window.callbacks[callbackId]) {
                window.callbacks[callbackId](message);
                delete window.callbacks[callbackId]; // 메모리 해제
              }
            };

            // 5. 브릿지 호출 함수
            function callToastWithCallback() {
              const displayElement = document.getElementById('display');
              
              // 고유 콜백 ID 생성
              const callbackId = 'cb_' + Date.now();
              
              // 응답이 왔을 때 실행할 로직 등록
              window.callbacks[callbackId] = function(message) {
                if (message.success !== false) {
                  displayElement.innerText = "Native 응답 성공: " + JSON.stringify(message.data);
                  displayElement.style.color = "#007AFF";
                } else {
                  displayElement.innerText = "Native 응답 에러: " + message.error;
                  displayElement.style.color = "#FF3B30";
                }
              };

              // JsMessage 규격에 맞는 메시지 구성
              const message = {
                type: "request",
                callbackId: callbackId,
                action: "showToast",
                data: {
                  message: "웹에서 보낸 콜백 테스트 메시지입니다."
                }
              };

              // 'common' 채널을 통해 메시지 전송
              if (window.common && window.common.postMessage) {
                window.common.postMessage(JSON.stringify(message));
                displayElement.innerText = "Native 요청 보냄 (ID: " + callbackId + ")";
                displayElement.style.color = "#8E8E93";
              } else {
                displayElement.innerText = "에러: 'common' 인터페이스를 찾을 수 없습니다.";
                displayElement.style.color = "#FF3B30";
              }
            }
          </script>
        </body>
      </html>
    ''';
  }

  @override
  Widget build(BuildContext context) {
    return JsWebView(
      htmlSource: _htmlSource,
      interfaces: _interfaces,
      onControllerReady: (controller) {
        widget.onControllerReady?.call(controller);
      },
      navigationDelegate: NavigationDelegate(
        onProgress: (int progress) {},
        onPageStarted: (String url) {},
        onPageFinished: (String url) {},
        onWebResourceError: (WebResourceError error) {},
        onNavigationRequest: (NavigationRequest request) {
          return NavigationDecision.navigate;
        },
      ),
    );
  }
}
```

---

### 3. `test_webview_intent.dart`

MVI 패턴의 Intent를 sealed class로 정의합니다.

```dart
// lib/test/webview/test_webview_intent.dart

sealed class TestWebviewIntent {
  const TestWebviewIntent();
}

/// Load 버튼 클릭 시 발생 — 입력된 URL을 WebView에 로드합니다
class OnLoadPressed extends TestWebviewIntent {
  final String url;
  const OnLoadPressed(this.url);
}
```

---

### 4. `test_webview_state.dart`

MVI 패턴의 State를 불변 클래스로 정의합니다. `copyWith`를 지원합니다.

```dart
// lib/test/webview/test_webview_state.dart

class TestWebviewState {
  final bool isLoading;

  const TestWebviewState({
    this.isLoading = false,
  });

  TestWebviewState copyWith({
    bool? isLoading,
  }) {
    return TestWebviewState(
      isLoading: isLoading ?? this.isLoading,
    );
  }
}
```

---

### 5. `test_webview_viewmodel.dart`

`Notifier<TestWebviewState>` 기반의 ViewModel을 작성합니다.
- `processIntent(TestWebviewIntent)` 로 모든 Intent를 처리합니다.
- WebView URL 로딩은 `JsWebViewController`에 직접 접근해야 하므로, Screen에서 `onLoadUrl` 콜백을 주입합니다. ViewModel이 WebView 컨트롤러에 직접 의존하지 않도록 분리합니다.

```dart
// lib/test/webview/test_webview_viewmodel.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'test_webview_intent.dart';
import 'test_webview_state.dart';

class TestWebviewViewModel extends Notifier<TestWebviewState> {
  /// OnLoadPressed 처리 시 호출할 URL 로딩 콜백 (Screen에서 주입)
  void Function(String url)? onLoadUrl;

  @override
  TestWebviewState build() => const TestWebviewState();

  void processIntent(TestWebviewIntent intent) {
    switch (intent) {
      case OnLoadPressed():
        _onLoadPressed(intent.url);
    }
  }

  void _onLoadPressed(String url) {
    onLoadUrl?.call(url);
  }
}

/// Provider 선언
final testWebviewViewModelProvider =
    NotifierProvider<TestWebviewViewModel, TestWebviewState>(
        TestWebviewViewModel.new);
```

---

### 6. `test_webview_screen.dart`

`ConsumerStatefulWidget` 기반 화면을 작성합니다.

**레이아웃 구조:**
- `Scaffold` body를 `Column`으로 구성합니다 (Stack/Overlay 방식 사용 안 함).
- **URL 입력 영역**: 상단에 실제 공간을 차지하는 형태로 배치합니다. `SafeArea(bottom: false)` → `Container(black87)` → `Row(TextField + ElevatedButton)`.
- **WebView 영역**: `Expanded`로 URL 입력 영역 아래에 배치합니다.
- **Native 버튼 영역**: URL 미입력 시에만(`!_hasLoadedUrl`) 하단에 표시합니다. `SafeArea(top: false)` → `Row(Send + SendAsync)`. URL 입력 후에는 사라집니다.

**상태 분기:**
- `_hasLoadedUrl = false` (초기): WebView가 `_htmlSource`를 로드하고, 하단에 Send/SendAsync 버튼 표시.
- `_hasLoadedUrl = true` (URL 입력 후): WebView가 입력된 URL을 로드하고, 하단 버튼 미표시.

**컨트롤러 취득:**
- `onControllerReady` 콜백에서 `setState`로 `_webViewController`를 저장하여 Native 버튼 활성화를 트리거합니다.

```dart
// lib/test/webview/test_webview_screen.dart

import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../core/jswebview/js_config.dart';
import '../../core/jswebview/js_message.dart';
import '../../core/jswebview/js_webview_controller.dart';
import 'test_webview.dart';
import 'test_webview_intent.dart';
import 'test_webview_viewmodel.dart';

class TestWebviewScreen extends ConsumerStatefulWidget {
  const TestWebviewScreen({super.key});

  @override
  ConsumerState<TestWebviewScreen> createState() => _TestWebviewScreenState();
}

class _TestWebviewScreenState extends ConsumerState<TestWebviewScreen> {
  final TextEditingController _urlController = TextEditingController();
  JsWebViewController? _webViewController;
  bool _hasLoadedUrl = false;

  @override
  void dispose() {
    _urlController.dispose();
    super.dispose();
  }

  void _load(String url) {
    if (url.trim().isEmpty) return;
    ref
        .read(testWebviewViewModelProvider.notifier)
        .processIntent(OnLoadPressed(url.trim()));
    setState(() => _hasLoadedUrl = true);
  }

  void _onSend() {
    _webViewController?.send(JsMessage(
      type: JsMessageType.request,
      action: 'nativeData',
      data: {'message': 'Native에서 단방향으로 보낸 데이터입니다.'},
    ));
  }

  void _onSendAsync() async {
    final response = await _webViewController?.sendAsync(JsMessage(
      type: JsMessageType.request,
      action: 'nativeAsyncRequest',
      callbackId: 'cb_1234567890',
      data: {'message': 'Native에서 비동기 요청을 보냈습니다.'},
    ));
    debugPrint('>>> sendAsync response: ${response?.toJson()}');
  }

  @override
  Widget build(BuildContext context) {
    final vm = ref.read(testWebviewViewModelProvider.notifier);

    // JsWebViewController 콜백 주입 — ViewModel이 WebView에 직접 의존하지 않도록 분리
    vm.onLoadUrl = (url) => _webViewController?.loadUrl(url);

    return Scaffold(
      resizeToAvoidBottomInset: false,
      body: Column(
        children: [
          // ── URL 입력 영역 (상단 고정, 실제 공간 차지) ────────────
          SafeArea(
            bottom: false,
            child: Container(
              color: Colors.black87,
              padding: const EdgeInsets.symmetric(
                horizontal: 12,
                vertical: 8,
              ),
              child: Row(
                children: [
                  Expanded(
                    child: TextField(
                      controller: _urlController,
                      style: const TextStyle(color: Colors.white),
                      decoration: const InputDecoration(
                        hintText: 'https://',
                        hintStyle: TextStyle(color: Colors.white54),
                        isDense: true,
                        contentPadding: EdgeInsets.symmetric(
                          horizontal: 12,
                          vertical: 10,
                        ),
                        filled: true,
                        fillColor: Colors.white12,
                        border: OutlineInputBorder(
                          borderSide: BorderSide.none,
                          borderRadius: BorderRadius.all(Radius.circular(8)),
                        ),
                      ),
                      keyboardType: TextInputType.url,
                      textInputAction: TextInputAction.go,
                      autocorrect: false,
                      onSubmitted: _load,
                    ),
                  ),
                  const SizedBox(width: 8),
                  ElevatedButton(
                    onPressed: () => _load(_urlController.text),
                    child: const Text('Load'),
                  ),
                ],
              ),
            ),
          ),

          // ── WebView 영역 ────────────────────────────────────────
          Expanded(
            child: TestWebView(
              onControllerReady: (controller) {
                setState(() => _webViewController = controller);
              },
            ),
          ),

          // ── Native 버튼 영역 (URL 미입력 시) ─────────────────────
          if (!_hasLoadedUrl)
            SafeArea(
              top: false,
              child: Padding(
                padding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 12,
                ),
                child: Row(
                  children: [
                    Expanded(
                      child: CupertinoButton.filled(
                        onPressed:
                            _webViewController == null ? null : _onSend,
                        child: const Text('Send'),
                      ),
                    ),
                    const SizedBox(width: 12),
                    Expanded(
                      child: CupertinoButton.filled(
                        onPressed:
                            _webViewController == null ? null : _onSendAsync,
                        child: const Text('SendAsync'),
                      ),
                    ),
                  ],
                ),
              ),
            ),
        ],
      ),
    );
  }
}
```

---

### 7. GoRouter 등록

기존 라우터 설정 파일에 `/test-webview` 경로를 추가해 주세요.

```dart
// lib/core/router/app_router.dart

import '../../test/webview/test_webview_screen.dart';

// routes 목록에 추가:
GoRoute(
  path: '/test-webview',
  builder: (context, state) => const TestWebviewScreen(),
),
```

`TestScreen`에서 이동하려면 `test_screen.dart` 내 버튼 중 하나에 아래와 같이 연결합니다.

```dart
// test_viewmodel.dart의 _onButton1Pressed() 등에서:
onGoToTestWebview?.call();

// test_screen.dart에서 콜백 주입:
vm.onGoToTestWebview = () => context.push('/test-webview');
```

---

## Notes

- `TestWebView`는 `CommonInterface`를 내장하여 JS 브릿지를 자동으로 설정합니다. 추가 인터페이스가 필요하면 `test_interface.dart`에 새 `JsInterface` 구현체를 추가하고 `test_webview.dart`의 `_interfaces` 목록에 등록합니다.
- `onControllerReady`는 `initState` 이후 비동기로 호출될 수 있으므로, `_webViewController`를 nullable로 유지하고 null-safe 호출(`?.`)을 사용합니다.
- `resizeToAvoidBottomInset: false`로 설정하여 키보드가 올라와도 WebView 레이아웃이 밀리지 않도록 합니다.
- `test_interface.dart`와 `test_webview.dart`는 `lib/test/webview/` 내에 자체 포함되므로 `presentation/main/`의 동명 파일과 독립적으로 관리됩니다.
