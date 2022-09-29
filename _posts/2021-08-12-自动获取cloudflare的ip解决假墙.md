---
layout:     post
title:      自动获取cloudflare的ip解决假墙
subtitle:   cloudflare自动换IP脚本
date:       2021-08-12
author:     果果
header-img: img/post-bg-mma-1.jpg
catalog: true
tags:
    - 资源
---

    这个是php版的代码
    要先准备一台一个站群服务器，建议一整个c段253IP以上的服务器
    原理是，假墙封ip几个小时，每1分钟换一次解析，轮流使用250多个ip使其超过他封禁时间
    用linux定时任务，每一分钟运行此一次

```php
$file = './tqdata.txt'; // 同目录下新建tqdata.txt文件，里面填个数字 比如 1，用来记录当前使用的
$txt = file_get_contents($file);
$ips = [];
for ($i = 1; $i <= 254; $i++) {
    $ips[$i] = '45.45.45.' . $i;  // 自己的ip
}

$ip = $ips[$txt];
// 进cf解析页f12,找到那条记录，提交，得到下面的提交地址 xxxx是需要替换成自己的
$www = 'https://api.cloudflare.com/client/v4/zones/xxxx/dns_records/xxxx';

$datawww = array(
    "type" => "A",
    "name" => "www.domain.com",
    "content" => $ip,
    "ttl" => 120,
    "proxied" => false
);

curl_get($www, $datawww);

function curl_get($url, $data)
{
    $headers = array(
        'X-Auth-Email:自己cf的账号',
        'X-Auth-Key:自己cf的key',
        'Content-Type:application/json'
    );
    $data = json_encode($data);
    $ch = curl_init(); //初始化CURL句柄
    curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
    curl_setopt($ch, CURLOPT_URL, $url); //设置请求的URL
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); //设为TRUE把curl_exec()结果转化为字串，而不是直接输出
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "PUT"); //设置请求方式
    curl_setopt($ch, CURLOPT_POSTFIELDS, $data);//设置提交的字符串
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); // 信任任何证书
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2); // 检查证书中是否设置域名
    $output = curl_exec($ch);
    curl_close($ch);
    return json_decode($output, true);
}
```