카카오페이 연동을 위한 안드로이드 Native 코드 수정을 위한 가이드 문서입니다.

## 카카오톡 앱으로 이동  
결제를 위해 사용하시는 안드로이드 WebView 내에서, 아임포트 JavaScript 함수 인 `IMP.request_pay()`가 호출되면 카카오페이 결제화면이 나타나게 되는데 `카카오톡` 이라는 외부 앱으로 이동하여 사용자 인증을 진행하게 됩니다.   
이를 위해서는 WebView 에 WebViewClient.shouldOverrideUrlLoading() 메소드가 구현되어 `카카오톡` 이동을 할 수 있도록 Intent 호출처리를 하여야 합니다.  

### WebView 를 포함한 Activity 에서의 코드 예제  

```java
payWebView.setWebViewClient(new IamportWebViewClient(this, payWebView));
```

```java
public class IamportWebViewClient extends WebViewClient {

	@Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {

        if (!url.startsWith("http://") && !url.startsWith("https://") && !url.startsWith("javascript:")) {
            Intent intent = null;

            try {
                intent = Intent.parseUri(url, Intent.URI_INTENT_SCHEME); //IntentURI처리
                Uri uri = Uri.parse(intent.getDataString());

                activity.startActivity(new Intent(Intent.ACTION_VIEW, uri));
                return true;
            } catch (URISyntaxException ex) {
                return false;
            } catch (ActivityNotFoundException e) {
                if (intent == null) return false;

                if (handleNotFoundPaymentScheme(intent.getScheme())) return true;

                String packageName = intent.getPackage();
                if (packageName != null) {
                    activity.startActivity(new Intent(Intent.ACTION_VIEW, Uri.parse("market://details?id=" + packageName)));
                    return true;
                }

                return false;
            }
        }

        return false;
    }

}
```

### 카카오톡이 설치되지 않은 사용자를 위한 처리  

JavaScript를 통한 카카오톡 앱 호출 시 intent scheme 에 맞게 URL호출을 하게 되므로, 카카오톡이 설치되지 않은 단말이라 하더라도 intent URL로부터 설치해야하는 package 정보를 추출할 수 있습니다.  
따라서, 위 예제코드 중 `ActivityNotFoundException` 처리부에서 package 정보를 추출하여 Google PlayStore 로 이동시키는 로직까지 포함되어있습니다. 

```java
    if (intent == null) return false;

    if (handleNotFoundPaymentScheme(intent.getScheme())) return true;

    String packageName = intent.getPackage();
    if (packageName != null) {
        activity.startActivity(new Intent(Intent.ACTION_VIEW, Uri.parse("market://details?id=" + packageName)));
        return true;
    }

    return false;
```

## 카카오톡 앱내 인증 후 WebView 로 복귀   

카카오톡 내에서 결제할 카드 또는 머니 수단선택 후, 지문인증/페이스아이디/비밀번호 인증을 성공적으로 마치면 결제를 호출한 WebView의 Activity로 자동 복귀하게 됩니다.  
해당 페이지에서 `결제완료` 버튼을 최종적으로 사용자가 한 번 더 클릭하거나 일정시간이 지나면 자동으로 최종 요청이 submit되면서 승인완료처리가 됩니다.  

카카오톡으로 이동하기 직전에 남아있던 웹페이지에서 위 동작이 가능하므로, 직전 웹페이지가 유지될 수 있도록 Activity onResume 등 시점에 WebView 를 초기화하거나 다른 링크로 강제이동시키지 않아야 합니다.   
