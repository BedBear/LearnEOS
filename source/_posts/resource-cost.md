---
title: 【玩转EOS】转账、抵押等操作需要消耗多少net和cpu资源？
date: 2018-07-21 23:16:24
tags: cpu net
---

在EOS网络里面进行转账操作时，相信不少人都会遇到过因net或cpu资源不足，导致转账失败的情况。本文就结合实际的转账操作，分析一下转账EOS大致需要多少net和cpu资源。

## 写在前面
本文仅从实际使用者的角度，通过重复多次执行相同的操作，两次操作中间不超过30秒，观察net和cpu资源的使用情况，从而统计分析出不同转账方式，所消耗的资源数量。下面会先给出分析结果，文末会附上试验过程数据。

## net和cpu资源消耗测试数据

下面表格展示了 **转账** 和 **抵押** 操作平均消耗的资源。**转账时所带memo不同，消耗的资源也不同，而转账金额的多少不影响消耗** 。

|操作方式|命令示意|net消耗(KiB)|cpu消耗(ms)|
|--|--|--|--|
|【转账】不带memo|cleos transfer 账户A 账户B '0.0001 EOS' ''|0.124|0.884|
|【转账】带短memo|cleos transfer 账户A 账户B '0.0001 EOS' '感谢！'|0.133|1.018|
|【转账】带最长memo|cleos transfer 账户A 账户B '0.0001 EOS' '1111…<256个1>'|0.374|1.306|
|【转账】带中文memo|cleos transfer 账户A 账户B '0.0001 EOS' '测测测…<85个测>'|0.374|1.046|
|【抵押】换net和cpu|cleos system delegatebw 账户A 账户A "0.01 EOS" "0.01 EOS" |0.140|2.7671|

> 转账eos所带memo规定不能超过 256 bytes。以utf-8编码的中文字符占 3 bytes，所以在memo中的中文字符不能超过85个。

## net和cpu资源获取、消耗以及恢复

首先，通过下面的命令可以查看账户的cpu和net资源信息

```
> cleos get account <账户名>
...
net bandwidth: 
     staked:          0.1000 EOS           (total stake delegated from account to self)
     delegated:       1.0000 EOS           (total staked delegated to account from others)
     used:             5.487 KiB  
     available:        671.6 KiB  
     limit:              677 KiB  

cpu bandwidth:
     staked:          1.1000 EOS           (total stake delegated from account to self)
     delegated:       1.0000 EOS           (total staked delegated to account from others)
     used:             31.69 ms   
     available:        9.779 ms   
     limit:            41.47 ms  
...
```
- **staked**    本账户为自己抵押的eos数量
- **delegated** 别的账户为本账户抵押的eos数量
- **used**      已消耗的资源量
- **available** 可用的资源量
- **limit**     可消耗的资源上限

### net和cpu资源获取
EOS系统中，cpu和net资源是通过抵押eos获取的。抵押的时候只记录了抵押cpu/net对应的eos数量，获取到的资源不是固定的，而是在使用的时候会按照当前你所抵押eos的数量占全网抵押总量的比例实时计算的，最终体现为 **limit** 的值。实时计算由 eos/libraries/chain/resource_limits.cpp 中 get_account_net_limit_ex 和 get_account_cpu_limit_ex 方法实现。

计算 net 和 cpu 上限的算法基本一样，下面以 get_account_net_limit_ex 方法作为例子来说明。

```c++
account_resource_limit resource_limits_manager::get_account_net_limit_ex( const account_name& name ) const {
   const auto& config = _db.get<resource_limits_config_object>();
   const auto& state  = _db.get<resource_limits_state_object>();
   const auto& usage  = _db.get<resource_usage_object, by_owner>(name);

   int64_t net_weight, x, y;

   # 获取本账户的net资源eos抵押量 net_weight
   get_account_limits( name, x, net_weight, y ); 

   if( net_weight < 0 || state.total_net_weight == 0) {
      return { -1, -1, -1 };
   }

   account_resource_limit arl;

   uint128_t window_size = config.account_net_usage_average_window;
   # 窗口期内的全网net资源总量
   uint128_t virtual_network_capacity_in_window = state.virtual_net_limit * window_size;
   # 本账户的net资源eos抵押量
   uint128_t user_weight     = (uint128_t)net_weight;
   # 全网用户的net资源eos抵押总量
   uint128_t all_user_weight = (uint128_t)state.total_net_weight;
   # 窗口期内，该账户最大可用的net资源量
   auto max_user_use_in_window = (virtual_network_capacity_in_window * user_weight) / all_user_weight;
   # 窗口期内，该账户已使用net资源量
   auto net_used_in_window  = impl::integer_divide_ceil((uint128_t)usage.net_usage.value_ex * window_size, (uint128_t)config::rate_limiting_precision);

   if( max_user_use_in_window <= net_used_in_window )
      arl.available = 0;
   else
      # 窗口期内，该账户可用net资源量
      arl.available = impl::downgrade_cast<int64_t>(max_user_use_in_window - net_used_in_window);

   arl.used = impl::downgrade_cast<int64_t>(net_used_in_window);
   arl.max = impl::downgrade_cast<int64_t>(max_user_use_in_window);
   return arl;
}
```
> 窗口期内账户最大可用的net资源量 `limit` = 窗口期内全网net资源总量 * 本账户的net资源eos抵押量 / 全网用户的net资源eos抵押总量
>
>窗口期内账户可用net资源量 `available` = 窗口期内账户最大可用的net资源量 `limit` - 窗口期内该账户已使用net资源量 `used`

