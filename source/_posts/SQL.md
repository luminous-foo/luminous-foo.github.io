---
title: SQL整理
date: 2021-02-22 21:16:31
tags:
- sql
categories: 
---

# SQL整理

## 维度打平

```sql
with t1 as (
    select id  industry_id
    ,industry_name 
    from weier_ai.ods_monkey_ai_industry_df 
    where ds = '${ds}' 
    and id is not null 
    and industry_name is not null 
    and status = 0 
    group by id, industry_name
    
),
t2 as(
    select 
    'ks'                                   app 
    ,cid                                   cate_id
    ,name                                  cate_name
    ,parent_cid                            parent_cate_id
    ,cat_level                             cate_level
    ,if(is_parent = 1, 1, 0)               is_parent  
    ,industry_id                           industry_id
    ,status                                cate_status
    ,features                              feature                    
    ,gmt_create                            create_gmt
    ,gmt_modified                          update_gmt    
    ,ds    
    -- 这表中 有平台id 字段   
    from weier_ai.ods_resource_platform_product_category_df 
    where   ds = '${ds}'  
    AND     cid is not null 
    AND     name is not null 
    AND     status = '0'
)
select
a.app 
,a.cate_id
,a.cate_name
,a.parent_cate_id 
,CASE
            WHEN d.cate_id is not null THEN 4
            WHEN d.cate_id is null and c.cate_id is not null THEN 3
            WHEN c.cate_id is null and b.cate_id is not null THEN 2
            when b.cate_id is null and a.cate_id is not null THEN 1
    END     cate_level 
,a.is_parent
,a.industry_id
,f.industry_name
,a.cate_status
,a.feature
,a.create_gmt
,a.update_gmt
,CASE
            WHEN d.cate_id is not null THEN d.cate_id
            WHEN d.cate_id is null and c.cate_id is not null THEN c.cate_id
            WHEN c.cate_id is null and b.cate_id is not null THEN b.cate_id 
            when b.cate_id is null and a.cate_id is not null THEN a.cate_id 
    END cate_1_id
,CASE   
            WHEN d.cate_name is not null THEN d.cate_name
            WHEN d.cate_id is null and c.cate_name is not null THEN c.cate_name
            WHEN c.cate_id is null and b.cate_name is not null THEN b.cate_name 
            when b.cate_id is null and a.cate_name is not null THEN a.cate_name 
    END cate_1_name 
,CASE   
            WHEN d.cate_id is not null and c.cate_id is not null THEN c.cate_id 
            WHEN d.cate_id is null and c.cate_id is not null and b.cate_id is not null THEN b.cate_id 
            WHEN c.cate_id is null and b.cate_id is not null and a.cate_id is not null THEN a.cate_id 
            else null 
    END cate_2_id 
,CASE    
            WHEN d.cate_name is not null and c.cate_name is not null THEN c.cate_name 
            WHEN d.cate_name is null and c.cate_name is not null and b.cate_name is not null THEN b.cate_name 
            WHEN c.cate_name is null and b.cate_name is not null and a.cate_name is not null THEN a.cate_name  
            else null 
    END cate_2_name
,CASE  
            WHEN d.cate_id is not null and c.cate_id is not null and b.cate_id is not null THEN b.cate_id 
            WHEN d.cate_id is null and c.cate_id is not null and b.cate_id is not null and a.cate_id is not null THEN a.cate_id 
            else null 
    END cate_3_id
,CASE     
            WHEN d.cate_name is not null and c.cate_name is not null and b.cate_name is not null THEN b.cate_name 
            WHEN d.cate_name is null and c.cate_name is not null and b.cate_name is not null and a.cate_name is not null THEN a.cate_name 
            else null 
    END cate_3_name
,CASE   
            WHEN  d.cate_id is not null and c.cate_id is not null and b.cate_id is not null and a.cate_id is not null THEN a.cate_id 
            else null 
    END cate_4_id
,CASE   
            WHEN d.cate_name is not null and c.cate_name is not null and b.cate_name is not null and a.cate_name is not null THEN a.cate_name  
            else null 
    END cate_4_name
from  t2 a 
left join t2 b
on  a.parent_cate_id = b.cate_id 
left join t2 c
on  b.parent_cate_id = c.cate_id 
left join t2 d
on  c.parent_cate_id = d.cate_id  
LEFT JOIN   t1  f 
ON   a.industry_id = f.industry_id 
; 

--

SELECT  
a.app 
,a.cate_id
,a.cate_name
,a.parent_cate_id
,a.cate_level --1 为最高级
,a.is_parent
,a.industry_id
,f.industry_name
,a.cate_status
,a.feature
,a.create_gmt
,a.update_gmt
,CASE    WHEN a.cate_level = 1 THEN a.cate_id
            WHEN b.cate_level = 1 THEN b.cate_id
            WHEN c.cate_level = 1 THEN c.cate_id
            WHEN d.cate_level = 1 THEN d.cate_id 
            ELSE NULL 
    END cate_1_id
,CASE    WHEN a.cate_level = 1 THEN a.cate_name
            WHEN b.cate_level = 1 THEN b.cate_name
            WHEN c.cate_level = 1 THEN c.cate_name
            WHEN d.cate_level = 1 THEN d.cate_name 
            ELSE NULL 
    END cate_1_name
,CASE    WHEN a.cate_level = 2 THEN a.cate_id
            WHEN b.cate_level = 2 THEN b.cate_id
            WHEN c.cate_level = 2 THEN c.cate_id
            WHEN d.cate_level = 2 THEN d.cate_id 
            ELSE NULL 
    END cate_2_id
,CASE    WHEN a.cate_level = 2 THEN a.cate_name
            WHEN b.cate_level = 2 THEN b.cate_name
            WHEN c.cate_level = 2 THEN c.cate_name
            WHEN d.cate_level = 2 THEN d.cate_name 
            ELSE NULL 
    END cate_2_name
,CASE    WHEN a.cate_level = 3 THEN a.cate_id
            WHEN b.cate_level = 3 THEN b.cate_id
            WHEN c.cate_level = 3 THEN c.cate_id
            WHEN d.cate_level = 3 THEN d.cate_id 
            ELSE NULL 
    END cate_3_id
,CASE    WHEN a.cate_level = 3 THEN a.cate_name
            WHEN b.cate_level = 3 THEN b.cate_name
            WHEN c.cate_level = 3 THEN c.cate_name
            WHEN d.cate_level = 3 THEN d.cate_name 
            ELSE NULL 
    END cate_3_name
,CASE    WHEN a.cate_level = 4 THEN a.cate_id
            WHEN b.cate_level = 4 THEN b.cate_id
            WHEN c.cate_level = 4 THEN c.cate_id
            WHEN d.cate_level = 4 THEN d.cate_id 
            ELSE NULL 
    END cate_4_id
,CASE    WHEN a.cate_level = 4 THEN a.cate_name
            WHEN b.cate_level = 4 THEN b.cate_name
            WHEN c.cate_level = 4 THEN c.cate_name
            WHEN d.cate_level = 4 THEN d.cate_name 
            ELSE NULL 
    END cate_4_name
FROM    ods_monkey_product_category_df a
LEFT JOIN ods_monkey_product_category_df b
on      a.parent_cate_id = b.cate_id 
LEFT JOIN   ods_monkey_product_category_df c
ON      b.parent_cate_id = c.cate_id
LEFT JOIN ods_monkey_product_category_df d
ON      c.parent_cate_id = d.cate_id 
LEFT JOIN   ods_monkey_ai_industry_df  f
ON   a.industry_id = f.industry_id  
```

