# 淘宝用户行为分析 SQL 代码注释版

本文档整理本项目用到的核心 SQL 代码。每段代码都可以单独复制到 DB Browser for SQLite 的“执行 SQL”窗口中运行。

项目数据库表：`user_behavior`

字段说明：

| 字段名 | 含义 |
|---|---|
| `user_id` | 用户 ID |
| `item_id` | 商品 ID |
| `category_id` | 商品类目 ID |
| `behavior_type` | 用户行为类型，包含 `pv`、`fav`、`cart`、`buy` |
| `behavior_ts` | Unix 时间戳 |
| `behavior_time` | 转换后的完整时间 |
| `behavior_date` | 行为日期 |
| `behavior_hour` | 行为小时 |

行为类型说明：

| 行为类型 | 含义 |
|---|---|
| `pv` | 浏览商品详情页 |
| `fav` | 收藏商品 |
| `cart` | 加入购物车 |
| `buy` | 购买商品 |

## 1. 建表与时间字段处理

```sql
-- 如果原始表已经存在，先删除，避免重复建表报错
drop table if exists user_behavior_raw;

-- 创建原始数据表，用来接收 CSV 导入的数据
create table user_behavior_raw (
    user_id integer,        -- 用户 ID
    item_id integer,        -- 商品 ID
    category_id integer,    -- 商品类目 ID
    behavior_type text,     -- 行为类型：pv、fav、cart、buy
    behavior_ts integer     -- Unix 时间戳
);

-- 如果清洗后的分析表已经存在，先删除
drop table if exists user_behavior;

-- 创建清洗后的分析表
create table user_behavior (
    user_id integer,        -- 用户 ID
    item_id integer,        -- 商品 ID
    category_id integer,    -- 商品类目 ID
    behavior_type text,     -- 行为类型
    behavior_ts integer,    -- 原始时间戳
    behavior_time text,     -- 转换后的完整时间，例如 2017-11-25 00:00:00
    behavior_date text,     -- 日期，例如 2017-11-25
    behavior_hour integer   -- 小时，例如 13
);

-- 将原始表数据写入清洗后的分析表
insert into user_behavior (
    user_id,
    item_id,
    category_id,
    behavior_type,
    behavior_ts,
    behavior_time,
    behavior_date,
    behavior_hour
)
select
    user_id,
    item_id,
    category_id,
    behavior_type,
    behavior_ts,
    datetime(behavior_ts, 'unixepoch') as behavior_time, -- 将时间戳转换为完整时间
    date(behavior_ts, 'unixepoch') as behavior_date,     -- 提取日期
    cast(strftime('%H', datetime(behavior_ts, 'unixepoch')) as integer) as behavior_hour -- 提取小时
from user_behavior_raw
where behavior_type in ('pv', 'fav', 'cart', 'buy')      -- 只保留有效行为类型
  and date(behavior_ts, 'unixepoch') between '2017-11-25' and '2017-12-03'; -- 保留分析时间范围
```

用途：完成基础建表，并把时间戳转换成可分析的日期和小时字段。

## 2. 数据总体概况

```sql
select
    count(*) as behavior_count,                    -- 总行为次数
    count(distinct user_id) as user_count,         -- 去重用户数
    count(distinct item_id) as item_count,         -- 去重商品数
    count(distinct category_id) as category_count  -- 去重类目数
from user_behavior;
```

用途：了解样本数据规模，包括行为量、用户量、商品量和类目量。

## 3. 数据时间范围

```sql
select
    min(behavior_time) as start_time, -- 最早行为时间
    max(behavior_time) as end_time,   -- 最晚行为时间
    min(behavior_date) as start_date, -- 最早行为日期
    max(behavior_date) as end_date    -- 最晚行为日期
from user_behavior;
```

用途：确认本次分析覆盖的时间范围，避免误用异常日期数据。

## 4. 用户行为类型占比

```sql
select
    behavior_type,                                      -- 行为类型
    count(*) as behavior_count,                         -- 该行为出现次数
    round(100.0 * count(*) / sum(count(*)) over (), 2) as pct -- 该行为占全部行为的百分比
from user_behavior
group by behavior_type
order by behavior_count desc;
```

用途：分析用户行为结构，判断浏览、收藏、加购、购买各自占比。

## 5. 每日 PV、UV、购买用户数和购买次数

