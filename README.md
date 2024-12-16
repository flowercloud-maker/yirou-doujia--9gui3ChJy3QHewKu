
转载请注明出处：


　　在 InfluxDB 中，默认的时区是 UTC（协调世界时）。所有的时间戳在数据写入时默认视为 UTC。这意味着如果没有在插入数据时指定其他时区，InfluxDB 会将所有时间数据处理为 UTC 时间。


　　最近在使用 group by(1d) 进行数据汇总查询得时候，本应是北京时间一天得数据却查出了两天的数据：




```
> SELECT sum(rx_megabits_per_second) as in_traffic, sum(tx_megabits_per_second) as out_traffic FROM monitor.autogen.vpn_traffic_5m WHERE time >= '2024-12-12T16:00:00Z' and time <= '2024-12-13T02:39:54Z'  and vpn_id='l3_3023' group by time(1d), site_id, site_name FILL(0)
name: vpn_traffic_5m
tags: site_id=dae31a81-9336-4a76-838e-878144143bcd, site_name=ZTE
time                in_traffic            out_traffic
----                ----------            -----------
1733961600000000000 0.0007709866666666677 0.011690879999999997
1734048000000000000 0.0002471466666666666 0.003948480000000001
>
```


　　**group by（1d）**会根据时间戳将每个数据点分配到它所属的那一天（基于 UTC 时间）。由于时间范围是从一天的北京0点到当天的早晨10点，因此 InfluxDB 会创建两个UTC时间组：一个是从12月12日下午4点开始到12月12日24点之前的所有数据，另一个是从12月13日0点开始到早晨的2点的数据（如果有的话）。


　　由于时间范围跨越了午夜，并且使用了 `GROUP BY time(1d)`，因此 InfluxDB 会自动根据日期边界（午夜）来分割数据。


　　而我们想要查询的时间应该是使用东八区时间(比零时区时间快了8个小时)。**在查询数据时，可以通过使用 `time` 函数将结果转换为其他时区。例如，可以使用 `tz` 选项以指定将 `SELECT` 的时间戳转换为指定的时区。 例如：**




```
SELECT * FROM measurement_name WHERE time >= '2024-12-01T00:00:00Z' AND time <= '2024-12-31T23:59:59Z' tz('Asia/Shanghai')
```


* 在这个示例中，查询结果的时间戳将转换为 `Asia/Shanghai` 时区。


　　在实际应用的过程中，如下：




```
> SELECT sum(rx_megabits_per_second) as in_traffic, sum(tx_megabits_per_second) as out_traffic FROM monitor.autogen.vpn_traffic_5m WHERE time >= '2024-12-12T16:00:00Z' and time <= '2024-12-13T02:39:54Z'  and vpn_id='l3_3023' group by time(1d), site_id, site_name FILL(0) tz('Asia/Shanghai')
name: vpn_traffic_5m
tags: site_id=dae31a81-9336-4a76-838e-878144143bcd, site_name=ZTE
time                in_traffic           out_traffic
----                ----------           -----------
1734019200000000000 0.001018133333333335 0.01563936
>
```


 


 本博客参考[楚门加速器](https://shexiangshi.org)。转载请注明出处！
