# Python 装饰器与重试逻辑复习笔记

这份笔记整理的是当前代码里 `retry_request` 装饰器的理解重点，方便以后复习。

## 1. 装饰器整体结构

当前代码里的装饰器是一个“带参数的装饰器”：

```python
def retry_request(max_retries=3):
    def decorator(func):
        def wrapper(self, *args, **kwargs):
            ...
        return wrapper
    return decorator
```

它有三层：

1. `retry_request(max_retries=3)`：接收装饰器自己的参数。
2. `decorator(func)`：接收被装饰的原始函数，比如 `get_json` 或 `post_json`。
3. `wrapper(self, *args, **kwargs)`：真正替代原函数执行的包装函数。

使用方式：

```python
@retry_request(max_retries=3)
def get_json(self, url, params=None, extra_headers=None, attempt=0):
    ...
```

等价于：

```python
get_json = retry_request(max_retries=3)(get_json)
```

执行过程可以理解为：

```text
retry_request(max_retries=3) 返回 decorator
decorator(get_json) 返回 wrapper
最终 get_json 被替换成 wrapper
```

所以以后调用：

```python
self.get_json(...)
```

实际进入的是：

```python
wrapper(...)
```

## 2. 为什么最后是 wrapper，而不是 decorator

因为这是两步完成的：

```python
decorator = retry_request(max_retries=3)
get_json = decorator(get_json)
```

第一步返回 `decorator`。

第二步执行 `decorator(get_json)`，返回 `wrapper`。

所以最终：

```python
get_json = wrapper
```

`decorator` 只在函数定义阶段用来“接收原始函数”，真正运行时执行的是 `wrapper`。

## 3. self 是怎么传进去的

类方法正常调用：

```python
client.post_json(url, payload)
```

Python 实际会自动转换成：

```python
AliHealthApiClient.post_json(client, url, payload)
```

也就是说：

```python
self = client
```

加了装饰器之后，`post_json` 被替换成了 `wrapper`：

```python
post_json = wrapper
```

所以调用：

```python
client.post_json(url, payload)
```

实际进入：

```python
wrapper(client, url, payload)
```

因此 `wrapper` 需要这样定义：

```python
def wrapper(self, *args, **kwargs):
```

这里的 `self` 就是当前的 `client` 对象。

然后在 `wrapper` 里面调用原始函数：

```python
return func(self, *args, attempt=attempt, **kwargs)
```

这里的 `func` 是原始的 `post_json` 或 `get_json`。

所以完整链路是：

```text
client.post_json(url, payload)
  -> wrapper(client, url, payload)
  -> func(client, url, payload, attempt=attempt)
  -> 原始 post_json(self=client, url=url, payload=payload, attempt=attempt)
```

## 4. 参数打包和拆包

装饰器里这一句：

```python
def wrapper(self, *args, **kwargs):
```

是在“打包”参数。

比如调用：

```python
self.post_json(url, payload, extra_headers={"x-from": "brand-star-atlas"})
```

进入 `wrapper` 后：

```python
self = 当前 client 对象
args = (url, payload)
kwargs = {
    "extra_headers": {"x-from": "brand-star-atlas"}
}
```

然后这一句：

```python
func(self, *args, attempt=attempt, **kwargs)
```

是在“拆包”。

它等价于：

```python
func(
    self,
    url,
    payload,
    attempt=attempt,
    extra_headers={"x-from": "brand-star-atlas"}
)
```

总结：

```text
函数定义里：*args / **kwargs 是打包
函数调用里：*args / **kwargs 是拆包
```

## 5. for 循环什么时候执行

装饰器里的核心重试循环：

```python
for attempt in range(max_retries + 1):
    if attempt > 0:
        print(f"开始第 {attempt + 1} 次请求，最多 {max_retries + 1} 次")
        self._sleep_for_retry()

    try:
        return func(self, *args, attempt=attempt, **kwargs)
    except RuntimeError as exc:
        ...
```

只要调用被装饰的函数，就会进入这个 `for` 循环。

例如这些会进入：

```python
self.get_json(...)
self.post_json(...)
```

因为它们有：

```python
@retry_request(max_retries=3)
```

这些不会直接进入：