## 订单处理

```sql


INSERT OVERWRITE TABLE dwd_order_child_order_info_du
PARTITION( ds = '${ds}' , app = 'tb' )  
select 
/*+mapjoin(b)*/ 
-- 流程节点  
gmt_order_create   order_create_gmt 
,gmt_order_pay     pay_gmt   
,gmt_order_ship    ship_gmt 
,gmt_order_end     order_end_gmt 
,gmt_order_rated   rated_gmt      
,if(refund_status='SUCCESS',gmt_modified, NULL)    refund_gmt  
--订单状态   
,case when gmt_order_rated>gmt_order_create then 6
when a.order_status='TRADE_NO_CREATE_PAY' or a.order_status='WAIT_BUYER_PAY' then 1
when a.order_status='WAIT_SELLER_SEND_GOODS' then 2
when a.order_status='SELLER_CONSIGNED_PART' or a.order_status='WAIT_BUYER_CONFIRM_GOODS' then 3
when a.order_status='TRADE_FINISHED' then 4
when a.order_status='TRADE_CLOSED' or a.order_status='TRADE_CLOSED_BY_TAOBAO' then 5
else null end as order_status    							--1：待付款    2：待发货    3：已发货    4：已完成    5：已关闭    6：已评价
--退货状态
,case when a.refund_status='WAIT_SELLER_AGREE' then 1
when a.refund_status='WAIT_BUYER_RETURN_GOODS' or a.refund_status='SELLER_REFUSE_BUYER' then 2
when a.refund_status='WAIT_SELLER_CONFIRM_GOODS' then 3
when a.refund_status='SUCCESS' then 4
when a.refund_status='CLOSED' then 5
else null end as   refund_status  							--1、申请退款    2、处理退款申请    3、退货退款中    4、退款成功    5、退款关闭
,if(gmt_order_end is not null and order_status = 'TRADE_FINISHED', 1, 0)  is_order_success  
,if(refund_status = 'SUCCESS', 1, 0)   is_refund_success 
,if(gmt_order_rated is not null, buyer_rate, 0 ) is_buyer_rated   -- 需要处理   

--维度字段    
,tid   order_id 
,oid   child_order_id  
,c.product_id
,a.iid  product_src_id  
,c.product_name 
,b.shop_id           shop_id   
,b.shop_name         shop_name 
,b.shop_account 
,b.org_id            org_id 
,b.org_name          org_name  
 
,buyer_nick          buyer_id  
,GET_PHONE(tb_tel(GET_JSON_OBJECT(a.features, '$.wr_t_code')))         buyer_phone 
,GET_JSON_OBJECT(a.features, '$.receiver_name')             buyer_name   

,null    promotion_id   
,GET_ADDRESS(tb_order_decode(GET_JSON_OBJECT(a.features, '$.wr_s_code')),4) as province	--省份
,tb_order_decode(GET_JSON_OBJECT(a.features, '$.wr_c_code')) as city	 	--城市
,tb_order_decode(GET_JSON_OBJECT(a.features, '$.wr_d_code')) as town		--地区 
-- 
,if(post_fee is not null, post_fee, 0 )   post_fee_amt
,if(payment is not null,   payment , 0)   payment_amt
,if(num > 0, num, 1)   product_num 
,if(price > 0, price, if(num > 0, ROUND((total_fee+discount_fee)/num), total_fee+discount_fee))   product_amt  
,if(num*price > 0, num*price, total_fee+discount_fee )   product_total_amt    
,if(discount_fee is not null,abs(discount_fee), 0)  discount_amt 
--
,a.gmt_create       create_gmt  
,a.gmt_modified     update_gmt  
,if(a.status = 0, 0, 1)   is_del  
,a.features         feature 
,TO_JSON("flag", flag
    ,"reason",reason
    ,"refund_success_time",refund_success_time
)    extra_info 

from  (
    select *
    from  weier_ai.ods_order_info_du 
    where ds = '${ds}' 
    and tid is not null 
    AND oid is not null 
    AND iid is NOT NULL 
    AND seller_nick is NOT NULL 
    AND buyer_nick is NOT NULL 
    AND gmt_order_create is NOT NULL  
)   a 
-- 排除非客户的订单数据
join (
    select * 
    from (
        select * 
        ,ROW_NUMBER()over(partition by app, shop_account order by shop_end_gmt desc) rk 
        from weier_ai.dim_shop 
        where ds = '${ds}' 
        -- 只获取 tb 的店铺  
        and app = 'tb' 
        and shop_id is not null 
        and shop_account is not null 
    ) 
    where rk = 1 
)   b 
on  trim(a.seller_nick) = trim(b.shop_account) 
left JOIN (
    select * 
    from (
        select * 
        ,ROW_NUMBER()over(partition by app, shop_account, product_src_id order by create_gmt desc) rk 
        from weier_ai.dim_product 
        where ds = '${ds}'  
        -- 只获取 tb 的商品
        and app = 'tb'
        and product_id is not null 
        and product_src_id is not null 
        and shop_account is not null 
    ) 
    where rk = 1 
)  c 
on a.iid = c.product_src_id 
and trim(b.shop_account) = trim(c.shop_account)     
;


set odps.stage.mapper.split.size = 512;
INSERT OVERWRITE TABLE dwd_order_child_order_info_di
PARTITION( ds,app )  
select 
-- 流程节点  
order_create_gmt 
,pay_gmt   
,ship_gmt 
,order_end_gmt 
,rated_gmt      
,refund_gmt     
,order_status   
,refund_status  
,is_order_success  
,is_refund_success 
,is_buyer_rated   
--维度字段    
,order_id 
,child_order_id  
,product_id
,product_src_id  
,product_name
,shop_id  
,shop_name
,shop_account 
,org_id
,org_name
,buyer_id
,buyer_phone
,buyer_name
,promotion_id
,province
,city
,town
,post_fee_amt
,payment_amt
,product_num
,product_amt
,product_total_amt
,discount_amt
,create_gmt
,update_gmt
,is_del
,feature
,extra_info
    
--重新放进order_create_gmt订单创建时间分区
,TO_CHAR(order_create_gmt, 'yyyymmdd') ds
,app     
from dwd_order_child_order_info_du 
where ds = '${ds}' and app='tb'
and TO_CHAR(order_create_gmt, 'yyyymmdd') > TO_CHAR(dateadd(to_date('${ds}', 'yyyymmdd'), -9, 'mm'),'yyyymmdd')

union all 

select 
-- 流程节点  
a.order_create_gmt 
,a.pay_gmt   
,a.ship_gmt 
,a.order_end_gmt 
,a.rated_gmt      
,a.refund_gmt     
,a.order_status   
,a.refund_status  
,a.is_order_success  
,a.is_refund_success 
,a.is_buyer_rated   
--维度字段    
,a.order_id 
,a.child_order_id  
,a.product_id
,a.product_src_id  
,a.product_name
,a.shop_id  
,a.shop_name
,a.shop_account 
,a.org_id
,a.org_name
,a.buyer_id
,a.buyer_phone
,a.buyer_name
,a.promotion_id
,a.province
,a.city
,a.town
,a.post_fee_amt
,a.payment_amt
,a.product_num
,a.product_amt
,a.product_total_amt
,a.discount_amt
,a.create_gmt
,a.update_gmt
,a.is_del
,a.feature
,a.extra_info  
--
,a.ds    
,a.app 
from (
     -- 发生变动的老分区订单数据   
    select * 
    from dwd_order_child_order_info_di  
    where ds in (
        select TO_CHAR(order_create_gmt, 'yyyymmdd')  
        from dwd_order_child_order_info_du  
        where ds = '${ds}' and app='tb' 
        and order_create_gmt is not null 
        and TO_CHAR(order_create_gmt, 'yyyymmdd') > TO_CHAR(dateadd(to_date('${ds}', 'yyyymmdd'), -9, 'mm'),'yyyymmdd')
        group by TO_CHAR(order_create_gmt, 'yyyymmdd')   
    )  
    and app='tb'
    and order_id is not null 
    and child_order_id is not null 
) a 
left join (
    -- 新订单数据
    select * 
    from dwd_order_child_order_info_du  
    where ds = '${ds}' and app='tb' 
    and order_id is not null 
    and child_order_id is not null 
) b 
on a.order_id = b.order_id 
and a.child_order_id = b.child_order_id 
where b.child_order_id is null 
;
```

