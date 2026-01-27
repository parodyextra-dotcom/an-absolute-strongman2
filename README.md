# 절대강자 가계부

스마트한 자산관리를 위한 가계부 웹 애플리케이션입니다.

## 주요 기능

### 완료된 기능
- ✅ 수입/지출 거래 관리
- ✅ 카테고리별 지출 분류
- ✅ 통계 및 차트 분석
- ✅ 목표 설정 및 추적
- ✅ 투자 관리 (배당금, 투자일지)
- ✅ 환전 관리 (USD/JPY)
- ✅ 대출 관리
- ✅ 은행 잔고 관리
- ✅ 자산 등급 시스템 (기초수급자 ~ 절대강자)
- ✅ PWA 지원 (오프라인 사용 가능)
- ✅ **Flutter WebView 연동 지원**
- ✅ **고정비 관리 (월 고정비 자동 입력)**

## 파일 구조

```
index.html    # 메인 애플리케이션 (단일 파일)
README.md     # 프로젝트 문서
```

## 진입점

- `/index.html` - 메인 가계부 애플리케이션

---

## Flutter WebView 연동 가이드

### JavaScript Bridge 구조

웹 앱은 Flutter WebView와 통신하기 위해 `FlutterBridge` 객체를 사용합니다.

### 지원되는 채널 이름

다음 JavaScript 채널 중 하나를 사용하여 Flutter와 통신합니다:
- `FlutterChannel`
- `flutter_inappwebview`
- `FlutterWebView`
- `JavaScriptChannel`

### Flutter에서 받는 메시지 형식

```json
{
  "action": "string",
  "data": { ... }
}
```

### Action 종류

#### 1. `exportJSON` - JSON 데이터 백업
```json
{
  "action": "exportJSON",
  "data": {
    "content": "JSON 문자열",
    "filename": "budget_backup_2025-01-25.json",
    "mimeType": "application/json"
  }
}
```

#### 2. `exportCSV` - CSV 거래내역 내보내기
```json
{
  "action": "exportCSV",
  "data": {
    "content": "CSV 문자열 (BOM 포함)",
    "filename": "거래내역_2025-01-25.csv",
    "mimeType": "text/csv"
  }
}
```

#### 3. `requestImport` - 파일 가져오기 요청
```json
{
  "action": "requestImport",
  "data": {
    "accept": ".json",
    "mimeTypes": ["application/json"]
  }
}
```

#### 4. `confirmClearData` - 데이터 삭제 확인 요청
```json
{
  "action": "confirmClearData",
  "data": {}
}
```

#### 5. `dataChanged` - 데이터 변경 알림
```json
{
  "action": "dataChanged",
  "data": {
    "type": "import" | "clearAll" | "fixedExpenseDeleted" | "fixedExpensePermanentDeleted" | "fixedExpenseEdited" | "fixedExpenseAdded"
  }
}
```

#### 6. `fixedExpenseChanged` - 고정비 변경 알림 (신규)
```json
{
  "action": "fixedExpenseChanged",
  "data": {
    "action": "delete" | "permanentDelete" | "edit" | "add",
    "data": { ... }
  }
}
```

#### 7. `requestFixedExpenses` - 고정비 목록 요청 응답 (신규)
```json
{
  "action": "requestFixedExpenses",
  "data": [
    {
      "id": 1234567890,
      "startDate": "2025-01-01",
      "endDate": "2025-12-31",
      "payDay": 25,
      "amount": 100000,
      "paymentMethod": "card",
      "category": "보험료",
      "memo": "실비보험",
      "active": true,
      "createdAt": "2025-01-01T00:00:00.000Z"
    }
  ]
}
```

---

### Flutter에서 JavaScript 호출하기

#### 데이터 가져오기 후 처리
```dart
// JSON 파일 내용을 읽은 후 웹뷰에 전달
await controller.runJavaScript(
  'handleFlutterImport(\'${jsonEncode(fileContent)}\');'
);
```

#### 데이터 삭제 확인 후 처리
```dart
// 사용자가 삭제 확인 다이얼로그에서 확인을 누른 경우
await controller.runJavaScript('handleFlutterClearData();');

// 또는
await controller.runJavaScript('performResetAllData();');
```