```python
fetch_item_sale_detail(...)
fetch_paginated(...)
translate_sale_detail_to_rows(...)
```

但如果它们内部调用了 `get_json` 或 `post_json`，仍然会间接进入重试循环。

## 6. max_retries=3 实际请求几次

代码是：

```python
for attempt in range(max_retries + 1):
```

如果：

```python
max_retries = 3
```

那么：

```python
range(4)
```

会执行：

```text
attempt = 0  第 1 次请求
attempt = 1  第 2 次请求
attempt = 2  第 3 次请求
attempt = 3  第 4 次请求
```

所以它的含义是：

```text
首次请求 1 次 + 失败后最多重试 3 次
```

## 7. 什么时候继续下一轮，什么时候不继续

如果请求成功：

```python
return func(self, *args, attempt=attempt, **kwargs)
```

这里会直接 `return`，整个 `wrapper` 结束，`for` 循环不会继续。

也就是说，第一次成功只跑一次：

```text
attempt = 0
请求成功
return result
循环结束
```

如果请求失败，并且原函数抛出 `RuntimeError`：

```python
except RuntimeError as exc:
```

装饰器会捕获异常，记录错误，然后进入下一次循环。

例如第一次失败、第二次成功：

```text
attempt = 0
请求失败，抛 RuntimeError
记录错误

attempt = 1
等待 1-3 秒
请求成功
return result
循环结束
```

如果四次都失败：

```text
attempt = 0
失败，记录

attempt = 1
等待，失败，记录

attempt = 2
等待，失败，记录

attempt = 3
等待，失败，记录
达到最大重试次数
raise 抛出异常
```

## 8. 异常在哪里第一次抛出

真正创建并抛出异常的地方是：

```python
def _raise_error(self, error):
    exc = RuntimeError(error)
    exc.error_detail = error
    raise exc
```

只要代码执行到：

```python
self._raise_error(...)
```

就会抛出 `RuntimeError`。

例如：

```python
if result.get("code") != 1 or result.get("status") is not True:
    self._raise_error({
        "type": "business_error",
        ...
    })
```

这表示接口虽然返回了 JSON，但业务状态不是成功，于是主动抛异常。

## 9. 异常为什么不会立刻到外层

因为异常会先被装饰器捕获：

```python
except RuntimeError as exc:
```

装饰器捕获后会：

1. 保存最后一次异常：

```python
last_error = exc
```

2. 取出错误详情：

```python
error = getattr(exc, "error_detail", {...})
```

3. 记录错误：

```python
self._record_error(error)
```

4. 打印错误：

```python
print(f"请求异常: {error}")
```

5. 判断是否达到最大重试次数：

```python
if attempt >= max_retries:
    print("已达到最大重试次数，停止请求")
    raise
```

如果没达到最大次数，就不会抛给外层，而是继续下一轮。

## 10. 异常什么时候真正抛到外层

当最后一次也失败时：

```python
attempt = max_retries
```

条件成立：

```python
if attempt >= max_retries:
```

然后执行：

```python
raise
```

这里的 `raise` 表示把刚刚捕获的 `RuntimeError` 原样重新抛出去。

之后异常会沿着调用链向外冒：

```text
wrapper()
  -> fetch_item_sale_detail()
    -> get_product_sale_detail()
      -> xbot 调用模块
```

因为外层没有 `try/except` 吞掉异常，所以 xbot 会感知到失败。

## 11. raise last_error 是什么

装饰器最后有一行：

```python
raise last_error
```

它是兜底逻辑。

正常情况下，如果重试到最大次数仍然失败，会在这里抛出：

```python
if attempt >= max_retries:
    raise
```

所以正常不会走到最后的：

```python
raise last_error
```

它的作用是防御性兜底：万一循环结束了，但前面没有成功 `return`，也没有在最大次数处 `raise`，就把最后一次异常抛出去，避免静默失败。

更稳的写法可以是：

```python
if last_error is not None:
    raise last_error

raise RuntimeError("请求失败，但没有捕获到具体异常")
```

## 12. 当前代码的异常流总结

以 `get_product_sale_detail()` 为例：

