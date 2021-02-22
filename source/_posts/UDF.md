---
title: UDF整理
date: 2021-02-22 21:16:31
tags:
- udf
categories: 
---

# UDF

```python
#coding:utf-8
from odps.udf  import annotate
import re
@annotate("string,bigint->string")
# class GetAddress(object):
#     def evaluate(self, str,index):
#         if str is None or len(str) <= 0:
#             return None
#         pattern = re.compile(r'(?P<province>上海市|天津市|北京市|云南省|台湾省|吉林省|四川省|安徽省|山东省|山西省|广东省|江苏省|江西省|河北省|河南省|浙江省|海南省|湖北省|湖南省|甘肃省|福建省|黑龙江省|贵州省|辽宁省|重庆市|陕西省|青海省|香港特别行政区|西藏自治区|澳门特别行政区|广西壮族自治区|新疆维吾尔自治区|内蒙古自治区|宁夏回族自治区)(?P<city>.*?市|.*?自治州|.*县|.*?区|.*行政单位|.*市辖区|.*?行政区|.*盟)(?P<county>[^县]+县|.+县|.+区|.+市|.+旗|.+海域|.+岛)?(?P<town>[^区]+区|.+镇)?(?P<village>.*)')
#         #pattern = re.compile(r'(?<province>[^省]+省|.+自治区|.*?自治区|.*?省|.*?行政区|上海市|北京市|天津市|重庆市)(?P<city>[^市]+市|.+自治州)(P?<county>[^县]+县|.+区|.+镇|.+局)?(?P<town>[^区]+区|.+镇)?(?P<village>.*)')
        
#         try:
#             match = re.match(pattern, str.split('收货人')[0].split('所在地区')[0].split('超市')[0].split('夜市')[0].split('小区')[0].split('校区')[0])
#             if match:
#                 # 使用Match获得分组信息
#                 if(index==1):return (match.group("province"))
#                 if(index==2):return(match.group("city"))
#                 if(index==3):return(match.group("county"))
#             # return regions
#         except Exception as e:
#             print(e)
#             return None
class GetAddress(object):
    def evaluate(self, address, index):
        if address is None or len(address) <= 0:
            return None

        try:
            if (index == 4):
                p = re.search(r'(上海|天津|北京|重庆)',address)
                if p is not None :return (p.group()+'市')
                else:return address
            recity = re.compile(
                r'(?P<city>.*?自治州|.*?自治县|兴安盟|锡林郭勒盟|阿拉善盟|大兴安岭地区|和田地区|阿克苏地区|阿勒泰地区|阿里地区|塔城地区|喀什地区|.*?市|.*?县|.*?区)?(?P<county>.*)')
            recounty = re.compile(r'(?P<county>.*?县|.{2,5}区|.*?镇|.*?旗|.+海域|.*?市|.+岛)?(?P<town>.*)')
            pattern = re.compile(
                r'(?P<province>上海市|天津市|北京市|云南省|台湾省|吉林省|四川省|安徽省|山东省|山西省|广东省|江苏省|江西省|河北省|河南省|浙江省|海南省|湖北省|湖南省|甘肃省|福建省|黑龙江省|贵州省|辽宁省|重庆市|陕西省|青海省|香港特别行政区|西藏自治区|澳门特别行政区|广西壮族自治区|新疆维吾尔自治区|内蒙古自治区|宁夏回族自治区)?(?P<city>.*)')
            match = re.match(pattern, address.split('收货人')[0])
            province = match.group("province");
            if (index == 1): return (province)
            addr1 = match.group("city").replace('市辖区', province).replace('所在地区', '').replace('自治区直辖县', '').replace('省直辖县',
                                                                                                              '').replace(
                '省直辖县级行政区划', '').replace('级行政区划', '')
            city = re.match(recity, addr1[:35]).group("city")
            if (len(province) > 0 and province.__contains__("市")):
                city = province
            if (index == 2):
                if (len(province) > 0):
                    return (city)
                else:
                    return None
            if (len(city) == 0 or city == ''):
                addr2 = addr1
            else:
                addr2 = addr1[len(city):]
            county = re.match(recounty, addr2[:20]).group("county")
            if (index == 3): return (county.replace(province,''))


            # return regions
        except Exception as e:
            print(e)
            return None

#coding:utf-8
from odps.udf  import annotate

# 此资源用于处理字符串

import json
@annotate("*->string")
class ToJson(object):
    def evaluate(self, *args):
        data = {}
        for index in range(len(args)):
            if index % 2 == 0:
                data[args[index]] = args[index+1]

        return json.dumps(data,ensure_ascii=False) 

#coding:utf-8
from odps.udf  import annotate

# 通过变量给数据打标签


@annotate("bigint->bigint")
class SetGroupFlag(object):

    flag_value = 1

    def evaluate(self, inputdata):
        if inputdata is None :
            return None

        try:
            if inputdata == 0 :
                return self.flag_value
            elif inputdata == 1 :
                self.flag_value += 1
                return self.flag_value
            else:
                return None
        except Exception as e:
            print(e)
            return None


#coding:utf-8
from odps.udf  import annotate

# 从字符串中获取url
import re
@annotate("string->array<string>")
class GetURLs(object):
    specialCharList = [']', '[','-', ',', '$', '(', ')', '#', '+', '&', '*', '?', '!', '.', '`', '~', '@', '%', '^',
                       '\\', '。',
                       '？', '！', '】', '【']

    def getMatchCount(self, str):
        count = 0
        for i in range(len(str) - 1, -1, -1):
            if str[i] in self.specialCharList:
                count += 1
            else:
                break
        return count

    def evaluate(self, str):
        if str is None or len(str) <= 0:
            return None

        # 匹配模式
        pattern = re.compile(
            r'http[s]?://(?:(?!http[s]?://)[a-zA-Z]|[0-9]|[=?$\-_@.&+/]|[!*\(\),]|(?:%[0-9a-fA-F][0-9a-fA-F]))+')
        urls = re.findall(pattern, str)

        try:
            for i in range(len(urls)):
                length = self.getMatchCount(urls[i])
                if length != 0:
                    urls[i] = urls[i][0:-length]
            if len(urls) > 0:
                return urls
            else:
                return None
        except:
            return None