#### 고정비 관련 함수 호출 (신규)
```dart
// 고정비 목록 가져오기
String result = await controller.runJavaScriptReturningResult(
  'handleFlutterGetFixedExpenses();'
);
List<dynamic> fixedExpenses = jsonDecode(result);

// 고정비 삭제 (비활성화)
await controller.runJavaScript(
  'handleFlutterDeleteFixedExpense(${id});'
);

// 고정비 완전 삭제
await controller.runJavaScript(
  'handleFlutterPermanentDeleteFixedExpense(${id});'
);

// 고정비 수정
await controller.runJavaScript(
  'handleFlutterEditFixedExpense(${id}, ${amount}, "${endDate}");'
);

// 고정비 추가
await controller.runJavaScript(
  'handleFlutterAddFixedExpense(\'${jsonEncode(fixedExpenseData)}\');'
);

// 전체 데이터 가져오기
String allData = await controller.runJavaScriptReturningResult(
  'handleFlutterGetAllData();'
);
```

---

### Flutter 코드 예제 (webview_flutter)

```dart
import 'dart:convert';
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';
import 'package:path_provider/path_provider.dart';
import 'package:share_plus/share_plus.dart';
import 'package:file_picker/file_picker.dart';

class BudgetWebView extends StatefulWidget {
  @override
  _BudgetWebViewState createState() => _BudgetWebViewState();
}

class _BudgetWebViewState extends State<BudgetWebView> {
  late WebViewController _controller;

  @override
  void initState() {
    super.initState();
    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..addJavaScriptChannel(
        'FlutterChannel',
        onMessageReceived: _handleMessage,
      )
      ..loadRequest(Uri.parse('YOUR_WEB_APP_URL'));
  }

  void _handleMessage(JavaScriptMessage message) async {
    final data = jsonDecode(message.message);
    final action = data['action'];
    final payload = data['data'];

    switch (action) {
      case 'exportJSON':
        await _exportFile(
          payload['content'],
          payload['filename'],
          payload['mimeType'],
        );
        break;
      case 'exportCSV':
        await _exportFile(
          payload['content'],
          payload['filename'],
          payload['mimeType'],
        );
        break;
      case 'requestImport':
        await _importFile();
        break;
      case 'confirmClearData':
        await _showClearDataConfirmation();
        break;
      case 'dataChanged':
        // 데이터 변경 알림 처리
        _handleDataChanged(payload['type']);
        break;
      case 'fixedExpenseChanged':
        // 고정비 변경 알림 처리
        _handleFixedExpenseChanged(payload);
        break;
    }
  }
  
  void _handleDataChanged(String type) {
    print('Data changed: $type');
    // 필요한 경우 UI 업데이트 또는 추가 처리
    switch (type) {
      case 'fixedExpenseDeleted':
      case 'fixedExpensePermanentDeleted':
      case 'fixedExpenseEdited':
      case 'fixedExpenseAdded':
        // 고정비 관련 변경 처리
        print('Fixed expense changed: $type');
        break;
    }
  }
  
  void _handleFixedExpenseChanged(Map<String, dynamic> payload) {
    print('Fixed expense action: ${payload['action']}');
    // 필요한 경우 추가 처리
  }

  Future<void> _exportFile(String content, String filename, String mimeType) async {
    try {
      final directory = await getTemporaryDirectory();
      final file = File('${directory.path}/$filename');
      await file.writeAsString(content);
      
      await Share.shareXFiles(
        [XFile(file.path)],
        text: '가계부 데이터',
      );
    } catch (e) {
      print('Export error: $e');
    }
  }

  Future<void> _importFile() async {
    try {
      FilePickerResult? result = await FilePicker.platform.pickFiles(
        type: FileType.custom,
        allowedExtensions: ['json'],
      );

      if (result != null) {
        File file = File(result.files.single.path!);
        String content = await file.readAsString();
        
        // Escape for JavaScript
        String escaped = content
            .replaceAll('\\', '\\\\')
            .replaceAll("'", "\\'")
            .replaceAll('\n', '\\n')
            .replaceAll('\r', '\\r');
        
        await _controller.runJavaScript("handleFlutterImport('$escaped');");
      }
    } catch (e) {
      print('Import error: $e');
    }
  }

  Future<void> _showClearDataConfirmation() async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('데이터 삭제'),
        content: Text('정말 모든 데이터를 삭제하시겠습니까?\n이 작업은 되돌릴 수 없습니다.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('취소'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            style: TextButton.styleFrom(foregroundColor: Colors.red),
            child: Text('삭제'),
          ),
        ],
      ),
    );

    if (confirmed == true) {
      await _controller.runJavaScript('handleFlutterClearData();');
    }
  }
  
  // 고정비 목록 가져오기 예제
  Future<List<dynamic>> getFixedExpenses() async {
    final result = await _controller.runJavaScriptReturningResult(
      'handleFlutterGetFixedExpenses();'
    );
    return jsonDecode(result as String);
  }
  
  // 고정비 삭제 예제
  Future<void> deleteFixedExpense(int id) async {
    await _controller.runJavaScript(
      'handleFlutterDeleteFixedExpense($id);'
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: WebViewWidget(controller: _controller),
      ),
    );
  }
}
```

