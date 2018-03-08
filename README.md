
# 用友U8黑盒开发实现收款单及应收单录入
---
需求：实现自动创建用友U8收款单及应收单。
思路：由于用友U8没有提供API创建，所以只能用过实际操作记录数据库操作，以此来推测相关数据，最终构造格式统统的数据直接入库
---

用友U8用的数据库是SQL Server，监控SQL Server 的执行可以用SQL Server Profiler。具体使用可以自行google。

## 收款单录入
业务表 Ap_CloseBill、Ap_CloseBills
##### 主表插入字段来源：
cVouchType      单据类型编码 48

cVouchID        单据号 0000006826

dVouchDate  	单据时间 2017-12-24

iPeriod     产品结帐状态表GL_Mend应收结账标志（bFlag_AR）会计期间（iPeriod）最大值+1 Select Max(iPeriod) From GL_Mend with(nolock) Where bFlag_AR<>0 and iyear=2017

cDwCode         客户编号 Customer 表 cCusCode 字段

cDeptCode   业务员部门编码 Department 表 cDepCode 字段

cPerson,	业务员编码 Person 表 cPersonCode 字段 

cItem_Class,	项目大类编码  null

cItemCode,	项目编码  null

cSSCode,	结算编号 SettleStyle 表 cSSCode 字段 

cNoteNo,	填入的票据号 

cDigest,	填入的摘要

cBankAccount,	客户银行 账号 null

cexch_name,	充值币种名称 如，美元

iExchRate,	币种汇率

iAmount,	本币金额 （转换本币后金额）

iAmount_f,	原币金额 （原充值金额）

iRAmount,	本币余额

iRAmount_f,	原币余额 

cOperator,	录入人 

bStartFlag,	期初标志 0

cCode,		结算科目编码 （科目编号 1002001004）

cFlag,		应收应付标志 （AR）

iID,		主表标识（ 类似主键 ？）

cBank,		对方银行 null

VT_ID,		单据模版号 ( 8052 来源 select * from VoucherTemplates where VT_ID=8052  )

cItemName,	项目名称 null

iAmount_s,	数量 0

iSource,	来源 null

dcreatesystime,	制单时间 内置函数获取 GETDATE()

csysbarcode	单据条码 由 AA_GeneralBarCode 存储过程 产生 

##### 明细表数据

INSERT INTO Ap_CloseBills

"iID",		Ap_CloseBill主表主键 iID

"ID",		Ap_CloseBills 表主键id

"iType",	款项类型号 0

"bPrePay",	预收预付标志 0

"cCusVen",	客户编码 与主表的 cDwCode 一致

"iAmt_f",	原币金额 $
（原充值金额）所有明细累加必须与主表的总计相等

"iAmt",		本币金额 $  转换本币后金额）

"iRAmt_f",	原币余额 $ （原充值金额）

"iRAmt",	本币余额 $  转换本币后金额）

"cKm",		科目编码 与主表的 cCode 不一样

"cXmClass",	项目大类编码 null

"cXm",		项目编码  null

"cDepCode",	部门编码  与主表的 cDeptCode 一致

"cPersonCode",	业务员编码 与主表的 cPerson 一致

"cOrderID",	订单号  null

"cItemName",	项目名称 null

"cConType",	合同类型  null

"cConID",	合同号  null

"iAmt_s",	数量 0

"iRAmt_s",	数量余额 0

"iOrderType",	来源 null

"cDLCode",	发货单  null

"ccItemCode",	信用证项目 null

"cDefine22",	自定义项22 null

"cDefine23",	自定义项23  null

"cDefine24",

"cDefine25",

"cDefine26",

"cDefine27",

"cDefine28",

"cDefine29",

"cDefine30",

"cDefine31",

"cDefine32",

"cDefine33",

"cDefine34",

"cDefine35",

"cDefine36",

"cDefine37",	自定义项37  null

"cStageCode",	合同阶段 null

"cExecID",	合同执行单号 null

"cMemo",	表体备注 null

"ifaresettled_f",	费用结算金额 $0.00000

"cEBOrderCode"		电商订单号 null

## 流程
- 获取登录账号信息 
SELECT * FROM ufsystem..UA_User WHERE (cUser_Id ='789'  or cuser_name='789' ) and (nstate=0 or nstate is null);

