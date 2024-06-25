# 桶生命周期配置指南
在桶上开启`对象`声明周期配置，以设置在指定天数或指定日期后自动删除对象。

## 1.先决条件
- 安装Minio
- 安装客户端工具`mc`

## 2.开启桶生命周期配置
- 创建桶生命周期配置，使前缀`old/`下的对象于`2020-01-01T00:00:00.000Z`过期，并使前缀`temp/`下的对象在7天后过期。
- 使用`mc`工具开启桶生命周期配置。
```shell
$ mc ilm import play/testbucket <<EOF
{
    "Rules": [
        {
            "Expiration": {
                "Date": "2020-01-01T00:00:00.000Z"
            },
            "ID": "OldPictures",
            "Filter": {
                "Prefix": "old/"
            },
            "Status": "Enabled"
        },
        {
            "Expiration": {
                "Days": 7
            },
            "ID": "TempUploads",
            "Filter": {
                "Prefix": "temp/"
            },
            "Status": "Enabled"
        }
    ]
}
EOF

Lifecycle configuration imported successfully to `play/testbucket`.
```
- 查看ilm配置
```shell
$ mc ilm ls play/testbucket
     ID     |  Prefix  |  Enabled   | Expiry |  Date/Days   |  Transition  |    Date/Days     |  Storage-Class   |       Tags
------------|----------|------------|--------|--------------|--------------|------------------|------------------|------------------
OldPictures |   old/   |    ✓       |  ✓     |  1 Jan 2020  |     ✗        |                  |                  |
------------|----------|------------|--------|--------------|--------------|------------------|------------------|------------------
TempUploads |  temp/   |    ✓       |  ✓     |   7 day(s)   |     ✗        |                  |                  |
------------|----------|------------|--------|--------------|--------------|------------------|------------------|------------------
```

## 3.开启ILM版本控制功能
该功能仅对开启了版本控制功能的桶有效。

### 3.1 自动删除非当前对象版本
`非当前对象版本`指对象版本不是给定对象的最新版本。当版本早于给定天数时，可以设置自动删除非当前版本。
eg：扫描`user-uploads/`前缀的对象并删除一年以上的版本。
```shell
{
    "Rules": [
        {
            "ID": "Removing all old versions",
            "Filter": {
                "Prefix": "users-uploads/"
            },
            "NoncurrentVersionExpiration": {
                "NoncurrentDays": 365
            },
            "Status": "Enabled"
        }
    ]
}
```
上面的JSON规则和下面的命名作用一致：
```shell
mc ilm rule add --noncurrent-expire-days 365 --prefix "user-uploads/" myminio/mydata
```

### 3.2 自动删除非当前版本，仅保留最新的N个版本
可以使用`NewerNoncurrentVersions`配置自动删除旧的非当前版本，仅保留最新的`N`个版本。
eg：扫描`user-uploads/`前缀的对象并删除30天以上的版本，同时保留最新的5个版本。
```shell
{
    "Rules": [
        {
            "ID": "Keep only most recent 5 noncurrent versions",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "users-uploads/"
            },
            "NoncurrentVersionExpiration": {
                "NewerNoncurrentVersions": 5,
                "NoncurrentDays": 30
            }
        }
    ]
}
```

## 4.开启ILM转换功能
MinIO支持分层到公共云提供商以及其他MinIO集群。通过在桶生命周期配置中设置`转换规则`将旧对象转换到不同的集群或公共云。
此功能使应用程序能够通过将不常访问的数据移动到更便宜的存储来优化存储成本，而不会影响数据的可访问性。

要将存储桶中的对象转换到不同集群上的目标存储桶，应用程序需要在设置ILM生命周期规则时指定MinIO上定义的转换层而不是存储类。
```shell
mc admin tier add azure source AZURETIER --endpoint https://blob.core.windows.net --access-key AZURE_ACCOUNT_NAME --secret-key AZURE_ACCOUNT_KEY  --bucket azurebucket --prefix testprefix1/
mc ilm add --expiry-days 365 --transition-days 45 --storage-class "AZURETIER" myminio/srcbucket
```