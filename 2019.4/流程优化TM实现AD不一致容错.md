# 数据表

## 辅助信息表

ETVBMS.t_netinfo 
|  sn | hgwmac | srip| planebip | countnum  | createtime  | updatetime |
| - | - | - | - | - | - | -|
|  |  |  |  |  |  | |


## AD一致信息表

iptvboss.t_netsame

|  sn | etvaccount | iptvaccount | ifadsame | ifdemo  | notsame_type  | notsame_ad | notsame_count | createtime | updatetime |
| - | - | - | - | - | - | -| - | -|
|  |  |  |  |  |  | | | |

# 业务逻辑

## ServiceAuth

```
调用radius接口，获取AD、accesstype、pppoeaccount
演示终端、CallError、RadiusOff ： AD用etvuser.adname覆盖，accesstype分别是demoAccesstype|CallError|RadiusOff,pppoeaccount用etvuser.pppoeaccount覆盖

NetSameInfo初始化： ifadsame='Y',ifdemo='N',notsame_type='',notsame_ad=''

NetInfo_server = null

如AD与etvuser.adname不一致
	如上报NetInfo不为空
		从t_netinfo读出NetInfo_server, 如果NetInfo_server为空，则new一个，写入上报NetInfo（countnum=0）
		NetInfo_server.countnum+1  (AD不一致次数累加)
		如果上报NetInfo_server和上报NetInfo一致（主要是hgwmac）
			如果NetInfo_server.countnum<= 设置的阈值，则容错允许订购  notsame_type='ADNotSame', notsame_ad='AD,etvuser.adname'
			如果NetInfo_server.countnum > 设置的阈值， 则不允许订购  ifadsame='N',notsame_type='ADNotSame', notsame_ad='AD,etvuser.adname'
		如果上报NetInfo_server和上报NetInfo不一致（主要是hgwmac），则不允许订购  ifadsame='N',notsame_type='ADNotSame', notsame_ad='AD,etvuser.adname'
	如上报NetInfo为空，则不允许订购 ifadsame='N',notsame_type='ADNotSame', notsame_ad='AD,etvuser.adname'
如AD与etvuser.adname一致
	new一个NetInfo_server，写入上报NetInfo（countnum=0）
	如果accesstype=demoAccesstype|CallError|RadiusOff, notsame_type=accesstype, notsame_ad=',etvuser.adname',如果access_type=demoaccesstype, ifdemo='Y'

把NetInfo_server直接入库（先update，结果为0则insert）

把NetSameInfo放入cache，等待后台线程入库

```

## NetSameInfoDao

```
如果notsame_type为空，notsame_count=0
如果notsame_type='ADNotSame',notsame_count+1
```

​    	

## 订购逻辑

```
ifadsame='N'  或者 ifadsame='Y' and updatetime早于两天， 则不允许订购
```

ifadsame	ifdemo	notsame_type	notsame_ad	notsame_count

Y		    N                 null			null			0				AD一致允许订购

Y		    Y                 DemoAccesstype   ‘，etvuser.ad’       原值			   演示机顶盒，AD一致允许订购

Y		   N		 CallError                ‘，etvuser.ad’      原值                           接口调用失败，容错，AD一致允许订购

Y		   N		 RadiusOff              ‘，etvuser.ad’      原值                          接口调用关闭，容错，AD一致允许订购

Y		   N		 ADNotSame           ‘AD，etvuser.ad’   +1                          AD不一致但次数小于阈值，容错，AD一致允许订购

N		   N		 ADNotSame           ‘AD，etvuser.ad’      +1                          AD不一致，不容错，AD不一致不允许订购