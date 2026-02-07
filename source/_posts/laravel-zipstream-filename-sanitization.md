---
title: Laravel zipstream filename sanitization
date: 2022-06-04 16:47:59
tags:
    - laravel
    - laravel zipstream
---

## Introduction

When I develop one of my cases, there is a requirement to generate a zip file. So I find a package [`laravel-zipstream`](https://github.com/stechstudio/laravel-zipstream) to do it.

<!-- more -->

## Problem

Everything is fine until my customer told me files should be placed in a specific path which contains chinese characters. (´;ω;`)

At begining, I just change the filename like:

```php
$zip = Zip::create('user_data.zip');
$zip->add($content, "中文資料夾1/測試資料.pdf");
```

But when I download this zip, I get something like:

![](https://i.imgur.com/9R2AKAA.png)

Where is my filename and folder name :(

## Identify Problem

At begining, I thought it's some encoding problem. But I tried to change encoding and still got the same result.

So I went to looked up the source code, and found:

<https://github.com/stechstudio/laravel-zipstream/blob/master/src/Models/File.php>

```php=105
public function getZipPath(): string
{
    $path = ltrim(preg_replace('|/{2,}|', '/', $this->zipPath), '/');

    return config('zipstream.file.sanitize')
        ? Str::ascii($path)
        : $path;
}
```

When `config('zipstream.file.sanitize')` is `true`, it will try to translate filename to ascii by Laravel's Helper function `Str::ascii()`(more information in [official doc.](https://laravel.com/docs/9.x/helpers)).

## Solve

So I looked up the package's `config.php`

```php=12
    // Default options for files added
    'file'    => [
        'method' => env('ZIPSTREAM_FILE_METHOD', 'store'),

        'deflate' => env('ZIPSTREAM_FILE_DEFLATE'),

        'sanitize' => env('ZIPSTREAM_FILE_SANITIZE', true)
    ],
```

Thus, we can just add below line in our `.env` to use non-ascii characters in filename!

```env
ZIPSTREAM_FILE_SANITIZE=true
```

## Additional information

Because I don't see any description in `README` about this feature, so I also open a PR in GitHub to add some description about it.

If you're interested in it, you can find it [here](https://github.com/stechstudio/laravel-zipstream/pull/74).

## Reference

-   <https://github.com/stechstudio/laravel-zipstream>
-   <https://laravel.com/docs/9.x>