- 根据账号id cUser_Id 取对应的账套 id 即这里对应是  112  和 113 两个账套
select cacc_id from ufsystem..ua_holdauth with(nolock) where cuser_id='789' and iisuser=1  group by cacc_id 
Union All 
select cacc_id from ufsystem..ua_holdauth with (nolock) 
where cUser_id in(select  distinct cgroup_id from ufsystem..ua_role with(nolock)  where cUser_id='789') 
and iIsuser=0 
group by cacc_id;

- 根据 账套id 取账套信息等等
select  cAcc_ID as code,cAcc_name as name ,cUnitAbbre as Abbre,isnull(EnumName,'') as industrytype 
from ufsystem..ua_account with (nolock) 
left join V_AA_enum on EnumType='Admin.Industry' and UA_Account.cIndustryCode=V_AA_enum.EnumCode 
where cAcc_id in('112', '113') 
order by cacc_id;


1. 查询客户（客户编号 0106）

Select cCusCode,cCusAbbName,cCusName,dEndDate,cCusDepart,cCusPPerson,cCusPayCond,cCusOtherProtocol,cCusBank,cCusAccount,cCusSSCode,cCusExch_name,bShop,bOnGPinStore From Customer with(nolock) Where cCusCode=N'0106' or cCusName=N'0106' or cCusAbbName=N'0106' or ccusmnemcode=N'0106' order by case cCusCode when N'0106' then 0 else 1 end

2. 查询业务员（编号 0401）

Select cPersonName,cPersonCode,cDepCode,dPInValidDate From Person with(nolock) Where cPersonCode=N'0401' or cPersonName=N'0401'

3. 产品结帐状态表 应收结账标志（bFlag_AR） ，会计期间 （iPeriod）最大值

Select Max(iPeriod) From GL_Mend with(nolock) Where bFlag_AR<>0 and iyear=2017

4. 查询对应科目（科目编号 1002001004）

Select * From Code with(nolock) Where iYear = 2017 and (ccode=N'1002001004' or ccode_name=N'1002001004' or chelp=N'1002001004')

5. 查询结算方式（结算编号 201）

Select cSSName,cSSCode From SettleStyle with(nolock) Where cSSCode=N'201' or cSSName=N'201'
Select * From SettleStyle with(nolock) Where cSSCode=N'201'

6. 查询账套信息（系统/产品id GL 或者 AA）帐套名称（ cname ）

Select cValue,cName From Accinformation with(nolock) Where cSysID='GL' or cSysID=N'AA';
Select cValue,cName From Accinformation_Year with(nolock) Where (cSysID='GL' or cSysID=N'AA') and cname =N'cGradeLevel' and iyear=2017;
Select cValue,cName From Accinformation_Year with(nolock) Where (cSysID='GL' or cSysID=N'AA') and cname = N'iPlanCtlStl'  and iyear=2017;
Select cValue,cName From Accinformation_Year with(nolock) Where (cSysID='GL' or cSysID=N'AA') and cname like N'bcDefineAss%'  and iyear=2017;

7. 用户自定义档案字段设置 表编号 coderemark

select * from AutoSetFieldInf Where isDefined = 0 and cTableCode =N'coderemark' order by AutoID


8. 查询币种转换、、

select * from foreigncurrency where cexch_name=N'英镑'
select b.cexch_code,b.cexch_name from foreigncurrency b where   b.cexch_name = '美元'

-- 折算方式 bCal

Select bCal From foreigncurrency with(nolock) Where cExch_Name=N'美元'


9. 查询部门 部门编码 或者 部门名称 04

Select cDepCode,cDepName,bDepEnd From Department with(nolock) Where cDepCode=N'04' or cDepName=N'04'


10. 获得唯一的ID码

declare @p5 int

set @p5=1000010748

declare @p6 int

set @p6=1000010752

exec sp_GetID '','113','SK',1,@p5 output,@p6 output

select @p5, @p6


11. 查询对应科目（科目编号 1122001 ）

Select * From Code with(nolock) Where iYear = 2017 and (ccode=N'1122001' or ccode_name=N'1122001' or chelp=N'1122001')

12. 应收应付单据类型表 应收应付标志 cFlag ，单据类型 cTypeCode