---

### Flutter 코드 예제 (flutter_inappwebview)

```dart
import 'dart:convert';
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_inappwebview/flutter_inappwebview.dart';
import 'package:path_provider/path_provider.dart';
import 'package:share_plus/share_plus.dart';
import 'package:file_picker/file_picker.dart';

class BudgetWebView extends StatefulWidget {
  @override
  _BudgetWebViewState createState() => _BudgetWebViewState();
}

class _BudgetWebViewState extends State<BudgetWebView> {
  InAppWebViewController? _controller;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: InAppWebView(
          initialUrlRequest: URLRequest(
            url: WebUri('YOUR_WEB_APP_URL'),
          ),
          initialSettings: InAppWebViewSettings(
            javaScriptEnabled: true,
          ),
          onWebViewCreated: (controller) {
            _controller = controller;
            
            // JavaScript 핸들러 등록
            controller.addJavaScriptHandler(
              handlerName: 'FlutterChannel',
              callback: (args) async {
                if (args.isNotEmpty) {
                  await _handleMessage(args[0]);
                }
                return null;
              },
            );
          },
        ),
      ),
    );
  }

  Future<void> _handleMessage(String message) async {
    final data = jsonDecode(message);
    final action = data['action'];
    final payload = data['data'];

    switch (action) {
      case 'exportJSON':
      case 'exportCSV':
        await _exportFile(
          payload['content'],
          payload['filename'],
        );
        break;
      case 'requestImport':
        await _importFile();
        break;
      case 'confirmClearData':
        await _showClearDataConfirmation();
        break;
      case 'dataChanged':
        _handleDataChanged(payload['type']);
        break;
      case 'fixedExpenseChanged':
        _handleFixedExpenseChanged(payload);
        break;
    }
  }
  
  void _handleDataChanged(String type) {
    print('Data changed: $type');
  }
  
  void _handleFixedExpenseChanged(Map<String, dynamic> payload) {
    print('Fixed expense action: ${payload['action']}');
  }

  Future<void> _exportFile(String content, String filename) async {
    final directory = await getTemporaryDirectory();
    final file = File('${directory.path}/$filename');
    await file.writeAsString(content);
    await Share.shareXFiles([XFile(file.path)]);
  }

  Future<void> _importFile() async {
    FilePickerResult? result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: ['json'],
    );

    if (result != null && _controller != null) {
      File file = File(result.files.single.path!);
      String content = await file.readAsString();
      String escaped = jsonEncode(content);
      await _controller!.evaluateJavascript(
        source: "handleFlutterImport($escaped);"
      );
    }
  }

  Future<void> _showClearDataConfirmation() async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('데이터 삭제'),
        content: Text('정말 모든 데이터를 삭제하시겠습니까?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('취소'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: Text('삭제'),
          ),
        ],
      ),
    );

    if (confirmed == true && _controller != null) {
      await _controller!.evaluateJavascript(
        source: 'handleFlutterClearData();'
      );
    }
  }
  
  // 고정비 목록 가져오기
  Future<List<dynamic>> getFixedExpenses() async {
    if (_controller == null) return [];
    final result = await _controller!.evaluateJavascript(
      source: 'handleFlutterGetFixedExpenses();'
    );
    return jsonDecode(result as String);
  }
  
  // 고정비 추가
  Future<void> addFixedExpense(Map<String, dynamic> data) async {
    if (_controller == null) return;
    await _controller!.evaluateJavascript(
      source: "handleFlutterAddFixedExpense('${jsonEncode(data)}');"
    );
  }
}
```

---

## 전역 JavaScript 함수 (Flutter에서 호출 가능)