#coding:utf-8
from odps.udf  import annotate
@annotate("*->array<String>")
class GetSetNvl(object):
    def evaluate(self, *args):
        data = set()
        result = []
        for value in args:
            if value is not None:
                if isinstance(value,list):
                    if len(value)>0:
                        for vue1 in value:
                            data.add(vue1)
                else:
                    data.add(value)
        for s in data:
            result.append(s)
        return result

#coding:utf-8
from odps.udf  import annotate
import re
@annotate("string->string")
class GetPhone(object):
    pattern = re.compile("^1\d{10}$|^861\d{10}$|^001\d{10}$")
    def evaluate(self, str):
        if str is None or len(str) <= 0:
            return None
        if re.match(self.pattern, str):
            if len(str)==11 and str[0]=="1":
                return str
            if len(str)==13 and str[0:3]=="861":
                return str[2:13]
            if len(str) == 13 and str[0:3] == "001":
                return str[2:13]
            else : None 
        else :
            return None 
            

#coding:utf-8
from odps.udf  import annotate
import re 

# 用于提取html 标签属性的值
@annotate("string,string,string,string->array<String>")
class GetHTMLABLEATTR(object):
    """   
     :param value: 固定参数，定义当天日期
     :param lable_name: 固定参数，标签名
     :param attr: 固定参数，表示当天是否学习python
     :param extend_regex: 默认参数，扩展正则对属性里面的值进行提取 注意一定要加上(?:regex)表示非捕获分组
     :return:返回匹配的数据 list
     """
    def evaluate(self, value,lable_name, attr, extend_regex=''):
        data = []
        if attr is not None and len(attr) > 0 and lable_name is not None and len(lable_name) > 0 and extend_regex is not None:
            # regex = '<{lable_name}[^>]*{attr}\s*=\s*[\"\']?([^\s\"\'>]*){extend_regex}'
            regex = '<' + lable_name + '[^>]*' + attr + '\s*=\s*[\"\']?([^\s\"\'>]*' + extend_regex +')'
            pattern = re.compile(
                r'%s'%regex,re.M | re.I
            )
        else:
            return data
        if value is not None and len(value) > 0:
            try:
                results = re.findall(pattern, value)
                for result in results:
                    data.append(result)
            except Exception as e:
                print(e)
                return []
        return data

```

