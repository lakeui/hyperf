# 簡介

Hyperf 為您提供了分佈式系統的外部化配置支持，默認適配了:

- 由攜程開源的 [ctripcorp/apollo](https://github.com/ctripcorp/apollo)，由 [hyperf/config-apollo](https://github.com/hyperf/config-apollo) 組件提供功能支持。
- 阿里雲提供的免費配置中心服務 [應用配置管理(ACM, Application Config Manager)](https://help.aliyun.com/product/59604.html)，由 [hyperf/config-aliyun-acm](https://github.com/hyperf/config-aliyun-acm) 組件提供功能支持。
- ETCD
- Nacos
- Zookeeper

## 為什麼要使用配置中心？

隨着業務的發展，微服務架構的升級，服務的數量、應用的配置日益增多（各種微服務、各種服務器地址、各種參數），傳統的配置文件方式和數據庫的方式已經可能無法滿足開發人員對配置管理的要求，同時對於配置的管理可能還會牽涉到 ACL 權限管理、配置版本管理和回滾、格式驗證、配置灰度發佈、集羣配置隔離等問題，以及：

- 安全性：配置跟隨源代碼保存在版本管理系統中，容易造成配置泄漏
- 時效性：修改配置，需要每台服務器每個應用修改並重啟服務
- 侷限性：無法支持動態調整，例如日誌開關、功能開關等   

因此，我們可以通過一個配置中心以一種科學的管理方式來統一管理相關的配置。

## 安裝

### 配置中心統一接入層

```bash
composer require hyperf/config-center
```

### 使用 Apollo 需安裝

```bash
composer require hyperf/config-apollo
```

### 使用 Aliyun ACM 需安裝

```bash
composer require hyperf/config-aliyun-acm
```

### 使用 Etcd 需安裝

```bash
composer require hyperf/config-etcd
```

### 使用 Nacos 需安裝

```bash
composer require hyperf/config-nacos
```

### 使用 Zookeeper 需安裝

```bash
composer require hyperf/config-zookeeper
```

## 接入配置中心

### 配置文件

```php
<?php

declare(strict_types=1);

use Hyperf\ConfigCenter\Mode;

return [
    // 是否開啟配置中心
    'enable' => (bool) env('CONFIG_CENTER_ENABLE', true),
    // 使用的驅動類型，對應同級別配置 drivers 下的 key
    'driver' => env('CONFIG_CENTER_DRIVER', 'apollo'),
    // 配置中心的運行模式，多進程模型推薦使用 PROCESS 模式，單進程模型推薦使用 COROUTINE 模式
    'mode' => env('CONFIG_CENTER_MODE', Mode::PROCESS),
    'drivers' => [
        'apollo' => [
            'driver' => Hyperf\ConfigApollo\ApolloDriver::class,
            // Apollo Server
            'server' => 'http://127.0.0.1:9080',
            // 您的 AppId
            'appid' => 'test',
            // 當前應用所在的集羣
            'cluster' => 'default',
            // 當前應用需要接入的 Namespace，可配置多個
            'namespaces' => [
                'application',
            ],
            // 配置更新間隔（秒）
            'interval' => 5,
            // 嚴格模式，當為 false 時，拉取的配置值均為 string 類型，當為 true 時，拉取的配置值會轉化為原配置值的數據類型
            'strict_mode' => false,
            // 客户端IP
            'client_ip' => current(swoole_get_local_ip()),
            // 拉取配置超時時間
            'pullTimeout' => 10,
            // 拉取配置間隔
            'interval_timeout' => 1,
        ],
        'nacos' => [
            'driver' => Hyperf\ConfigNacos\NacosDriver::class,
            // 配置合併方式，支持覆蓋和合並
            'merge_mode' => Hyperf\ConfigNacos\Constants::CONFIG_MERGE_OVERWRITE,
            'interval' => 3,
            // 如果對應的映射 key 沒有設置，則使用默認的 key
            'default_key' => 'nacos_config',
            'listener_config' => [
                // dataId, group, tenant, type, content
                // 映射後的配置 KEY => Nacos 中實際的配置
                'nacos_config' => [
                    'tenant' => 'tenant', // corresponding with service.namespaceId
                    'data_id' => 'hyperf-service-config',
                    'group' => 'DEFAULT_GROUP',
                ],
                'nacos_config.data' => [
                    'data_id' => 'hyperf-service-config-yml',
                    'group' => 'DEFAULT_GROUP',
                    'type' => 'yml',
                ],
            ],
            'client' => [
                // nacos server url like https://nacos.hyperf.io, Priority is higher than host:port
                // 'uri' => '',
                'host' => '127.0.0.1',
                'port' => 8848,
                'username' => null,
                'password' => null,
                'guzzle' => [
                    'config' => null,
                ],
            ],
        ],
        'aliyun_acm' => [
            'driver' => Hyperf\ConfigAliyunAcm\AliyunAcmDriver::class,
            // 配置更新間隔（秒）
            'interval' => 5,
            // 阿里雲 ACM 斷點地址，取決於您的可用區
            'endpoint' => env('ALIYUN_ACM_ENDPOINT', 'acm.aliyun.com'),
            // 當前應用需要接入的 Namespace
            'namespace' => env('ALIYUN_ACM_NAMESPACE', ''),
            // 您的配置對應的 Data ID
            'data_id' => env('ALIYUN_ACM_DATA_ID', ''),
            // 您的配置對應的 Group
            'group' => env('ALIYUN_ACM_GROUP', 'DEFAULT_GROUP'),
            // 您的阿里雲賬號的 Access Key
            'access_key' => env('ALIYUN_ACM_AK', ''),
            // 您的阿里雲賬號的 Secret Key
            'secret_key' => env('ALIYUN_ACM_SK', ''),
            'ecs_ram_role' => env('ALIYUN_ACM_RAM_ROLE', ''),
        ],
        'etcd' => [
            'driver' => Hyperf\ConfigEtcd\EtcdDriver::class,
            'packer' => Hyperf\Utils\Packer\JsonPacker::class,
            // 需要同步的數據前綴
            'namespaces' => [
                '/application',
            ],
            // `Etcd` 與 `Config` 的映射關係。映射中不存在的 `key`，則不會被同步到 `Config` 中
            'mapping' => [
                // etcd key => config key
                '/application/test' => 'test',
            ],
            // 配置更新間隔（秒）
            'interval' => 5,
            'client' => [
                # Etcd Client
                'uri' => 'http://127.0.0.1:2379',
                'version' => 'v3beta',
                'options' => [
                    'timeout' => 10,
                ],
            ],
        ],
        'zookeeper' => [
            'driver' => Hyperf\ConfigZookeeper\ZookeeperDriver::class,
            'server' => env('ZOOKEEPER_SERVER', '127.0.0.1:2181'),
            'path' => env('ZOOKEEPER_CONFIG_PATH', '/conf'),
            'interval' => 5,
        ],
    ],
];
```

如配置文件不存在可執行 `php bin/hyperf.php vendor:publish hyperf/config-center` 命令來生成。


## 配置更新的作用範圍

在默認的功能實現下，是由一個 `ConfigFetcherProcess` 進程根據配置的 `interval` 來向 配置中心 Server 拉取對應 `namespace` 的配置，並通過 IPC 通訊將拉取到的新配置傳遞到各個 Worker 中，並更新到 `Hyperf\Contract\ConfigInterface` 對應的對象內。   
需要注意的是，更新的配置只會更新 `Config` 對象，故僅限應用層或業務層的配置，不涉及框架層的配置改動，因為框架層的配置改動需要重啟服務，如果您有這樣的需求，也可以通過自行實現 `ConfigFetcherProcess` 來達到目的。

## 注意事項

在命令行模式時，默認不會觸發事件分發，導致無法正常獲取到相關配置，可通過添加 `--enable-event-dispatcher` 參數來開啟。