```sql
select
    behavior_date,                                                       -- 行为日期
    sum(case when behavior_type = 'pv' then 1 else 0 end) as pv,         -- 当日浏览次数
    count(distinct user_id) as uv,                                       -- 当日活跃用户数
    count(distinct case when behavior_type = 'buy' then user_id end) as buyer_count, -- 当日购买用户数
    sum(case when behavior_type = 'buy' then 1 else 0 end) as buy_count  -- 当日购买次数
from user_behavior
group by behavior_date
order by behavior_date;
```

用途：观察每日活跃趋势，判断哪一天流量和购买行为最高。

## 6. 各小时用户行为分布

```sql
select
    behavior_hour,                                                       -- 行为发生小时，0 到 23
    sum(case when behavior_type = 'pv' then 1 else 0 end) as pv,         -- 每小时浏览次数
    sum(case when behavior_type = 'fav' then 1 else 0 end) as fav,       -- 每小时收藏次数
    sum(case when behavior_type = 'cart' then 1 else 0 end) as cart,     -- 每小时加购次数
    sum(case when behavior_type = 'buy' then 1 else 0 end) as buy        -- 每小时购买次数
from user_behavior
group by behavior_hour
order by behavior_hour;
```

用途：分析一天中哪个时间段用户更活跃，适合做活动投放和运营触达。

## 7. 行为次数口径转化漏斗

```sql
with behavior_counts as (
    select
        sum(case when behavior_type = 'pv' then 1 else 0 end) as pv_count, -- 浏览次数
        sum(case when behavior_type in ('fav', 'cart') then 1 else 0 end) as interest_count, -- 收藏和加购次数
        sum(case when behavior_type = 'buy' then 1 else 0 end) as buy_count -- 购买次数
    from user_behavior
)
select
    pv_count,                                                    -- 浏览次数
    interest_count,                                              -- 兴趣行为次数，收藏 + 加购
    buy_count,                                                   -- 购买次数
    round(1.0 * interest_count / pv_count, 4) as pv_to_interest_rate,       -- 浏览到兴趣行为转化率
    round(1.0 * buy_count / interest_count, 4) as interest_to_buy_rate,     -- 兴趣行为到购买转化率
    round(1.0 * buy_count / pv_count, 4) as pv_to_buy_rate                  -- 浏览到购买整体转化率
from behavior_counts;
```

用途：从行为次数角度分析“浏览 → 收藏/加购 → 购买”的转化过程。

## 8. 用户人数口径转化漏斗

```sql
with user_flags as (
    select
        user_id,
        max(case when behavior_type = 'pv' then 1 else 0 end) as has_pv, -- 用户是否浏览过
        max(case when behavior_type in ('fav', 'cart') then 1 else 0 end) as has_interest, -- 用户是否收藏或加购过
        max(case when behavior_type = 'buy' then 1 else 0 end) as has_buy -- 用户是否购买过
    from user_behavior
    group by user_id
)
select
    sum(has_pv) as pv_user_count,                         -- 有浏览行为的用户数
    sum(has_interest) as interest_user_count,             -- 有收藏或加购行为的用户数
    sum(has_buy) as buy_user_count,                       -- 有购买行为的用户数
    round(1.0 * sum(has_interest) / sum(has_pv), 4) as pv_user_to_interest_user_rate, -- 浏览用户到兴趣用户转化率
    round(1.0 * sum(has_buy) / sum(has_interest), 4) as interest_user_to_buy_user_rate, -- 兴趣用户到购买用户转化率
    round(1.0 * sum(has_buy) / sum(has_pv), 4) as pv_user_to_buy_user_rate -- 浏览用户到购买用户转化率
from user_flags;
```

用途：从用户人数角度分析转化漏斗。它和行为次数口径不同，一个用户只要发生过某类行为就记为 1。

## 9. 每日浏览到购买转化率

```sql
select
    behavior_date,                                                       -- 行为日期
    sum(case when behavior_type = 'pv' then 1 else 0 end) as pv,         -- 当日浏览次数
    sum(case when behavior_type = 'buy' then 1 else 0 end) as buy,       -- 当日购买次数
    round(
        1.0 * sum(case when behavior_type = 'buy' then 1 else 0 end)
        / nullif(sum(case when behavior_type = 'pv' then 1 else 0 end), 0),
        4
    ) as pv_to_buy_rate                                                  -- 当日浏览到购买转化率
from user_behavior
group by behavior_date
order by behavior_date;
```

用途：比较不同日期的购买转化率，判断高流量日期是否也带来高转化。

## 10. 购买用户复购分析