Select cTypeName,cTypeCode From Ap_VouchType with(nolock)  Where (cTypeCode=N'48' Or cTypeName=N'48') And cFlag LIKE N'%R%'

13. 查询收付款单主表标识 （ 1000010748 ） ，单据编号 0000006826 ，单据类型编码 48   应收应付标志 AR

Select iID From Ap_CloseBill WITH (UPDLOCK) Where cVouchType=N'48' and cVouchID=N'0000006826' and cFlag=N'AR'

14. 单据编号规则设置表 单据类型编码 RR （单据名称 CardName，收款单）

Select * From VoucherNumber Where CardNumber='RR'

Select * from VoucherPrefabricateview Where CardNumber='RR'


15. 查询单据流水号历史表 最大已使用流水号（ 6825 ），单据类型编码 RR

select cNumber as Maxnumber From VoucherHistory  with (ROWLOCK)  Where  CardNumber='RR' and cContent is NULL

update VoucherHistory set cNumber='6826' Where  CardNumber='RR' and cContent is NULL

---
## 保存

16. 插入前执行的。。 0000006826 为单据号 单据类型编码 48   应收应付标志 AR

declare @p6 nvarchar(5)

set @p6=N'|'

declare @p7 nvarchar(200)

set @p7=N'||ar48|0000006826'

exec AA_GeneralBarCode N'113',N'AR48',N'',N'',N'0000006826',@p6 output,@p7 output

select @p6, @p7

17. 插入主表及明细表
18. 更新 客户 应收余额 