| 함수명 | 설명 | 파라미터 | 반환값 |
|--------|------|----------|--------|
| `handleFlutterImport(jsonString)` | JSON 데이터 가져오기 | JSON 문자열 | boolean |
| `handleFlutterClearData()` | 모든 데이터 삭제 | 없음 | boolean |
| `performResetAllData()` | 모든 데이터 삭제 (동일 기능) | 없음 | void |
| `handleFlutterGetFixedExpenses()` | 고정비 목록 가져오기 | 없음 | JSON 문자열 |
| `handleFlutterDeleteFixedExpense(id)` | 고정비 삭제 (비활성화) | id (number) | boolean |
| `handleFlutterPermanentDeleteFixedExpense(id)` | 고정비 완전 삭제 | id (number) | boolean |
| `handleFlutterEditFixedExpense(id, amount, endDate)` | 고정비 수정 | id, amount, endDate | boolean |
| `handleFlutterAddFixedExpense(jsonData)` | 고정비 추가 | JSON 문자열 | boolean |
| `handleFlutterGetAllData()` | 전체 데이터 가져오기 | 없음 | JSON 문자열 |

---

## 고정비(fixedExpenses) 데이터 구조

```json
{
  "id": 1234567890,
  "startDate": "2025-01-01",
  "endDate": "2025-12-31",
  "payDay": 25,
  "amount": 100000,
  "paymentMethod": "card",
  "category": "보험료",
  "memo": "실비보험",
  "active": true,
  "createdAt": "2025-01-01T00:00:00.000Z"
}
```

### 필드 설명

| 필드명 | 타입 | 설명 |
|--------|------|------|
| `id` | number | 고유 ID (timestamp 기반) |
| `startDate` | string | 시작 날짜 (YYYY-MM-DD) |
| `endDate` | string/null | 종료 날짜 (YYYY-MM-DD), null이면 무기한 |
| `payDay` | number | 매월 출금일 (1-28) |
| `amount` | number | 금액 |
| `paymentMethod` | string | 결제수단 (card/cash/other) |
| `category` | string | 카테고리 |
| `memo` | string | 메모 |
| `active` | boolean | 활성 상태 |
| `createdAt` | string | 생성 시간 (ISO 8601) |

---

## 백업 데이터 구조 (전체)

```json
{
  "transactions": [],
  "memos": [],
  "goals": { "year7": "", "year5": "", "year3": "", "thisYear": "" },
  "bankBalances": [],
  "investments": [],
  "dividends": [],
  "investmentJournals": [],
  "loans": [],
  "events": [],
  "categories": {
    "expense": ["식비", "교통비", ...],
    "income": ["급여", "이자", ...],
    "fixedExpense": ["보험료", ...]
  },
  "fixedExpenses": [],
  "exchangeData": {
    "usdPhases": [],
    "jpyPhases": [],
    "usdRate": 1380,
    "jpyRate": 920,
    "usdNextId": 1,
    "jpyNextId": 1,
    "usdInitialBudget": 10000000,
    "jpyInitialBudget": 10000000
  },
  "exportDate": "2025-01-25T00:00:00.000Z",
  "version": "2.1"
}
```

---

## 기술 스택

- HTML5 / CSS3 / JavaScript (ES5)
- TailwindCSS 4 (CDN)
- Lucide Icons
- PWA (Progressive Web App)

## 버전 정보

- 버전: 2.1
- Flutter WebView 연동 지원 버전: 2.1
- 고정비 Flutter 연동 지원 버전: 2.1

## 변경 이력

### v2.1 (2025-01-25)
- Flutter WebView에서 고정비(fixedExpenses) 데이터 동기화 문제 수정
- `handleFlutterImport` 함수에서 `fixedExpenses` 데이터 처리 추가
- `handleFlutterClearData` 함수에서 `fixedExpenses` 초기화 추가
- 고정비 조작을 위한 전역 JavaScript 함수 추가:
  - `handleFlutterGetFixedExpenses()`
  - `handleFlutterDeleteFixedExpense(id)`
  - `handleFlutterPermanentDeleteFixedExpense(id)`
  - `handleFlutterEditFixedExpense(id, amount, endDate)`
  - `handleFlutterAddFixedExpense(jsonData)`
  - `handleFlutterGetAllData()`
- FlutterBridge에 고정비 관련 메시지 추가
- exportData에 version 정보 및 exchangeData 초기예산 포함

### v2.0
- 기본 Flutter WebView 연동 지원
- PWA 지원

## 라이선스

Private Use