```sql
with user_buy_count as (
    select
        user_id,
        count(*) as buy_count                 -- 每个用户的购买次数
    from user_behavior
    where behavior_type = 'buy'               -- 只统计购买行为
    group by user_id
)
select
    count(*) as buyer_count,                  -- 购买用户总数
    sum(case when buy_count = 1 then 1 else 0 end) as one_time_buyer_count, -- 仅购买一次的用户数
    sum(case when buy_count >= 2 then 1 else 0 end) as repurchase_buyer_count, -- 购买两次及以上的用户数
    round(
        1.0 * sum(case when buy_count >= 2 then 1 else 0 end) / count(*),
        4
    ) as repurchase_rate                      -- 复购率
from user_buy_count;
```

用途：判断购买用户中有多少是复购用户，衡量用户粘性。

## 11. 用户留存分析

```sql
-- 如果首次活跃日期视图已经存在，先删除
drop view if exists user_first_active;

-- 创建每个用户的首次活跃日期视图
create view user_first_active as
select
    user_id,
    min(behavior_date) as first_active_date -- 用户第一次出现行为的日期
from user_behavior
group by user_id;

-- 计算次日、3 日、7 日留存
with active_by_day as (
    select distinct
        user_id,
        behavior_date                       -- 用户在哪些日期活跃过
    from user_behavior
),
retention_flags as (
    select
        f.user_id,
        f.first_active_date,
        max(case when a.behavior_date = date(f.first_active_date, '+1 day') then 1 else 0 end) as retained_1d, -- 次日是否留存
        max(case when a.behavior_date = date(f.first_active_date, '+3 day') then 1 else 0 end) as retained_3d, -- 3 日是否留存
        max(case when a.behavior_date = date(f.first_active_date, '+7 day') then 1 else 0 end) as retained_7d  -- 7 日是否留存
    from user_first_active f
    left join active_by_day a
        on f.user_id = a.user_id
    group by f.user_id, f.first_active_date
)
select
    first_active_date,                                           -- 首次活跃日期
    count(*) as cohort_users,                                    -- 该日期首次活跃的用户数
    sum(retained_1d) as retained_1d_users,                       -- 次日留存用户数
    round(1.0 * sum(retained_1d) / count(*), 4) as retention_1d,  -- 次日留存率
    sum(retained_3d) as retained_3d_users,                       -- 3 日留存用户数
    round(1.0 * sum(retained_3d) / count(*), 4) as retention_3d,  -- 3 日留存率
    sum(retained_7d) as retained_7d_users,                       -- 7 日留存用户数
    round(1.0 * sum(retained_7d) / count(*), 4) as retention_7d   -- 7 日留存率
from retention_flags
group by first_active_date
order by first_active_date;
```

用途：分析不同日期首次活跃用户在后续日期是否继续活跃。注意：本项目数据只到 `2017-12-03`，所以靠后的日期无法完整观察 7 日留存。

## 12. 商品购买次数 Top10

```sql
select
    item_id,                                      -- 商品 ID
    count(*) as buy_count,                       -- 商品被购买的次数
    count(distinct user_id) as buyer_count       -- 购买该商品的去重用户数
from user_behavior
where behavior_type = 'buy'                      -- 只统计购买行为
group by item_id
order by buy_count desc, buyer_count desc        -- 先按购买次数降序，再按购买用户数降序
limit 10;
```

用途：找出购买次数最高的 10 个商品，辅助分析热门商品。

## 13. 类目购买次数 Top10

```sql
select
    category_id,                                  -- 类目 ID
    count(*) as buy_count,                       -- 该类目下商品被购买的次数
    count(distinct user_id) as buyer_count       -- 购买该类目的去重用户数
from user_behavior
where behavior_type = 'buy'                      -- 只统计购买行为
group by category_id
order by buy_count desc, buyer_count desc        -- 先按购买次数降序，再按购买用户数降序
limit 10;
```

用途：找出购买次数最高的 10 个商品类目，类目层面比单个商品更适合总结用户偏好。

## 14. 本项目常用指标口径

| 指标 | SQL 口径 |
|---|---|
| PV | `behavior_type = 'pv'` 的行为次数 |
| UV | `count(distinct user_id)` |
| 购买次数 | `behavior_type = 'buy'` 的行为次数 |
| 购买用户数 | 有过 `buy` 行为的去重用户数 |
| 兴趣行为 | `fav` 收藏 + `cart` 加购 |
| 浏览到购买转化率 | 购买次数 / 浏览次数 |
| 复购用户 | 购买次数大于等于 2 的用户 |
| 复购率 | 复购用户数 / 购买用户总数 |

