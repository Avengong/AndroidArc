场景：

1. 上报每次启动数据到数据库： 设备号
2. 上报每次卡顿信息到数据库： 设备号

step1： 从卡顿信息表查出，每一天同一个设备号的总次数 选择其中>=3的数据。 得到新的数据： 时间、设备、总数量count

```

slect t, count as "卡顿数"，
jank/total as "卡顿率"
total as "总卡顿数"
(
select tt as t, count(cout) as jank from (
select  $timeSeries as tt, device_id, count() as cout,from apm_lag_distribute 
where app_ver in 4.0.9 and platform in android 
group by tt,device_id having cout()>=3 )aa group by t
) ods 
join left 



```

step2：

从启动表中，查询没听去重后的设备总数。

step3： 总的卡顿数/ 设备数。

```
SELECT  
login.t,
jank / login.total as `卡顿率`, 
jank as `卡顿设备`,
login.total as `活跃设备` 
FROM 
        (
        select tt as t, count(cout) as jank from 
                (
                select $timeFilter as tt,device_id, count() as cout from technical_monitor.cl_apm_lag_distributed 
                WHERE $timeFilter and app_id in ('$product') and platform in ($platform) and app_ver in ($version)
                GROUP BY device_id,$timeFilter
                having count()  >= 3
                )aa 
        GROUP BY tt 
        )ods 
left join 
        (
        select $timeFilter as t,count(DISTINCT device_id) as total 
                from 
                (
                select create_at as create_at, device_id 
                from technical_monitor.zt_apm_init_distributed where $timeFilter and platform_key_str_0 in ('$product') and platform in ($platform) and app_ver in ($version)
                ) 
        group by $timeFilter
        )login 
on login.t = ods.t  
ORDER BY login.t
```

也可以改为：

```
SELECT  
login.t,
jank / login.total as `卡顿率`, 
jank as `卡顿设备`,
login.total as `活跃设备` 
FROM 
        (
        select tt as t, countIf(cout>=3) as jank from 
                (
                select $timeFilter as tt,device_id, count() as cout from technical_monitor.cl_apm_lag_distributed 
                WHERE $timeFilter and app_id in ('$product') and platform in ($platform) and app_ver in ($version)
                GROUP BY device_id,$timeFilter
                )aa 
        GROUP BY tt 
        )ods 
left join 
        (
        select $timeFilter as t,count(DISTINCT device_id) as total 
                from 
                (
                select create_at as create_at, device_id 
                from technical_monitor.zt_apm_init_distributed where $timeFilter and platform_key_str_0 in ('$product') and platform in ($platform) and app_ver in ($version)
                ) 
        group by $timeFilter
        )login 
on login.t = ods.t  
ORDER BY login.t

```












