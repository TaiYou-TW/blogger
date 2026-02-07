---
title: Laravel HTTP mock domain case-sensitive problem
date: 2022-08-21 15:30:11
tags:
    - laravel
    - mocking
---

## Introduction

簡單記錄一下之前在開發某個產品時踩到的雷，不過因為之後打算修正這個問題再發 PR，所以這邊就先用中文筆記一下問題，之前弄好的話再用英文寫一篇詳細的。

而這個雷就如同標題所述，是個 HTTP 這個 Facade 中的 mock function 的問題，會導致 mock 失效，害我當初卡超久 （；´д ｀）ゞ

<!-- more -->

## Problem

總之，問題是這樣的，東西寫完總要寫測試，寫完測試也都一切安好，但某天同事密我：「欸，我跑測試掛了，你那邊有這問題ㄇ」，於是開始檢查，看起來是串接外部 service 的測試掛了，但近期明明沒有改到那部分的 code，這就神奇了，開始追查原因，發現原因是即使 HTTP mock 了，他還是會直接打到外部服務，也就是 mock 失效。

而在找了快一天後，發現是第三方提供的 service 網址的 domain 含有大寫（例如：`https://Google.com`），而 Laravel mock 網址時，會 mock 完全一樣的網址，也就是含大寫的網址，但，HTTP facade 送出 request 時，domain 會轉成小寫，猜測是為了符合 RFC 1035 的規範。而這個不一致就導致了我 mock 的網址與實際送出的網址不符，才導致失效。

範例 code，我們先 mock `Google.com`，再 assert 他應為 404，但由於前述的問題，他會連上真實的 `google.com`，而不是我們自己 mock 的，這邊的測試會是 failed 的：

```php
Http::fake([
    'Google.com' => Http::response('Hello World2', 404),
]);
$this->get('google.com')->assertNotFound();
```