UPDATE SA_CreditSum
SET farsum = isnull(farsum, 0) + isnull(Q.iamount, 0)
FROM ( SELECT cdwcode,Sum(isnull(iamount, 0)) AS iamount FROM #creditTmp GROUP BY cdwcode ) Q
WHERE SA_CreditSum.itype = 1 AND SA_CreditSum.ccuscode = Q.cdwcode;

19. 更新 业务员 应收余额 

UPDATE SA_CreditSum
SET farsum = isnull(farsum, 0) + isnull(Q.iamount, 0)
FROM (SELECT cPerson,Sum(isnull(iamount, 0)) AS iamount FROM #creditTmp GROUP BY cPerson ) Q
WHERE SA_CreditSum.itype = 2 AND SA_CreditSum.ccuscode = Q.cPerson;

20. 更新 业务员所在部门/部门 应收余额 

UPDATE SA_CreditSum
SET farsum = isnull(farsum, 0) + isnull(Q.iamount, 0)
FROM ( SELECT cDeptCode,Sum(isnull(iamount, 0)) AS iamount FROM #creditTmp GROUP BY cDeptCode ) Q
WHERE SA_CreditSum.itype = 3 AND SA_CreditSum.ccuscode = Q.cDeptCode;


---
## PHP代码实现（片段）
```
$db = new SqlSrvPdoConnection($pdo);
            $db->beginTransaction();

            try {

                //取对应的 U8 账套信息
                $platFormAppConfigRow = PlatformAppConfig::where('app_code',$this->appcode)->where('platform_code','U8')->where('status',1)->first();
                if(!$platFormAppConfigRow||!$platFormAppConfigRow->app_code){
                    $result['errorcode'] = '0x00170009';//属性配置异常
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }

                $cUser_Id = $platFormAppConfigRow->cuser_id; // 操作员编码
                if(empty($cUser_Id)){
                    $result['errorcode'] = '0x00170009';//属性配置异常
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                $cAcc_Name = $platFormAppConfigRow->cacc_name; // 账套名称
                if(empty($cAcc_Name)){
                    $result['errorcode'] = '0x00170009';//属性配置异常
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }

                //查询账号 得到用户名，并赋值给 编辑人 cOperator 字段
                $cOperator = $db->query("select cUser_Name from Ua_user where cUser_Id=:cUser_Id", [":cUser_Id" => $cUser_Id])->queryOne();
                $cOperator = $cOperator['cUser_Name'] ?? '';

                //查询账套信息，得到账套 cAcc_Id 编号
                $accountRow = $db->query("select * from UFSystem..UA_Account where cAcc_Name=:cAcc_Name", [":cAcc_Name" => $cAcc_Name])->queryOne();
                if (empty($accountRow) || empty($accountRow['cAcc_Id'])) {
                    $result['errorcode'] = '0x00170006';//UA_Account 账套不存在
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                $cAcc_Id = $accountRow['cAcc_Id'];

                $this->appcode = $platFormAppConfigRow->app_id;//这里将code转成id来查询

                //验证金额
                $checkiAmountRs = $this->checkiAmount($this->input['iAmount'],$this->input['cexch_name'],$this->input['items']);
                if($checkiAmountRs['ret']!=0){
                    $result = $checkiAmountRs;
                    $db->rollBack();
                    break;
                }

                //进行币种属性的转换
                $cexch_code = $this->input['cexch_name'];
                $currencyValue = $commonProcess->findAttrbuteValue(U8CommonProcess::CATEGORY_CURRENCY,$this->appcode,$this->input['cexch_name']);
                if(empty($currencyValue)){
                    $result['errorcode'] = '0x00171011';//币种不合法
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                $this->input['cexch_name'] = $currencyValue;
                //客户的转换
                $relationCustomer = $commonProcess->getRelation(U8CommonProcess::RELATIONSTYPE_CUSTOMER,$this->appcode,$this->input['cDwCode']);
                if($relationCustomer['ret']!=0){
                    $result = $relationCustomer;
                    $db->rollBack();
                    break;
                }
                //业务员的转换
                $relationSaleMan = $commonProcess->getRelation(U8CommonProcess::RELATIONSTYPE_SALESMAN,$this->appcode,$this->input['cPerson']);
                if($relationSaleMan['ret']!=0){
                    $result = $relationSaleMan;
                    $db->rollBack();
                    break;
                }

                $cardName = '收款单';
                //收款单科目
                $keys = null;
                $keys = $commonProcess->getCodeByCurrency($cardName,$currencyValue);
                if(is_null($keys)){
                    $result['errorcode'] = '0x00171012';//未配置币种对应的应收账款科目
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                if(is_null($keys)){
                    $result['errorcode'] = '0x0017000b';//未配置币种对应的收款科目
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                $attributeValueIncome = $commonProcess->findAttrbuteValue(U8CommonProcess::CATEGORY_INCOME,$this->appcode,$keys);
                if(empty($attributeValueIncome)){
                    $result['errorcode'] = '0x0017000b';//未配置币种对应的收款科目
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }

                //获取汇率
                $exch = $commonProcess->getExch($pdo,$this->input['cexch_name'],$this->input['dVouchDate']);

                //查询 应收款管理 收款单 对应的 CardNumber（用于从 VoucherHistory 表获取cNumber）、GlideLen（编号长度？一般是10）、iGlideStep（单号步长 一般是1）、cSub_id（应收应付标志 这里是 AR ）、bRdFlag（ 0 ）
                $voucherNumberRow = $db->query("select CardName, CardNumber, GlideLen, cSub_id, iGlideStep, iStartNumber, bRdFlag from VoucherNumber where AppName=:AppName and CardName=:CardName", [":AppName" => '应收款管理', ':CardName' => $cardName])->queryOne();
                if (empty($voucherNumberRow)) {
                    $result['errorcode'] = '0x00170005';
                    $result['msg'] = __('logic.' . $result['errorcode']); //收款单 VoucherNumber 数据不存在
                    $db->rollBack();
                    break;
                }
                $cSub_id = $voucherNumberRow['cSub_id']; // AR

                //查询单据类型编码 （这里是 48）
                $vouchTypeRow = $db->query("select * from  Ap_VouchType where ctypename=:ctypename AND cflag=:cflag", [":ctypename" => '收款单', ':cflag' => 'RP'])->queryOne();
                if (empty($vouchTypeRow)) {
                    $result['errorcode'] = '0x0017000a';//收款单 Ap_VouchType 数据不存在
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                $ctypecode = $vouchTypeRow['ctypecode']; // 48

                //获取最大单据号
                $voucherHistoryMaxNumber = 1;
                $voucherHistoryRow = $db->query("select cNumber From VoucherHistory  with (ROWLOCK)  Where  CardNumber=:CardNumber and cContent is NULL", [":CardNumber" => $voucherNumberRow['CardNumber']])->queryOne();
                if (!empty($voucherHistoryRow['cNumber'])) {
                    $voucherHistoryMaxNumber = $voucherHistoryRow['cNumber'] + 1;
                }
                //更新最大单据号
                if (!$db->update('VoucherHistory', ['cNumber' => $voucherHistoryMaxNumber], 'CardNumber=:CardNumber AND cContent is NULL', [":CardNumber" => $voucherNumberRow['CardNumber']])->execute()) {
                    $result['errorcode'] = '0x00170008';//更新收款最大单据号失败
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }

                //模板代码
                $VT_CardNumber = $db->query("select VT_CardNumber, VT_ID from vouchertemplates where vt_name like '%应收收款单显示%'")->queryOne();
                if (!empty($VT_CardNumber)) {
                    $cardNumber = $VT_CardNumber['VT_CardNumber'];
                }

                $iPeriodRow = $db->query("Select Max(iPeriod) as iPeriod From GL_Mend with(nolock) Where bFlag_AR<>0 and iyear=:iyear", [':iyear' => date('Y')])->queryOne();
                if (!empty($iPeriodRow)) {
                    $iPeriod = $iPeriodRow['iPeriod'] + 1;
                }


                $spIdArr = $commonProcess->sp_GetID($pdo, $cAcc_Id);
                if (empty($spIdArr) || empty($spIdArr['iFatherId']) || empty($spIdArr['iChildId'])) {
                    $result['errorcode'] = '0x00170001';//执行 sp_GetID 失败
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $result['errorArr'] = $errors;
                    $db->rollBack();
                    break;
                }

                $cardNumber = $cardNumber ?? 'AR48';
                $voucherHistoryMaxNumber = sprintf("%0{$voucherNumberRow['GlideLen']}s", $voucherHistoryMaxNumber);
                $barCodeArr = $commonProcess->AA_GeneralBarCode($pdo, $cAcc_Id, $cardNumber, $voucherHistoryMaxNumber);
                if (empty($barCodeArr) || empty($barCodeArr['splitStr']) || empty($barCodeArr['barCode'])) {
                    $result['errorcode'] = '0x00170002';//执行 AA_GeneralBarCode 失败
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }

                $cSSCode = $commonProcess->getSettleStyle($pdo, "网银");//收款单结算默认为网银结算
                $cSSCode = $cSSCode ?? '201';

                $cDigest = $commonProcess->apCloseBillDigest($app_code, $this->input['dVouchDate'], $attributeValueIncome, $relationCustomer['data'], $this->input['iAmount'], $cexch_code);

                $closeBillRow = [
                    'cVouchType' => $ctypecode,                     // 单据类型编码 48
                    'cVouchID' => $voucherHistoryMaxNumber,         // 单据号 0000006826
                    'dVouchDate' => $this->input['dVouchDate'],                  // 单据时间 2017-12-24
                    'iPeriod' => $iPeriod ?? 1,                     // 产品结帐状态表 GL_Mend 应收结账标志（bFlag_AR） ，会计期间 （iPeriod）最大值+1 《 Select Max(iPeriod) From GL_Mend with(nolock) Where bFlag_AR<>0 and iyear=2017 》
                    'cDwCode' => $relationCustomer['data'],                            // 客户编号 Customer 表 cCusCode 字段
                    'cDeptCode' => $platFormAppConfigRow['department_code'],                            // 业务员部门编码 Department 表 cDepCode 字段
                    'cPerson' => $relationSaleMan['data'],                            // 业务员编码 Person 表 cPersonCode 字段
                    'cItem_Class' => null,                          // 项目大类编码  null
                    'cItemCode' => null,                            // 项目编码  null
                    'cSSCode' => $cSSCode,                             // 结算编号 SettleStyle 表 cSSCode 字段
                    'cNoteNo' => $this->input['cNoteNo']??'',        // 填入的票据号
                    'cDigest' => $cDigest??'', // 填入的摘要
                    'cBankAccount' => null,                         // 客户银行 账号 null
                    'cexch_name' => $this->input['cexch_name']??'',                          // 充值币种名称 如，美元
                    'iExchRate' => $exch,                          // 币种汇率
                    'iAmount' => bcmul($this->input['iAmount'],$exch,4),                          // 本币金额 （转换本币后金额）
                    'iAmount_f' => $this->input['iAmount'],                        // 原币金额 （原充值金额）
                    'iRAmount' => bcmul($this->input['iAmount'],$exch,4),                         // 本币余额
                    'iRAmount_f' => $this->input['iAmount'],                       // 原币余额
                    'cOperator' => $cOperator,                      // 录入人
                    'bStartFlag' => 0,                              // 期初标志 0
                    'cCode' => $attributeValueIncome,                        // 结算科目编码 （科目编号 1002001004）
                    'cFlag' => $cSub_id,                            // 应收应付标志 （AR）
                    'iID' => $spIdArr['iFatherId'],                 // 主表标识（ 来源于 sp_GetID ）
                    'cBank' => null,                                // 对方银行 null
                    'VT_ID' => $VT_CardNumber['VT_ID'] ?? 8052,     // 单据模版号 ( 8052 来源 select * from VoucherTemplates where VT_ID=8052  )
                    'cItemName' => null,                            // 项目名称 null
                    'iAmount_s' => 0,                               // 数量 0
                    'iSource' => null,                              // 来源 null
                    'dcreatesystime' => $commonProcess->SqlSrv_GETDATE(),    // 制单时间 内置函数获取 GETDATE()
                    'csysbarcode' => $barCodeArr['barCode'],        // 单据条码 由 AA_GeneralBarCode 存储过程 产生
                ];

                $closeBillsRows = [];
                foreach ($this->input['items'] as $item){
                    $attributeValueIncome = $commonProcess->findAttrbuteValue(U8CommonProcess::CATEGORY_SETTLEMENT,$this->appcode,$item['cCode']);
                    if(empty($attributeValueIncome)){
                        $result['errorcode'] = '0x00170007';//未配置币种对应的收款科目
                        $result['msg'] = __('logic.' . $result['errorcode']);
                        $db->rollBack();
                        break 2;
                    }

                    $_closeBillsRows = [
                        "iID" => $spIdArr['iFatherId'],        //Ap_CloseBill主表主键 iID
                        "ID" => $spIdArr['iChildId'],         //		Ap_CloseBills 表主键id
                        "cCusVen" => $closeBillRow['cDwCode'],    //	客户编码 与主表的 cDwCode 一致
                        "iAmt_f" => $item['iAmount'],     //	原币金额 $ （原充值金额）所有明细累加必须与主表的总计相等
                        "iAmt" => bcmul($item['iAmount'],$exch,4),       //		本币金额 $  转换本币后金额）
                        "iRAmt_f" => $item['iAmount'],    //	原币余额 $ （原充值金额）
                        "iRAmt" => bcmul($item['iAmount'],$exch,4),      //	本币余额 $  转换本币后金额）
                        "cKm" => $attributeValueIncome,        //		科目编码 与主表的 cCode 不一样 1122001
                        "cDepCode" => $closeBillRow['cDeptCode'],   //	部门编码  与主表的 cDeptCode 一致
                        "cPersonCode" => $closeBillRow['cPerson'],//	业务员编码 与主表的 cPerson 一致
                        "ifaresettled_f" => 0.00000, //	费用结算金额 $0.00000
                        "iAmt_s" => 0,     //	数量 0
                        "iRAmt_s" => 0,    //	数量余额 0
                        "iType" => 0,      //	款项类型号 0
                        "bPrePay" => 0,    //	预收预付标志 0
                        "cXmClass" => null,   //	项目大类编码 null
                        "cXm" => null,        //		项目编码  null
                        "cOrderID" => null,   //	订单号  null
                        "cItemName" => null,  //	项目名称 null
                        "cConType" => null,   //	合同类型  null
                        "cConID" => null,     //	合同号  null
                        "iOrderType" => null, //	来源 null
                        "cDLCode" => null,    //	发货单  null
                        "ccItemCode" => null, //	信用证项目 null
                        "cDefine22" => null,  //	自定义项22 null
                        "cDefine23" => null,  //	自定义项23  null
                        "cDefine24" => null,  //
                        "cDefine25" => null,  //
                        "cDefine26" => null,
                        "cDefine27" => null,
                        "cDefine28" => null,
                        "cDefine29" => null,
                        "cDefine30" => null,
                        "cDefine31" => null,
                        "cDefine32" => null,
                        "cDefine33" => null,
                        "cDefine34" => null,
                        "cDefine35" => null,
                        "cDefine36" => null,
                        "cDefine37" => null,  //	自定义项37  null
                        "cStageCode" => null, //	合同阶段 null
                        "cExecID" => null,    //	合同执行单号 null
                        "cMemo" => null,      //	表体备注 null
                        "cEBOrderCode" => null,   //		电商订单号 null
                    ];

                    $closeBillsRows[] = $_closeBillsRows;

                }

                if (!$db->insert('Ap_CloseBill', $closeBillRow)) {
                    $result['errorcode'] = '0x00170003';// 收款单数据插入失败
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }
                if (count($closeBillsRows) != $db->batchInsert('Ap_CloseBills', $closeBillsRows)) {
                    $result['errorcode'] = '0x00170004';// 收款单明细数据插入失败
                    $result['msg'] = __('logic.' . $result['errorcode']);
                    $db->rollBack();
                    break;
                }

                //更新 客户 信用余额
                $db->getPdo()->setAttribute(constant('PDO::SQLSRV_ATTR_DIRECT_QUERY'), true);
                $sql = "if object_id('tempdb..#creditTmp') is not null drop table #creditTmp";
                $db->query($sql)->execute();
                $sql = "Select sum(case when cVouchType='{$ctypecode}' then -iRAmt else iRAmt end) As iAmount,isnull(cCusVen,'') AS cDwCode,isnull(cDepCode,'') AS cDeptCode,isnull(cPersonCode,'') AS cPerson, 0 as bVouchZZ
                        into #creditTmp
                        From Ap_CloseBills inner join ap_closebill on ap_closebills.iid=ap_closebill.iid
                        Where isnull(cCancelNo,'') not like 'XJ%' and iType<2
                        And ap_closebills.iid={$spIdArr['iFatherId']} Group by cCusVen,cDepCode,cPersonCode";
                $rowCount = $db->query($sql)->execute();
                if($rowCount) {
                    // 更新 客户 应收余额
                    $updateSql = "UPDATE SA_CreditSum SET farsum = isnull(farsum, 0) + isnull(Q.iamount, 0)
                                FROM ( SELECT cdwcode,Sum(isnull(iamount, 0)) AS iamount FROM #creditTmp GROUP BY cdwcode ) Q
                                WHERE SA_CreditSum.itype = 1 AND SA_CreditSum.ccuscode = Q.cdwcode";
                    $rowCount = $db->query($updateSql)->execute();

                    //更新 业务员 应收余额
                    $updateSql = "UPDATE SA_CreditSum
                                SET farsum = isnull(farsum, 0) + isnull(Q.iamount, 0)
                                FROM (SELECT cPerson,Sum(isnull(iamount, 0)) AS iamount FROM #creditTmp GROUP BY cPerson ) Q
                                WHERE SA_CreditSum.itype = 2 AND SA_CreditSum.ccuscode = Q.cPerson";
                    $rowCount = $db->query($updateSql)->execute();

                    //更新 业务员所在部门/部门 应收余额
                    $updateSql = "UPDATE SA_CreditSum
                                SET farsum = isnull(farsum, 0) + isnull(Q.iamount, 0)
                                FROM ( SELECT cDeptCode,Sum(isnull(iamount, 0)) AS iamount FROM #creditTmp GROUP BY cDeptCode ) Q
                                WHERE SA_CreditSum.itype = 3 AND SA_CreditSum.ccuscode = Q.cDeptCode";
                    $rowCount = $db->query($updateSql)->execute();

                }

                $result['data'] = [
                    'iID' => $closeBillRow['iID'],
                    'cVouchID' => $voucherHistoryMaxNumber,
                ];
                $result['ret'] = 0;
                $result['msg'] = "Success";
            } catch (\Exception $e) {
                $db->rollBack();
                $result['errorcode'] = '0x00999999';
                $result['msg'] = __('logic.' . $result['errorcode']);
                Common::mongoLog($e);
            } catch (\Error $e) {
                $db->rollBack();
                $result['errorcode'] = '0x00999999';
                $result['msg'] = __('logic.' . $result['errorcode']);
                Common::mongoLog($e);
            }

```