## 正则处理

```sql
 select
    replace(
    replace(
    replace(
    replace(
    regexp_replace(
        regexp_substr(
            content,'\{.*\}'),'\\\\',''),'\"\{','{'),
            '\}\"','}'),'\"\[','['),'\]\"',']'
            ) json
    from ods_loghub_weierai_im_group_online_di_test
    where
    ds = '${ds}'
    and INSTR(content,'发送至客户端成功')>0


-- select 
-- case when company regexp '.*公司$' then 1 when company regexp '.*中心$' then 2 end company_category;

select regexp_extract('IloveYou','(I)(.*?)(You)',2) ;
select regexp_replace("IloveYou","You","") ;

-- 1 REGEXP_LIKE ：与LIKE的功能相似
--查询value中以1开头60结束的记录并且长度是7位
select * from fzq where value like '1____60';
select * from fzq where regexp_like(value,'1....60');

-- 2 REGEXP_INSTR ：与INSTR的功能相似
--返回匹配上的初始索引位置
select REGEXP_INSTR('http3213', 'http') ; --1

-- 3 REGEXP_SUBSTR ：与SUBSTR的功能相似
select REGEXP_SUBSTR('http3213', 'http') ; --http
-- 4 REGEXP_REPLACE ：与REPLACE的功能相似

-- REGEXP_SUBSTR延伸substr()函数的功能，让你搜索一个正则表达式模式字符串。

-- 这也类似于REGEXP_INSTR，而是返回子字符串的位置，它返回的子字符串本身。

-- 5 REGEXP_COUNT(msg_content, "[.?!。？！]+") 统计匹配上的正则出现次数
select REGEXP_COUNT('！2131232131FFF!', "[.?!。？！]+") ;
```