而為了確認 domain 的確是大小寫不敏感，也就是 `https://Google.com` 等同於 `https://google.com`，我去翻了 RFC 1035 的[白皮書](https://www.rfc-editor.org/rfc/rfc1035)，確認他裡面的定義，引述一下內容：

> Note that while upper and lower case letters are allowed in domain
> names, no significance is attached to the case. That is, two names with
> the same spelling but different case are to be treated as if identical.

## Code Trace

後來 Trace 了一下底層的 Code，由於 mock 的部分看起來是沒有對大小寫處理，因此這邊就先不探討，只研究 Http send request 的部分，而 Laravel 這邊底層的實作是使用 psr7，所以我去翻了他的 source code，看到他的確有把 uri 中 host 的部分由大寫轉小寫，這邊實際 trace 一次：

首先，我們簡單帶過前面的部分，只有最底層的 psr7 會講比較仔細（因為前面用 IDE trace 一下就有了 XD）。

這邊以 `post()` 為例：
`vendor\laravel\framework\src\Illuminate\Http\Client\PendingRequest.php#659`

```php
public function post(string $url, $data = [])
{
    return $this->send('POST', $url, [
        $this->bodyFormat => $data,
    ]);
}
```

`send`
`vendor\laravel\framework\src\Illuminate\Http\Client\PendingRequest.php#737`

```php
public function send(string $method, string $url, array $options = [])
{
    if (! Str::startsWith($url, ['http://', 'https://'])) {
        $url = ltrim(rtrim($this->baseUrl, '/').'/'.ltrim($url, '/'), '/');
    }

    $options = $this->parseHttpOptions($options);

    [$this->pendingBody, $this->pendingFiles] = [null, []];

    if ($this->async) {
        return $this->makePromise($method, $url, $options);
    }

    $shouldRetry = null;

    return retry($this->tries ?? 1, function ($attempt) use ($method, $url, $options, &$shouldRetry) {
        try {
            return tap(new Response($this->sendRequest($method, $url, $options)), function ($response) use ($attempt, &$shouldRetry) {
......
```

`sendRequest`
`vendor\laravel\framework\src\Illuminate\Http\Client\PendingRequest.php#874`

這邊他會視是否同步呼叫不同的 function，我們先追 `request` 就好，可以看到 `$this->buildClient()` 的 type 為 `\GuzzleHttp\Client`，所以我們繼續追。

```php
protected function sendRequest(string $method, string $url, array $options = [])
{
    $clientMethod = $this->async ? 'requestAsync' : 'request';

    $laravelData = $this->parseRequestData($method, $url, $options);

    return $this->buildClient()->$clientMethod($method, $url, $this->mergeOptions([
        'laravel_data' => $laravelData,
        'on_stats' => function ($transferStats) {
            $this->transferStats = $transferStats;
        },
    ], $options));
}
```

`request`
`vendor\guzzlehttp\guzzle\src\Client.php#184`

```php
public function request(string $method, $uri = '', array $options = []): ResponseInterface
{
    $options[RequestOptions::SYNCHRONOUS] = true;
    return $this->requestAsync($method, $uri, $options)->wait();
}
```

`requestAsync`
`vendor\guzzlehttp\guzzle\src\Client.php#152`

```php
public function requestAsync(string $method, $uri = '', array $options = []): PromiseInterface
{
    $options = $this->prepareDefaults($options);
    // Remove request modifying parameter because it can be done up-front.
    $headers = $options['headers'] ?? [];
    $body = $options['body'] ?? null;
    $version = $options['version'] ?? '1.1';
    // Merge the URI into the base URI.
    $uri = $this->buildUri(Psr7\Utils::uriFor($uri), $options);
    if (\is_array($body)) {
        throw $this->invalidBody();
    }
    $request = new Psr7\Request($method, $uri, $headers, $body, $version);
    // Remove the option so that they are not doubly-applied.
    unset($options['headers'], $options['body'], $options['version']);

    return $this->transfer($request, $options);
}
```

`uriFor`
`vendor\guzzlehttp\psr7\src\Utils.php#400`

```php
public static function uriFor($uri): UriInterface
{
    if ($uri instanceof UriInterface) {
        return $uri;
    }

    if (is_string($uri)) {
        return new Uri($uri);
    }

    throw new \InvalidArgumentException('URI must be a string or UriInterface');
}
```

再來，我們進到 psr7 的部分：

`vendor\guzzlehttp\psr7\src\Uri.php#80` 中可以看到 `Uri` 這個 class 的 `__consturct()` 的正常流程中，呼叫了 `applyParts()`：

```php
public function __construct(string $uri = '')
    {
        if ($uri !== '') {
            $parts = self::parse($uri);
            if ($parts === false) {
                throw new MalformedUriException("Unable to parse URI: $uri");
            }
            $this->applyParts($parts);
        }
    }
```

`vendor\guzzlehttp\psr7\src\Uri.php#538`
追進去，可以看到若有 host 時，會呼叫 `filterHost()`：

```php
private function applyParts(array $parts): void
{
    $this->scheme = isset($parts['scheme'])
        ? $this->filterScheme($parts['scheme'])
        : '';
    $this->userInfo = isset($parts['user'])
        ? $this->filterUserInfoComponent($parts['user'])
        : '';
    $this->host = isset($parts['host'])
        ? $this->filterHost($parts['host'])
        : '';
......
```

`vendor\guzzlehttp\psr7\src\Uri.php#605`
再追進去就可以看到他實作了大寫轉小寫的部分：

```php
private function filterHost($host): string
{
    if (!is_string($host)) {
        throw new \InvalidArgumentException('Host must be a string');
    }

    return \strtr($host, 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz');
}
```

`vendor\guzzlehttp\psr7\src\Uri.php#573`
而除了 URI 的 Host 部分，可以看到 Scheme 的部分也有作一樣的事情，所以這部分也需要注意：

```php
private function filterScheme($scheme): string
{
    if (!is_string($scheme)) {
        throw new \InvalidArgumentException('Scheme must be a string');
    }

    return \strtr($scheme, 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz');
}
```

## Summary

所以看起來問題是存在的，預計會找個有空的時間把這東西修好發個 PR 到 Laravel 那邊，目前初步想法是看能不能將 mock 的網址中的 Host 及 Scheme 部分一樣實作大寫轉小寫，或與官方討論一下看能怎麼處理。

總之這邊就簡單筆記一下，希望可以幫助一樣踩到這個坑的人，也希望我有時間去研究一下怎麼補起來 XD

這裡可以看到，「inconsistent」常常是 bug 或是漏洞的成因，因此我們自己在開發或是 review 時，也可以重點朝這方面去注意，可以避免一些淺在的問題。
