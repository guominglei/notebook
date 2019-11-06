# prometheus 监控系统

## 数据存储到远端 influxdb中
    https://docs.influxdata.com/influxdb/v1.7/supported_protocols/prometheus

    配置文件里添加规则
    remote_write:
      - url: "http://xxxx:8086/api/v1/prom/write?db=prometheus"
    # remote_read:
    #   - url: "http://xxxx:8086/api/v1/prom/read?db=prometheus"

## Python client 
    github: https://github.com/prometheus/client_python
    pip: pip install pip install prometheus_client

## 统计类型
### Histogram 直方图
    base_name = api_time
    bucket 阈值默认单位是秒。
    每个统计项数据：
    {base_name}_bucket 每个bucket 一条。共有N条。
    {base_name}_count 一条
    {base_name}_sum  一条
#### 查询P99耗时
    histogram_quantile(0.99, api_time_bucket{method_name="QA.view.api.v8.dialog.dialog_comment"})
#### 查询QPS
    查询1分钟内的请求量
    rate(api_time_count{method_name="QA.view.api.v8.dialog.dialog_comment"}[1m])

#### 统计机器QPS
sum by (container_name)(rate(api_time_count[1m]))
#### 统计接口QPS
sum(rate(api_time_count{method_name="QA.view.api.v8.dialog.dialog_comment"}[1m]))
#### 统计接口、机器、QPS
sum by (container_name) (rate(api_time_bucket{method_name="QA.view.api.v8.dialog.dialog_comment"}[1m]))

#### histogram 和 summary 的比较
    不同：
        histogram 的 quantile 是根据 bucket 里的数据实时计算的。
        summary的quantile是计算好的。
    相同：
        都有 {base_name}_count，{base_name}_sum