```text
get_product_sale_detail()
  -> client.fetch_item_sale_detail()
    -> self.get_json()
      -> wrapper()
        -> 原始 get_json()
          -> 请求接口
          -> 如果失败，调用 self._raise_error()
            -> 抛 RuntimeError
        -> wrapper 捕获 RuntimeError
        -> 记录错误
        -> 没到最大次数：等待后重试
        -> 到最大次数：raise 抛给外层
```

最核心的一句话：

```text
原始 get_json/post_json 负责发现错误并抛 RuntimeError；
装饰器 wrapper 负责捕获 RuntimeError、重试、最终失败后重新抛出。
```

## 13. 可以记住的简化模型

可以把装饰器理解成一个“请求保护壳”：

```text
你调用 get_json/post_json
  -> 实际先进入 wrapper
  -> wrapper 调用真正的 get_json/post_json
  -> 成功：直接返回结果
  -> 失败：记录错误，等待，重试
  -> 重试耗尽：把异常抛给外层
```

`wrapper` 不关心具体接口参数是什么，因为它用：

```python
*args
**kwargs
```

把参数统一接住，再原样转交给原函数。



## 参考代码

```python
# 使用提醒:
# 1. xbot 包提供软件自动化、数据表格、Excel、日志、AI 等功能
# 2. package 包提供访问当前应用数据的功能，如获取元素、访问全局变量、获取资源文件等功能
# 3. 当此模块作为流程独立运行时执行 main 函数
# 4. 可视化流程中可以通过“调用模块”的指令使用此模块

import xbot
from xbot import print, sleep
from .import package
from .package import variables as glv

import json
import random
import time
import functools
import requests

# 全局变量
CLIENT = None


def retry_request(max_retries=3):
    """请求重试装饰器：只要被装饰的方法抛 RuntimeError，就等待后重试。"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(self, *args, **kwargs):
            last_error = None

            for attempt in range(max_retries + 1):
                if attempt > 0:
                    print(f"开始第 {attempt + 1} 次请求，最多 {max_retries + 1} 次")
                    self._sleep_for_retry()

                try:
                    return func(self, *args, attempt=attempt, **kwargs)
                except RuntimeError as exc:
                    last_error = exc

                    error = getattr(exc, "error_detail", {
                        "type": "runtime_error",
                        "message": str(exc),
                        "attempt": attempt + 1,
                    })

                    self._record_error(error)
                    print(f"请求异常: {error}")

                    if attempt >= max_retries:
                        print("已达到最大重试次数，停止请求")
                        raise

            raise last_error

        return wrapper

    return decorator


class AliHealthApiClient:
    """阿里健康数据接口客户端。

    负责：
    1. 统一管理 cookies、x-xsrf-token、session
    2. 统一构造 headers
    3. 统一处理 GET/POST JSON 请求
    4. 统一处理失败重试、错误记录、请求间隔
    """

    def __init__(
        self,
        cookies,
        x_xsrf_token,
        timeout=20,
        no_cache=True,
    ):
        """初始化客户端。

        cookies:
            支持浏览器导出的 cookie list，也支持已经整理好的 dict。
        x_xsrf_token:
            请求头里的 x-xsrf-token。
        """
        self.cookies = self._cookies_to_dict(cookies)
        self.x_xsrf_token = x_xsrf_token
        self.timeout = timeout
        self.no_cache = no_cache
        self.errors = []
        self.session = requests.Session()

    def _cookies_to_dict(self, cookies):
        """将浏览器 cookie 列表转换成 requests 可用的 dict。"""
        if isinstance(cookies, dict):
            return cookies

        if not isinstance(cookies, list):
            raise TypeError("cookies 必须是 list 或 dict")

        result = {}

        for item in cookies:
            if "name" not in item or "value" not in item:
                raise KeyError(f"cookie 缺少 name 或 value: {item}")

            result[item["name"]] = item["value"]

        return result

    def _normalize_date(self, date_text):
        """将 2026-05-11 转成 20260511；如果已是 20260511，则保持不变。"""
        if not date_text:
            raise ValueError("日期不能为空")
        return str(date_text).replace("-", "")

    def _sleep_between_requests(self):
        """每次请求完成后等待 1-3 秒，降低请求频率。"""
        wait_seconds = random.uniform(1, 3)
        print(f"请求完成，等待 {wait_seconds:.2f} 秒")
        time.sleep(wait_seconds)

    def _sleep_for_retry(self):
        """请求失败后等待 1-3 秒再重试。"""
        wait_seconds = random.uniform(1, 3)
        print(f"请求失败，等待 {wait_seconds:.2f} 秒后重试")
        time.sleep(wait_seconds)

    def _record_error(self, error):
        """记录请求失败信息，方便外层排查是否少数据。"""
        self.errors.append(error)

    def _raise_error(self, error):
        """抛出带 error_detail 的 RuntimeError，交给重试装饰器处理。"""
        exc = RuntimeError(error)
        exc.error_detail = error
        raise exc

    def _build_headers(self, extra_headers=None):
        """构造通用 headers。

        注意：
        - x-from 不是通用 header，只在具体接口方法里通过 extra_headers 传入。
        """
        headers = {
            "accept": "application/json",
            "accept-language": "zh,en-GB;q=0.9,en-US;q=0.8,en;q=0.7",
            "content-type": "application/json; charset=UTF-8",
            "origin": "https://brand.alihealth.cn",
            "referer": "https://brand.alihealth.cn/",
            "user-agent": (
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/147.0.0.0 Safari/537.36"
            ),
            "x-xsrf-token": self.x_xsrf_token,
        }

        if self.no_cache:
            headers.update({
                "cache-control": "no-cache",
                "pragma": "no-cache",
            })

        if extra_headers:
            headers.update(extra_headers)

        return headers

    @retry_request(max_retries=3)
    def get_json(self, url, params=None, extra_headers=None, attempt=0):
        """发送 GET 请求，并校验接口返回 code/status。"""
        print(f"正在请求 URL: {url}")
        print(f"请求参数: {params}")

        try:
            response = self.session.get(
                url,
                headers=self._build_headers(extra_headers),
                cookies=self.cookies,
                params=params,
                timeout=self.timeout,
            )
            response.raise_for_status()
        except requests.RequestException as exc:
            self._raise_error({
                "type": "request_error",
                "method": "GET",
                "url": url,
                "params": params,
                "attempt": attempt + 1,
                "message": str(exc),
            })

        try:
            result = response.json()
        except ValueError:
            self._raise_error({
                "type": "json_error",
                "method": "GET",
                "url": url,
                "params": params,
                "attempt": attempt + 1,
                "response_text": response.text,
            })

        if result.get("code") != 1 or result.get("status") is not True:
            self._raise_error({
                "type": "business_error",
                "method": "GET",
                "url": url,
                "params": params,
                "attempt": attempt + 1,
                "result": result,
            })

        self._sleep_between_requests()
        return result


    @retry_request(max_retries=3)
    def post_json(self, url, payload, extra_headers=None, attempt=0):
        """发送 POST JSON 请求，并校验接口返回 code/status。"""
        print(f"正在请求 URL: {url}")
        print(f"请求参数: {payload}")

        try:
            response = self.session.post(
                url,
                headers=self._build_headers(extra_headers),
                cookies=self.cookies,
                data=json.dumps(payload, separators=(",", ":"), ensure_ascii=False),
                timeout=self.timeout,
            )
            response.raise_for_status()
        except requests.RequestException as exc:
            self._raise_error({
                "type": "request_error",
                "method": "POST",
                "url": url,
                "payload": payload,
                "attempt": attempt + 1,
                "message": str(exc),
            })

        try:
            result = response.json()
        except ValueError:
            self._raise_error({
                "type": "json_error",
                "method": "POST",
                "url": url,
                "payload": payload,
                "attempt": attempt + 1,
                "response_text": response.text,
            })

        if result.get("code") != 1 or result.get("status") is not True:
            self._raise_error({
                "type": "business_error",
                "method": "POST",
                "url": url,
                "payload": payload,
                "attempt": attempt + 1,
                "result": result,
            })

        self._sleep_between_requests()
        return result


    def fetch_paginated(
        self,
        url,
        payload_builder,
        extra_headers=None,
        page_size=10,
        start_page=1,
        data_key="data",
        total_page_key="totalPage",
    ):
        """通用分页请求。

        payload_builder:
            接收 page_num、page_size，返回当前页请求 payload。
        """
        all_rows = []
        page_num = start_page
        total_page = None
        last_result = None

        print(f"开始分页请求: {url}")

        while total_page is None or page_num <= total_page:
            payload = payload_builder(page_num, page_size)

            print(f"当前页数: {page_num}")
            print(f"每页条数: {page_size}")

            result = self.post_json(url, payload, extra_headers=extra_headers)

            last_result = result
            rows = result.get(data_key) or []
            all_rows.extend(rows)

            total_page = int(result.get(total_page_key) or 1)
            total_count = result.get("totalCount", len(all_rows))

            print(f"当前页返回条数: {len(rows)}")
            print(f"总页数: {total_page}")
            print(f"接口总条数: {total_count}")
            print(f"已累计处理条数: {len(all_rows)}")

            page_num += 1

        print(f"分页请求完成: {url}")
        print(f"最终处理条数: {len(all_rows)}")

        return {
            "data": all_rows,
            "totalPage": total_page,
            "totalCount": last_result.get("totalCount") if last_result else len(all_rows),
            "pageSize": page_size,
        }

    def fetch_itemanalysis_detail(self, start_time, end_time, page_size=10):
        """获取商品分析列表，用于拿 itemId。"""
        url = "https://osweb-b2c-alihealth.tmall.com/gw/brand/datav/itemanalysis/detail"

        start_time = self._normalize_date(start_time)
        end_time = self._normalize_date(end_time)

        def payload_builder(page_num, page_size):
            return {
                "pageNum": page_num,
                "pageSize": page_size,
                "searchKey": "",
                "sellerIds": [],
                "brandIds": [],
                "cycleType": "day",
                "startTime": start_time,
                "endTime": end_time,
                "orderBy": "visitorNum",
                "order": 0,
            }

        return self.fetch_paginated(
            url=url,
            payload_builder=payload_builder,
            extra_headers={
                "x-from": "brand-star-atlas",
            },
            page_size=page_size,
        )

    def fetch_item_sale_detail(
        self,
        item_id,
        start_time,
        end_time,
        cycle_type="day",
    ):
        """获取单个商品的 360 销售详情。"""
        url = "https://osweb-b2c-alihealth.tmall.com/gw/datav/item360/getSaleDetail"

        req = {
            "itemId": str(item_id),
            "startTime": self._normalize_date(start_time),
            "endTime": self._normalize_date(end_time),
            "cycleType": cycle_type,
        }

        params = {
            "req": json.dumps(req, separators=(",", ":"), ensure_ascii=False)
        }

        result = self.get_json(
            url=url,
            params=params,
            extra_headers={
                "accept": "application/json, text/plain, */*",
            },
        )

        return result.get("data") or {}

    def fetch_item_sale_details(self, items, start_time, end_time):
        """批量获取商品 360 销售详情。"""
        details = []
        total = len(items)

        print(f"开始批量获取商品详情，总条数: {total}")

        for index, item in enumerate(items, start=1):
            item_id = item.get("item_id")
            item_title = item.get("item_title", "")

            print(f"正在处理商品详情: {index}/{total}")
            print(f"item_id: {item_id}")
            print(f"item_title: {item_title}")

            detail = self.fetch_item_sale_detail(
                item_id=item_id,
                start_time=start_time,
                end_time=end_time,
            )

            details.append({
                "item_id": item_id,
                "item_title": item_title,
                "detail": detail,
                "sku": detail.get("sku", []),
            })

        print(f"商品详情处理完成，总条数: {len(details)}")
        return details



def get_client(cookies, x_xsrf_token):
    """获取全局复用的客户端，避免外层循环时反复创建对象。"""
    global CLIENT

    if CLIENT is None:
        CLIENT = AliHealthApiClient(
            cookies=cookies,
            x_xsrf_token=x_xsrf_token,
        )
        print("已创建 AliHealthApiClient")

    return CLIENT


def get_product_list(
    cookie_list=None, 
    x_xsrf_token=None, 
    start_time=None, 
    end_time=None, 
    **kwargs
):
    """只获取商品列表，返回 item_id / item_title，详情由外层循环处理。"""
    if kwargs:
        print(f"收到未定义参数: {kwargs}")

    if "" in kwargs:
        raise ValueError(f"调用模块里存在空参数名，请删除该参数。空参数值: {kwargs.get('')}")


    if not cookie_list:
        raise ValueError("缺少参数: cookies")

    if not x_xsrf_token:
        raise ValueError("缺少参数: x_xsrf_token")

    if not start_time:
        raise ValueError("缺少参数: start_time")

    if not end_time:
        raise ValueError("缺少参数: end_time")

    client = get_client(
        cookies=cookie_list,
        x_xsrf_token=x_xsrf_token,
    )

    list_result = client.fetch_itemanalysis_detail(
        start_time=start_time,
        end_time=end_time,
        page_size=10,
    )

    items = [
        {
            "item_id": item.get("itemId"),
            "item_title": item.get("itemTitle", ""),
        }
        for item in list_result.get("data", [])
        if item.get("itemId")
    ]

    total_count = list_result.get("totalCount")
    actual_count = len(items)

    print(f"商品列表获取完成，商品条数: {actual_count}")

    if total_count is not None and actual_count != total_count:
        print("=" * 60)
        print("严重警告：商品列表条数与接口总条数不一致，可能存在漏采！")
        print(f"接口 totalCount: {total_count}")
        print(f"实际提取条数: {actual_count}")
        print("=" * 60)

    return items


def get_product_sale_detail(
    cookie_list=None,
    x_xsrf_token=None,
    item_id=None,
    item_title=None,
    start_time=None,
    end_time=None,
    **kwargs
):
    """获取单个商品 360 销售详情，返回 item_id / item_title / detail / sku。"""
    # print("get_product_sale_detail 已进入")
    # print(f"cookie_list 是否存在: {bool(cookie_list)}")
    # print(f"x_xsrf_token 是否存在: {bool(x_xsrf_token)}")
    # print(f"item_id: {item_id}")
    # print(f"item_title: {item_title}")
    # print(f"start_time: {start_time}")
    # print(f"end_time: {end_time}")
    # print(f"额外参数 kwargs: {kwargs}")

    if kwargs:
        print(f"收到未定义参数: {kwargs}")

    if "" in kwargs:
        raise ValueError(f"调用模块里存在空参数名，请删除该参数。空参数值: {kwargs.get('')}")

    if not cookie_list:
        raise ValueError("缺少参数: cookie_list")

    if not x_xsrf_token:
        raise ValueError("缺少参数: x_xsrf_token")

    if not item_id:
        raise ValueError("缺少参数: item_id")

    if not item_title:
        raise ValueError("缺少参数: item_title")

    if not start_time:
        raise ValueError("缺少参数: start_time")

    if not end_time:
        raise ValueError("缺少参数: end_time")

    client = get_client(
        cookies=cookie_list,
        x_xsrf_token=x_xsrf_token,
    )

    detail = client.fetch_item_sale_detail(
        item_id=item_id,
        start_time=start_time,
        end_time=end_time,
    )

    return translate_sale_detail_to_rows({
        "item_id": str(item_id),
        "item_title": item_title,
        "sku": detail.get("sku", []),
    })



def translate_sale_detail_to_rows(sale_detail):
    """将商品销售详情转成中文字段行。

    输出字段：
    商品id / 商品名称 / sku名称 / skuid / 支付金额 / 支付金额占比 /
    支付件数 / 支付用户数 / 支付新客数 / 支付老客数
    """
    item_id = sale_detail.get("item_id")
    item_title = sale_detail.get("item_title")
    sku_list = sale_detail.get("sku") or []

    rows = []

    for sku in sku_list:
        rows.append({
            "商品id": item_id,
            "商品名称": item_title,
            "sku名称": sku.get("name"),
            "skuid": sku.get("id"),
            "支付金额": sku.get("payAmount"),
            "支付金额占比": sku.get("payAmtRatio"),
            "支付件数": sku.get("payNum"),
            "支付用户数": sku.get("payBuyerNum"),
            "支付新客数": sku.get("payNewUserNum"),
            "支付老客数": sku.get("payOldUserNum"),
        })

    return rows


def main(args):
    pass
```

