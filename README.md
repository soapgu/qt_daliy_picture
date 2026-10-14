# Qt 每日一图

## 1.0 功能范围

1. 获取当日每日一图，显示图片、标题（`title`）和版权信息（`copyright`）。
2. 提供“刷新”按钮，点击后重新请求当日图片信息并下载图片，更新图片、标题和版权信息。

## 实现约定

1. 采用最简 MVVM：单个 QML 文件作为 View，配合一个 ViewModel，通过属性和方法直接绑定，不引入 MVVM 框架或额外框架代码。
2. 使用 Qt 原生日志接口 `qCDebug`、`qCInfo` 打印日志，由 `spdlog` 统一输出，日志输出级别设为 `Debug`。
3. HTTP 组件待调研，暂不确定选型。

开发环境与 `caliburn-micro-qt` 对齐：

| 目标平台 | Qt 版本与 Kit | Qt Creator |
| --- | --- | --- |
| macOS arm64 | Qt 6.8.3，macOS Kit | 20.0.1 |
| 银河麒麟 V10 SP1 2403 x86_64 | Qt 6.8.3，gcc_64 Kit | 14.0.2 |

以上为本项目的开发目标，尚未表示本项目已在两个平台通过验收。

## Bing 每日一图 API

通过接口获取每日图片的标题、日期和图片地址，再根据返回的地址下载图片。

### 请求方式

`GET`

[完整请求地址](https://cn.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1&mkt=zh-CN)：

```text
https://cn.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1&mkt=zh-CN
```

```bash
curl --location --fail --show-error \
  'https://cn.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1&mkt=zh-CN'
```

### 请求参数

| 参数 | 本项目取值 | 说明 |
| --- | --- | --- |
| `format` | `js` | 返回 JSON 格式的数据。 |
| `idx` | `0` | 图片日期的偏移量：`0` 表示今天，`1` 表示昨天，`2` 表示前天。 |
| `n` | `1` | 请求返回的图片数量，此处只获取一张。 |
| `mkt` | `zh-CN` | 指定中国大陆中文区域。 |

获取昨天的图片时，将 `idx` 改为 `1`：

```text
https://cn.bing.com/HPImageArchive.aspx?format=js&idx=1&n=1&mkt=zh-CN
```

### 响应示例

以下为省略其他字段后的 JSON 示例，图片内容和日期会随接口更新变化。

```json
{
  "images": [
    {
      "startdate": "20221019",
      "fullstartdate": "202210191600",
      "enddate": "20221020",
      "url": "/th?id=OHR.SlothDay_ZH-CN4945330735_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp",
      "urlbase": "/th?id=OHR.SlothDay_ZH-CN4945330735",
      "copyright": "塔拉曼卡旧港的霍氏树懒母子，哥斯达黎加 (© Suzi Eszterhas/Minden Pictures)",
      "copyrightlink": "https://www.bing.com/search?q=%E9%9C%8D%E6%B0%8F%E6%A0%91%E6%87%92&FORM=hpcapt&mkt=zh-cn",
      "title": "到哪都挂着"
    }
  ]
}
```

### 响应字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `images` | 数组 | 图片信息列表，获取一张图片时读取 `images[0]`。 |
| `images[].startdate` | 字符串 | 图片开始日期，格式为 `yyyyMMdd`。 |
| `images[].fullstartdate` | 字符串 | 包含时分的开始时间，格式为 `yyyyMMddHHmm`。 |
| `images[].enddate` | 字符串 | 图片结束日期，格式为 `yyyyMMdd`。 |
| `images[].title` | 字符串 | 图片标题。 |
| `images[].url` | 字符串 | 图片的相对地址，包含尺寸及其他查询参数。 |
| `images[].urlbase` | 字符串 | 图片的基础相对地址，不包含尺寸后缀及其他查询参数。 |
| `images[].copyright` | 字符串 | 图片描述及版权署名信息。 |
| `images[].copyrightlink` | 字符串 | 图片相关的 Bing 搜索页面完整地址。 |

### 获取图片

读取 `images[0].url`，在前面拼接 `https://cn.bing.com`，得到图片的完整地址：

```text
图片完整地址 = https://cn.bing.com + images[0].url
```

以上述响应为例，拼接结果为：

```text
https://cn.bing.com/th?id=OHR.SlothDay_ZH-CN4945330735_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp
```

对完整地址发送 `GET` 请求即可获取图片二进制数据。下面的命令会将示例图片保存为 `daily.jpg`：

```bash
curl --location --fail --show-error \
  'https://cn.bing.com/th?id=OHR.SlothDay_ZH-CN4945330735_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp' \
  --output daily.jpg
```
