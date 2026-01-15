# Laravel에서 프록시 사용하기

[![Bright Data Promo](https://github.com/bright-kr/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/)

이 가이드는 차단 없는 Web스크레이핑 및 지오ロ케ーション 기반 접근 제어를 위해 Laravel 프로젝트에서 프록시를 구성하고 구현하는 방법을 설명합니다.

- [Laravel Proxy란 무엇입니까?](#what-is-a-laravel-proxy)
- [Laravel에서 プロキ시 사용 사례](#use-cases-for-proxies-in-laravel)
- [Laravel에서 プロキ시 사용하기: 단계별 가이드](#using-a-proxy-in-laravel-step-by-step-guide)
- [고급 사용 사례](#advanced-use-cases)
- [Laravel에서 Bright Data プロキ시 사용하기](#use-a-bright-data-proxy-in-laravel)
- [추가: Symfony의 `HttpClient` プロキ시 통합](#extra-symfonys-httpclient-proxy-integration)

## What Is a Laravel Proxy?

Laravel プロキ시는 Laravel 애플리케이션과 외부 서버 사이에서 중개자 역할을 합니다. 이를 통해 서버 트래픽을 프로그래밍 방식으로 [프로キ시 서버를 통해](https://brightdata.co.kr/blog/proxy-101/what-is-proxy-server) 전달하여 IP 주소를 숨길 수 있습니다.

Laravel에서 プロキ시가 동작하는 방식은 다음과 같습니다:

1. Laravel이 プロキ시 구성이 포함된 HTTP 클라이언트 라이브러리를 사용하여 HTTP 요청를 시작합니다.
2. リクエ스트가 プロキ시 서버를 통해 이동합니다.
3. プロキ시가 이를 대상 서버로 전송합니다.
4. 대상 서버가 응답를 プロキ시에 반환합니다.
5. プロキ시가 응답를 Laravel로 중계합니다.

그 결과, 대상 서버는 리クエ스트가 Laravel 서버가 아니라 プロキ시의 IP에서 발생한 것으로 인식합니다. 이 메커니즘은 지리적 제한 우회, 익명성 강화, 속도 제한 처리에 도움이 됩니다.

## Use Cases for Proxies in Laravel

Laravel에서 プロキ시는 다양한 목적에 사용되지만, 특히 다음 세 가지가 흔합니다:

- **Web스크레이핑**: Web스크레이핑 API를 만들 때 IP 차단을 방지하고, 속도 제한을 우회하거나, 기타 제한을 피하기 위해 プロキ시를 구현합니다. 추가 정보는 [Laravel로 web scraping하기](https://brightdata.co.kr/blog/web-data/web-scraping-with-laravel) 튜토리얼을 참고하십시오.
- **서드파티 API의 속도 제한 우회**: プロキ시 IP를 번갈아 사용하여 API 사용량 쿼터 내에 머무르고 스로틀링을 방지합니다.
- **지오 제한 콘텐츠 접근**: 특정 국가에서만 사용 가능한 서비스를 이용하기 위해 특정 위치의 プロキ시 서버를 선택합니다.

추가 예시는 [web data 및 プロキ시 사용 사례](https://brightdata.co.kr/use-cases) 가이드를 참고하십시오.

## Using a Proxy in Laravel: Step-By-Step Guide

이 섹션에서는 [기본 HTTP client](https://laravel.com/docs/master/http-client)를 사용하여 Laravel에 プロ키시를 통합하는 방법을 시연합니다. 또한 본문 후반부에서 [Symfony `HttpClient`](https://symfony.com/doc/current/http_client.html) 라이브러리와의 プロ키시 통합도 다룹니다.

> **Note**:
> 
> Laravel의 HTTP client는 Guzzle 기반이므로, [Guzzle プロ키시 통합 가이드](https://brightdata.co.kr/blog/how-tos/proxy-with-guzzle)도 함께 확인하시는 것이 좋습니다.

통합을 설명하기 위해 다음을 수행하는 `GET` `/api/v1/get-ip` 엔드포인트를 구성하겠습니다:

1. 구성된 プロ키시를 사용하여 [`https://httpbin.io/ip`](https://httpbin.io/ip)에 `GET` 요청를 전송합니다.
2. 응답에서 출구 IP 주소를 추출합니다.
3. 해당 IP를 Laravel 엔드포인트를 호출한 클라이언트에 반환합니다.

모든 것이 올바르게 구성되었다면, API가 반환하는 IP는 プロ키시의 IP 주소와 일치합니다.

### Step #1: Project Setup

이미 Laravel 애플리케이션이 구성되어 있다면 Step #2로 건너뛰셔도 됩니다.

그렇지 않다면 다음 지침에 따라 새 Laravel 프로젝트를 생성하십시오. 터미널을 열고 아래 Composer [`create-command`](https://getcomposer.org/doc/03-cli.md#create-project)를 실행하여 새 Laravel 프로젝트를 초기화합니다:

```sh
composer create-project laravel/laravel laravel-proxies-application
```

이 명령은 `laravel-proxies-application`이라는 디렉터리에 새 Laravel 프로젝트를 생성합니다. 선호하는 PHP IDE로 이 폴더를 여십시오.

이 시점에서 디렉터리에는 표준 Laravel 파일 구조가 포함되어 있어야 합니다:

![The Laravel projext structure](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/The-Laravel-projext-structure.png)

### Step #2: Define the Test API Endpoint

프로젝트 디렉터리에서 다음 [Artisan command](https://laravel.com/docs/11.x/artisan)를 실행하여 새 controller를 생성합니다:

```sh
php artisan make:controller IPController
```

그러면 `/app/Http/Controllers` 디렉터리에 `IPController.php` 파일이 생성되며 기본 내용은 다음과 같습니다:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class IPController extends Controller
{
    //
}
```

이제 아래 `getIP()` 메서드를 `IPController.php`에 추가하십시오:

```php
public function getIP(): JsonResponse
{
    // make a GET request to the "/ip" endpoint to get the IP of the server
    $response = Http::get('https://httpbin.io/ip'); 

    // retrieve the response data
    $responseData = $response->json();

    // return the response data 
    return response()->json($responseData);
}
```

이 메서드는 Laravel의 `Http` client를 사용해 `https://httpbin.io/ip`에서 IP 주소를 가져와 JSON으로 반환합니다.

다음 두 import를 포함하는 것을 잊지 마십시오:

```php
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;
```

Laravel 애플리케이션이 stateless API를 제공하도록 하려면 [`install:api`](https://laravel.com/docs/master/routing#api-routes) Artisan command로 API routing을 활성화하십시오:

```sh
php artisan install:api
```

이 메서드를 API 엔드포인트로 노출하려면 `routes/api.php` 파일에 다음 route를 추가하십시오:

```php
use App\Http\Controllers\IPController;

Route::get('/v1/get-ip', [IPController::class, 'getIP']);
```

새 API 엔드포인트는 다음에서 접근할 수 있습니다:

```php
/api/v1/get-ip
```

**Note**: Laravel의 모든 API는 기본적으로 `/api` 경로 아래에서 제공된다는 점을 기억하십시오.

이제 `/api/v1/get-ip` 엔드포인트를 테스트할 시간입니다!

다음 명령을 실행하여 Laravel 개발 서버를 실행하십시오:

```sh
php artisan serve
```

이제 서버는 로컬의 `8000` 포트에서 대기하고 있어야 합니다.

cURL을 사용하여 `/api/v1/get-ip` 엔드포인트로 `GET` リクエ스트를 보냅니다:

```sh
curl -X GET 'http://localhost:8000/api/v1/get-ip' 
```

**Note**: Windows에서는 `curl` 대신 `curl.exe`를 사용하십시오. 자세한 내용은 [cURL로 GET 요청 보내는 방법](https://brightdata.co.kr/faqs/curl/curl-get-requests) 가이드를 참고하십시오.

다음과 유사한 응답를 받아야 합니다:

```json
{
  "origin": "45.89.222.18"
}
```

이 응답는 HttpBin의 `/ip` 엔드포인트가 생성하는 것과 정확히 일치하며, Laravel API가 정상적으로 작동함을 확인해 줍니다. 특히 표시되는 IP 주소는 사용 중인 머신의 공인 IP입니다.

### Step #3: Retrieve a Proxy

Laravel 애플리케이션에서 プロキ시를 사용하려면 먼저 동작하는 プロ키시 서버에 접근할 수 있어야 합니다.

일반적인 プロ키시 URL은 다음 형식을 따릅니다:

```
<protocol>://<host>:<port>
```

여기서:

- `protocol`은 プロ키시 서버에 연결하는 데 필요한 프로토콜입니다(예: `http`, `https`, `socks5`)
- `host`는 プロ키시 서버의 IP 주소 또는 도메인입니다
- `port`는 트래픽을 라우팅하는 데 사용되는 포트입니다

이 예시에서는 プロ키시 URL이 다음과 같다고 가정합니다:

```
http://66.29.154.103:3128
```

이를 `getIP()` 메서드 안의 변수에 저장하십시오:

```php
$proxyUrl = 'http://66.29.154.103:3128';
```

### Step #4: Integrate the Proxy in `Http`

`Http` client를 사용하여 Laravel에 プロ키시를 통합하려면 추가 설정이 거의 필요하지 않습니다:

```php
$proxyUrl = 'http://66.29.154.103:3128';

$response = Http::withOptions([
    'proxy' => $proxyUrl
])->get('https://httpbin.io/ip');

$responseData = $response->json();
```

[`withOptions()`](https://laravel.com/docs/11.x/http-client#guzzle-options) 메서드를 사용하여 옵션으로 プロ키시 URL을 전달해야 합니다. 이는 Laravel의 HTTP client에 [Guzzle의 `proxy` 옵션](https://docs.guzzlephp.org/en/stable/request-options.html#proxy)을 사용하여 지정된 プロ키시 서버를 통해 요청를 라우팅하도록 지시합니다.

### Step #5: Put It All Together

이제 プロ키시 통합이 포함된 최종 Laravel API 로직은 다음과 같아야 합니다:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

class IPController extends Controller
{

    public function getIP(): JsonResponse
    {
        // the URL of the proxy server
        $proxyUrl = 'http://66.29.154.103:3128'; // replace with your proxy URL

        // make a GET request to /ip endpoint through the proxy server
        $response = Http::withOptions([
            'proxy' => $proxyUrl
        ])->get('https://httpbin.io/ip');

        // retrieve the response data
        $responseData = $response->json();

        // return the response data
        return response()->json($responseData);
    }
}
```

Laravel을 로컬에서 실행하여 테스트하십시오:

```sh
php artisan serve
```

그리고 `/api/v1/get-ip` 엔드포인트에 다시 연결합니다:

```sh
curl -X GET 'http://localhost:8000/api/v1/get-ip' 
```

이번에는 출력이 다음과 유사해야 합니다:

```json
{
  "origin": "66.29.154.103"
}
```

`"origin"` 필드는 プロ키시 서버의 IP 주소를 표시하므로, 이제 실제 IP가 プロ키시 뒤에 숨겨졌음을 확인할 수 있습니다.

> **Warning**:
> 
> 무료 プロ키시 서버는 일반적으로 불안정하거나 수명이 짧습니다. 여러분이 시도할 때쯤에는 예시 プロ키시가 더 이상 동작하지 않을 수 있습니다. 필요하다면 테스트 전에 `$proxyUrl`을 현재 활성화된 것으로 교체하십시오.

リクエ스트 수행 중 SSL 오류가 발생하면, 아래 고급 사용 사례 섹션에 제공된 문제 해결 팁을 참고하십시오.

## Advanced Use Cases

이제 Laravel에서의 プロ키시 통합 기본을 마스터하셨지만, 추가로 고려할 고급 시나리오가 있습니다.

### Proxy Authentication

프리미엄 プロ키시는 종종 인증을 요구하여 승인된 사용자만 접근하도록 합니다. 올바른 자격 증명이 없으면 다음 오류가 발생합니다:

```
cURL error 56: CONNECT tunnel failed, response 407
```

인증이 필요한 プロ키시 URL은 일반적으로 다음 형식을 따릅니다:

```
<protocol>://<username>:<password>@<host>:<port>
```

여기서 `username`과 `password`는 인증 자격 증명입니다.

(내부적으로 Guzzle을 사용하는) Laravel의 `Http` 클래스는 인증 プロ키시를 완전히 지원합니다. 큰 수정은 필요 없으며, プロ키시 URL에 인증 자격 증명을 직접 포함하기만 하면 됩니다:

```php
$proxyUrl = '<protocol>://<username>:<password>@<host>:<port>';
```

예를 들면 다음과 같습니다:

```php
// authenticated proxy with username and password
$proxyUrl = 'http://<username>:<password>@<host>:<port>';

$response = Http::withOptions([
    'proxy' => $proxyUrl
])->get('https://httpbin.io/ip');
```

`$proxyUrl` 값을 유효한 인증 プロ키시 URL로 바꾸십시오.

이제 `Http`가 구성된 인증 プロ키시 서버로 트래픽을 전달합니다!

### Avoid SSL Certificate Issues

Laravel의 `Http` client로 プロ키시를 구성할 때, 다음과 같은 SSL 인증서 검증 오류로 リクエ스트가 실패할 수 있습니다:

```
cURL error 60: SSL certificate problem: self-signed certificate in certificate chain
```

이는 보통 プロ키시 서버가 self-signed [SSL certificate](https://www.cloudflare.com/learning/ssl/what-is-an-ssl-certificate/)를 사용할 때 발생합니다.

プロ키시 서버를 신뢰하며 로컬 테스트 또는 보안된 환경에서만 사용한다면 SSL 검증을 비활성화할 수 있습니다:

```php
$response = Http::withOptions([
    'proxy' => $proxyUrl,
    'verify' => false, // disable SSL certificate verification
])->get('https://httpbin.io/ip');
```

> **Warning**:
> 
> SSL 검증을 비활성화하면 man-in-the-middle 공격에 취약해집니다. 신뢰할 수 있는 환경에서만 이 옵션을 사용하십시오.

또는 [프로키시 서버의 인증서 파일](https://docs.brightdata.com/general/account/ssl-certificate)(예: `proxy-ca.crt`)이 있다면 이를 SSL 검증에 사용할 수 있습니다:

```php
$response = Http::withOptions([
    'proxy' => $proxyUrl,
    'verify' => storage_path('certs/proxy-ca.crt'), // Path to the CA bundle
])->get('https://httpbin.io/ip');
```

`proxy-ca.crt` 파일이 안전하고 접근 가능한 디렉터리(예: `storage/certs/`)에 저장되어 있고, Laravel이 이를 읽을 권한이 있는지 확인하십시오.

어느 접근 방식을 적용하든, プरो키시로 인해 발생한 SSL 검증 오류는 해결되어야 합니다.

### Proxy Rotation

같은 プロ키시 서버를 반복해서 사용하면, 대상 웹사이트가 결국 해당 プロ키시의 IP 주소를 감지하고 차단할 가능성이 높습니다. 이를 방지하기 위해 각 リクエ스트마다 다른 것을 사용하는 방식으로 [プロ키시 서버를 로ーテ이션](https://brightdata.co.kr/blog/how-tos/how-to-rotate-an-ip-address)할 수 있습니다.

Laravel에서 プロ키시를 로ーテーション하는 단계는 다음과 같습니다:

1. 여러 プロ키시 URL을 포함하는 배열을 만듭니다
2. 각 リクエ스트 전에 임의로 하나를 선택합니다
3. 선택된 プロ키시를 HTTP client 구성에 설정합니다

다음 코드로 이를 구현하십시오:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

function getRandomProxyUrl(): string
{
    // a list of valid proxy URLs (replace with your proxy URLs)
    $proxies = [
        '<protocol_1>://<proxy_host_1>:<port_1>', 
        // ...
        '<protocol_n>://<proxy_host_n>:<port_n>',
    ];

    // return a proxy URL randomly picked from the list
    return $proxies[array_rand($proxies)];
}

class IPController extends Controller
{
    public function getIP(): JsonResponse
    {
        // the URL of the proxy server
        $proxyUrl = getRandomProxyUrl();
        // make a GET request to /ip endpoint through the proxy server
        $response = Http::withOptions([
            'proxy' => $proxyUrl
        ])->get('https://httpbin.io/ip');

        // retrieve the response data
        $responseData = $response->json();

        // return the response data
        return response()->json($responseData);
    }
}
```

이 스니펫은 목록에서 プロ키시를 무작위로 선택하여 로ーテーション을 구현하는 방법을 보여줍니다. 효과적이긴 하지만, 이 접근에는 한계가 있습니다:

1. 신뢰할 수 있는 プロ키시 서버 컬렉션을 유지해야 하며, 이는 일반적으로 비용이 듭니다.
2. 효과적인 로ーテーション을 위해서는 プロ키시 풀의 규모가 충분히 커야 합니다. プロ키시가 충분하지 않으면 동일한 서버가 반복 사용되어 감지 및 차단으로 이어질 수 있습니다.

이러한 과제를 극복하기 위해 [Bright Data의 로ーテーティングプロキ시 네트워크](https://brightdata.co.kr/solutions/rotating-proxies) 사용을 고려해 보십시오.

## Use a Bright Data Proxy in Laravel

다음 단계에 따라 Bright Data의 レジデンシャルプロキ시를 Laravel과 함께 구현하십시오.

아직 계정이 없다면 [Bright Data에 가입](https://brightdata.co.kr/cp/start)하십시오. 이미 계정이 있다면 로그인하여 사용자 대시보드에 접근하십시오:

![The Bright Data dashboard](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/The-Bright-Data-dashboard.png)

로그인 후 "Get proxy products" 버튼을 클릭하십시오:

![Clicking the "Get proxy products" button](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/Clicking-the-Get-proxy-products-button.png)

"Proxies & Scraping Infrastructure" 페이지로 이동합니다:

![The "Proxies & Scraping Infrastructure" page](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/The-Proxies-Scraping-Infrastructure-page-1.png)

테이블에서 "Residential" 행을 찾아 클릭하십시오:

![Clicking the "residential" row](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/Clicking-the-residential-row.png)

레ジデンシャルプロキ시 페이지로 이동합니다:

![The "residential" page](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/The-residential-page.png)

처음 사용하는 사용자는 설정 마법사를 따라 요구 사항에 맞게 プロ키시 서비스를 구성하십시오. 추가 지원이 필요하면 언제든지 [24/7 지원 팀에 문의](https://brightdata.co.kr/contact)하실 수 있습니다.

"Overview" 탭에서 プロ키시의 host, port, username, password를 확인하십시오:

![The proxy credentials](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/The-proxy-credentials.png)

이 정보를 사용해 プロ키시 URL을 구성하십시오:

```php
$proxyUrl = 'http://<brightdata_proxy_username>:<brightdata_proxy_password>@<brightdata_proxy_host>:<brightdata_proxy_port>';
```

플레이스홀더(`<brightdata_proxy_username>`, `<brightdata_proxy_password>`, `<brightdata_proxy_host>`, `<brightdata_proxy_port>`)를 실제 プロ키시 자격 증명으로 교체하십시오.

"Off" 스위치를 토글하여 プロ키시 제품을 활성화하고, 나머지 설정 지침을 완료하십시오:

![Clicking the activation toggle](https://github.com/bright-kr/laravel-with-proxy/blob/main/images/Clicking-the-activation-toggle.png)

プロ키시 URL이 구성되었으면, `Http` client를 사용해 Laravel에 통합할 수 있습니다. 다음은 Laravel에서 Bright Data의 로ーテーティング レジデンシャルプロキ시를 통해 リクエ스트를 보내는 코드입니다:

```php
public function getIp()
{
    // TODO: replace the placeholders with your Bright Data's proxy info
    $proxyUrl = 'http://<brightdata_proxy_username>:<brightdata_proxy_password>@<brightdata_proxy_host>:<brightdata_proxy_port>';

    // make a GET request to "/ip" through the proxy
    $response = Http::withOptions([
        'proxy' => $proxyUrl,
    ])->get('https://httpbin.org/ip');

    // get the response data
    $responseData = $response->json();

    return response()->json($responseData);
}
```

이 스크립트를 실행할 때마다 서로 다른 출구 IP를 확인할 수 있습니다.

## [Extra] Symfony's `HttpClient` Proxy Integration

Laravel 기본 HTTP client `Http`보다 Symfony의 `HttpClient` 컴포넌트를 선호한다면, 다음 지침에 따라 Laravel에서 `HttpClient`로 プロ키시 통합을 구현하십시오.

먼저 Composer를 통해 Symfony HTTP client 패키지를 설치합니다:

```sh
composer require symfony/http-client
```

다음으로 아래와 같이 プロ키시와 함께 Symfony의 `HttpClient`를 사용할 수 있습니다:

```php
<?php

namespace App\Http\Controllers;

use Symfony\Contracts\HttpClient\HttpClientInterface;
use Illuminate\Http\JsonResponse;

class IpController extends Controller
{
    // where to store the HttpClient instance
    private $client;

    public function __construct(HttpClientInterface $client)
    {
        // initialize the HttpClient private instance
        $this->client = $client;
    }

    public function getIp(): JsonResponse
    {
          // your proxy URL
          $proxyUrl = 'http://66.29.154.103:3128'; // replace with your proxy URL

          // make a GET request to the "/ip" endpoint through the proxy
          $response = $this->client->request('GET', 'https://httpbin.io/ip', [
              'proxy' => $proxyUrl,
          ]);

          // parse the response JSON and return it
          $responseData = $response->toArray();
          return response()->json($responseData);
    }
}
```

이 구성은 Symfony의 `HttpClient`를 사용하여 プロ키시를 통해 リクエ스트를 전송할 수 있게 해줍니다.

## Conclusion

무료 プロ키시 서비스는 신뢰할 수 없고 잠재적으로 위험할 수 있습니다. 일관된 성능, 보안, 확장성을 위해서는 신뢰할 수 있는 プロ키시 제공업체가 필요합니다. [시장 선도 プロ키시 제공업체](https://brightdata.co.kr/blog/proxy-101/best-proxy-providers) Bright Data를 선택하여 시간과 노력을 절약하십시오.

계정을 생성하고 오늘부터 무료로 당사의 プロ키시를 테스트해 보십시오!