通过上面的介绍，可以知道相同的eos抵押量，在不同时间可以使用资源量是会根据全网的资源情况以及全网抵押总量是动态变化的。可以通过[这里](https://explorer.eoseco.com/)查看实时的资源成本

### net和cpu资源消耗和恢复
net和cpu资源消耗和恢复的部分计算是由 
eos/libraries/chain/include/eosio/chain/resource_limits_private.hpp 
中 exponential_moving_average_accumulator 的 add 方法实现的。

```c++
void add( uint64_t units, uint32_t ordinal, uint32_t window_size /* must be positive */ )
 {
    // check for some numerical limits before doing any state mutations
    EOS_ASSERT(units <= max_raw_value, rate_limiting_state_inconsistent, "Usage exceeds maximum value representable after extending for precision");
    EOS_ASSERT(std::numeric_limits<decltype(consumed)>::max() - consumed >= units, rate_limiting_state_inconsistent, "Overflow in tracked usage when adding usage!");
    # 本次操作资源消耗量
    auto value_ex_contrib = downgrade_cast<uint64_t>(integer_divide_ceil((uint128_t)units * Precision, (uint128_t)window_size));
    EOS_ASSERT(std::numeric_limits<decltype(value_ex)>::max() - value_ex >= value_ex_contrib, rate_limiting_state_inconsistent, "Overflow in accumulated value when adding usage!");

    if( last_ordinal != ordinal ) {
       FC_ASSERT( ordinal > last_ordinal, "new ordinal cannot be less than the previous ordinal" );
       if( (uint64_t)last_ordinal + window_size > (uint64_t)ordinal ) {
          # 本次与上次操作之间的时间间隔
          const auto delta = ordinal - last_ordinal; // clearly 0 < delta < window_size
          # 已使用资源量衰减系数
          const auto decay = make_ratio(
                  (uint64_t)window_size - delta,
                  (uint64_t)window_size
          );
          # 计算已使用资源衰减后剩余量
          value_ex = value_ex * decay;
       } else {
          value_ex = 0;
       }

       last_ordinal = ordinal;
       consumed = average();
    }

    consumed += units;
    # 计算本次操作后资源已使用量
    value_ex += value_ex_contrib;
 }
```

> 本次操作后的资源已使用量 `used` = 上次操作后的资源已使用量 * 衰减系数 + 本次操作的资源消耗量
>
> `衰减系数` = (window_size - (本次时间段序号 - 上次时间段序号)) / window_size

其中，`衰减系数` 中涉及的 window_size 的值可以通过下面的方式得到
```c++
# 查看 eos/libraries/chain/include/eosio/chain/config.hpp
const static int      block_interval_ms = 500;
static const uint32_t account_cpu_usage_average_window_ms  = 24*60*60*1000l;
static const uint32_t account_net_usage_average_window_ms  = 24*60*60*1000l;

# 查看 eos/libraries/chain/resource_limits.cpp
uint32_t account_cpu_usage_average_window = config::account_cpu_usage_average_window_ms / config::block_interval_ms;
uint32_t account_net_usage_average_window = config::account_net_usage_average_window_ms / config::block_interval_ms;

uint128_t window_size = config.account_cpu_usage_average_window;
uint128_t window_size = config.account_net_usage_average_window;

# 可以推导出 window_size 的值
window_size = 24 * 60 * 60 * 1000 / 500
```

> 我们可以理解为，cpu和net资源的恢复周期是24小时。

举个例子，上一次操作后资源已使用量为 x，经过 t 小时以后，再执行一次操作，那么本次操作后的资源已使用量的计算方式如下：
used = x * ((24 - t) / 24) + 本次操作资源消耗量

## 总结
本文通过实际操作试验和结合源码分析了eos系统中，介绍了net和cpu资源获取、消耗以及恢复等涉及的相关知识。如果想了解试验过程中的数据可以查看 **【附录】试验数据**。

## 【附录】试验数据

1【转账】不带memo
```
cleos transfer 账户A 账户B '0.0001 EOS' ''
```
![image](https://user-images.githubusercontent.com/8636635/43038566-051bd878-8d4e-11e8-8dac-1b4f7d7ce163.png)

2【转账】不同金额
```
cleos transfer 账户A 账户B '0.1 EOS' ''
```
![image](https://user-images.githubusercontent.com/8636635/43038788-9d812aec-8d52-11e8-8185-5c138839ccda.png)

3【转账】带短memo
```
cleos transfer 账户A 账户B '0.0001 EOS' '感谢！'
```
![image](https://user-images.githubusercontent.com/8636635/43038747-dc6a3f1a-8d51-11e8-9236-762ba9a89285.png)

4【转账】带最长memo
```
cleos transfer 账户A 账户B '0.0001 EOS' '1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111'
```
![image](https://user-images.githubusercontent.com/8636635/43038580-72199d20-8d4e-11e8-9bc4-00747ab10fe9.png)

5【转账】带中文memo
```
cleos transfer 账户A 账户B '0.0001 EOS' '测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测'
```
![image](https://user-images.githubusercontent.com/8636635/43038677-706d87b4-8d50-11e8-88da-ffc4573441e3.png)

6【抵押】换net和cpu
```
cleos system delegatebw 账户A 账户A "0.01 EOS" "0.01 EOS"
```
![image](https://user-images.githubusercontent.com/8636635/43038799-fd30d078-8d52-11e8-923a-a9a82abb7b02.png)

## 【参考】
- [EOS零手续费免费?你不知道的EOS收费细节](https://blog.csdn.net/itleaks/article/details/80743836)