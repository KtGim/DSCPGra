# cudQuotationNew.vue 操作 → 函数调用链梳理

- 数据来源：graphify 图（`src/views/productsLib/quotation/components/graphify-out/graph.json`，299 节点 / 614 边，其中 cudQuotationNew 内部调用边 271 条）+ 源码核对（文件共 7825 行）
- 社区归属：括号内为 graphify 社区（社区 = 逻辑模块边界，见 GRAPH_REPORT.md）
- 说明：`[接口]` 表示发出 HTTP 请求；`→` 表示直接调用；`⟲` 表示异步串联（await）

---

## 0. 核心方法索引（被多个操作复用的枢纽）

| 方法 | 社区 | 被谁触发 |
|------|------|----------|
| `handleCalcShippingFee` | handleCalcShippingFee | 国家/数量/币种/重量/渠道/利润率等几乎所有联动 |
| `calcBatchTotalFee` | calcBatchTotalFee | 费用计算枢纽，16 处调用点 |
| `handleTotalFeeUpdate` | calcBatchTotalFee | 打包费/操作费/产品费/续件费等失焦 |
| `getDeclaredPriceByTable` | calcBatchTotalFee | 申报价整表试算 |
| `handleCallByCountry(2)` | handleCalcShippingFee | 国家变化全量/差异流程 |
| `creatTableRows(2)` | ensureNoCountryRow | 运费表格建行 |
| `fetchEuProcessFeeByRows` | calcBatchTotalFee | 欧盟操作费批量回填 |
| `ensureNoCountryRow` | ensureNoCountryRow | 国家为空时的兜底行 |
| `begin/endFormLinkageSaveBlock` | handleCalcShippingFee | 联动期间的保存锁 |
| `begin/endDeclaredPriceApi` | calcBatchTotalFee | 申报价接口并发计数 + 税费列 loading |

---

## 1. 页面初始化（mounted / 路由进入）

```
handler()                          [社区: handler]
├─ getDictData()                   [接口] 字典
├─ getBusinesOptions()             [接口] 业务开发人员
├─ getMerchantOptions()            [接口] 商户
├─ getShipMethodOptions()          [接口] 发货方式
├─ resetModelForm()                ── 新增路径
│   ├─ resetCommonData()
│   └─ ensureNoCountryRow() → createNoCountryRow() → genYfRowKey()
│                             └→ calcNoCountryTotalFee()
├─ getDetailById()                 ── 编辑路径
│   ├─ [接口] getDetailInfoById
│   ├─ 映射 ite.batches = quotationShippingFeeDTOList
│   ├─ syncRowEuProcessCostToBatches(row)
│   ├─ analyze1688ByDetail(url)    ── 解析 1688 属性
│   ├─ refreshYfVirtualScroll() → ensureYfRowKeys() → genYfRowKey()
│   └─ handleCallByCountry(countryCode, true)   ── isDetail 只回填不重建
├─ getPreviewArchiveById()         ── 历史预览路径
│   └─ [接口] previewArchive → applyQuotationDetailPayload(resData)
│       ├─ syncRowEuProcessCostToBatches() / refreshYfVirtualScroll()
│       ├─ analyze1688ByDetail() / handleCallByCountry(x, true)
│       └─ emitArchivePreviewDrawerTitle()
├─ initScrollListener() → addYfTableHScrollListener()   [社区: handler]
├─ bindEscToBack() → handleEscToBack()
└─ (beforeDestroy) unbindEscToBack() + closeYfFilterPanel()
```

---

## 2. 切换报价类型（location radio @change）

```
handleChangeLocation()
├─ beginFormLinkageSaveBlock() → setSaveLinkagePending()
├─ applyLocationDefaults()            [社区: calcBatchTotalFee]
│   ├─ resetToNoCountry()             [社区: handleCalcShippingFee]
│   │   ├─ countryCode=[] / bjyf=[] / costListByCountryWeight=null
│   │   ├─ ensureNoCountryRow() → createNoCountryRow() → calcNoCountryTotalFee()
│   │   └─ refreshYfVirtualScroll() → ensureYfRowKeys()
│   ├─ 【US】applyOverseasWarehouseDefaults()
│   │   ├─ packingFee=0.30 → convertOverseasPackingFeeByCurrency()   [接口] calcPackingFeeApi
│   │   ├─ calcProductFee()            ── 产品费 = (采购价+国内运费)/汇率×(1+利润率)
│   │   ├─ resetAndSelectDefaultCountry({expandPartition:true})
│   │   │   └─ applyDefaultCountryWithPartition('US')
│   │   │       ├─ creatTableRows2(['US']) → getDeclaredPriceByTable2() → getProfitByCountryCode2()
│   │   │       ├─ handleCountryPartition()（分区1-9）
│   │   │       ├─ getMatchByAtt() → handleShippCel()
│   │   │       └─ [接口] getCalcAllByCountry → handleCalcShippingFee()
│   │   └─ handleTotalFeeUpdate() → calcBatchTotalFee()
│   ├─ 【CN+大货】applyDomesticLargeCargoDefaults()
│   │   ├─ packingFee=0 / 利润率=15 → calcProductFee()
│   │   ├─ resetAndSelectDefaultCountry({expandPartition:false}) → handleCallByCountry2ByEnter(['US'])
│   │   └─ handleTotalFeeUpdate()
│   ├─ 【CN非大货】applyDomesticNormalDefaults()
│   │   └─ packingFee=0.5(EUR 0.46) → calcProductFee() → handleTotalFeeUpdate()
│   ├─ calcStepCost()                 ── US 分支
│   └─ fetchEuProcessFeeByRows(bjyf)  [接口] getCountryOperationFee
│       └─ resolveEuProcessFee() → setRowEuProcessCost() → calcBatchTotalFee()
└─ endFormLinkageSaveBlock()
```

---

## 3. 是否大货切换（CN 下 el-switch @change）

```
handleChangeLargeCargo(val)
├─ beginFormLinkageSaveBlock()
├─ 【CN && val=开】applyDomesticLargeCargoDefaults()   （见上，默认 US 不展开分区）
│   └─ fetchEuProcessFeeByRows(bjyf) → setRowEuProcessCost → calcBatchTotalFee
├─ 【关】resetToNoCountry()
│   ├─ 【CN】applyDomesticNormalDefaults() → calcProductFee() → handleTotalFeeUpdate()
│   └─ 【非CN】handleTotalFeeUpdate() → calcBatchTotalFee()
└─ endFormLinkageSaveBlock()
```

---

## 4. 切换币种（currency select @change）

```
currencyChange(val)                    [社区: calcBatchTotalFee]
├─ _currencyChangeSeq++（竞态令牌，快照 seq）
├─ setSaveLinkagePending('currency', true)
├─ [接口] getExchangeRateApi → exchangeRate / exchangeRMBRate
├─ convertPackingFeeByCurrency(sourceCurrency=prevCurrency, val, seq)
│   ├─ [接口] calcPackingFeeApi（打包费 USD→目标币种换算）
│   ├─ 成功 → syncPrev() → prevCurrency = 新币种
│   └─ 失败 → 保持 prevCurrency（下次按正确 source 换算）
├─ calcProductFee()
├─ calcStepCost()
├─ getDeclaredPriceByTable()           [接口] getDeclareAmountApi（税费随申报价）
│   ├─ beginDeclaredPriceApi() → setTaxFlgForDeclaredPriceCalc(true)
│   └─ endDeclaredPriceApi()
├─ fetchEuProcessFeeByRows(bjyf)
├─ handleCalcShippingFee()             [接口] getCalcWithCountryQty
│   ├─ changeTPLIdByCalcAll() → calcBatchTotalFee()
│   └─ calcNoCountryTotalFee()
├─ 全表 calcBatchTotalFee()            ── 兜底重算（含未进运费重算集合的行）
└─ setSaveLinkagePending('currency', false)
※ 每个 await 后 `seq !== this._currencyChangeSeq` 则丢弃（防旧请求覆盖新状态）
```

---

## 5. 客户代码变更 / 清空（cusCode @change / @clear）

```
changeCusCode()
├─ setSaveLinkagePending('cusCode', true)
├─ refreshCountryShortcutAvailability()
├─ applyLocationDefaults()             ── 复用报价类型默认值流程（详见 §2）
├─ calcProductFee() / calcStepCost()
├─ getDeclaredPriceByTable()
├─ handleCalcShippingFee()
└─ calcNoCountryTotalFee()

clearCusCode()
└─ setSaveLinkagePending('cusCode', false)
```

---

## 6. 产品数量变更（quantitys 多选 @change）

```
quantityChange(newQuantitys)           [社区: calcBatchTotalFee]
├─ beginFormLinkageSaveBlock()
├─ 重建每行 batches（batchMap 按数量保留旧值）+ 重建 productFees/stepCosts
├─ calcProductFee()
├─ calcStepCost()
├─ fetchEuProcessFeeByRows(bjyf)       [接口] 欧盟操作费按新数量
├─ getDeclaredPriceByTable()           [接口] 申报价按新数量
├─ handleCalcShippingFee()             [接口] 运费按新数量
└─ endFormLinkageSaveBlock()
※ 数量是全局维度：所有国家的行同时按新数量重建
```

---

## 7. 费用相关输入联动

### 7.1 产品利润率（@change）

```
calcChange($event, 'productFeeProfitRate')
├─ beginFormLinkageSaveBlock()
├─ calcProductFee()
├─ handleTotalFeeUpdate() → calcBatchTotalFee()
└─ endFormLinkageSaveBlock()
```

### 7.2 采购价 / 国内运费（@blur）

```
handleDomesticShippingCostsBlur()
├─ beginFormLinkageSaveBlock()
├─ calcProductFee()
├─ handleTotalFeeUpdate() → calcBatchTotalFee()
└─ endFormLinkageSaveBlock()
```

### 7.3 打包费 / 操作费 / 定制费 / 产品费 / 续件费（@blur）

```
handleTotalFeeUpdate(key?, item?)
├─ beginFormLinkageSaveBlock()
├─ 【key='productFees'|'stepCosts' 且 qus===1】级联 ×N 数量档位费用
├─ 遍历 bjyf 全部行全部 batch → calcBatchTotalFee(batch)
└─ endFormLinkageSaveBlock()
```

### 7.4 是否需要税费开关（@change）

```
handleChangeTaxesraquired()
├─ getDeclaredPriceByTable()   ── 税费 = 申报价×税率/汇率（taxesRequired 为 false 时置 0）
└─ calcBatchTotalFee()         [社区: calcBatchTotalFee]
```

---

## 8. 国家选择（countryCode 多选）

### 8.1 下拉展开 / 收起（@visible-change）

```
handleCallByCountry2(flg, val)         [社区: handleCalcShippingFee]
├─ 展开(flg=true)：diableInput=true，缓存 oldCountryCode，直接返回
└─ 收起(flg=false)：防抖 500ms
    ├─ beginFormLinkageSaveBlock()
    ├─ compareArrayChanges(oldCountryCode, countryCode)
    │   ├─ removed → 删除 bjyf 对应行 + 清理 costListByCountryWeight
    │   └─ added  → removeNoCountryRows() → creatTableRows2(added)
    │       ├─ getDeclaredPriceByTable2(added)  [接口] 申报价（增量）
    │       ├─ getProfitByCountryCode2(added)   [接口] 客户国家平均利润率
    │       └─ refreshYfVirtualScroll()
    ├─ getMatchByAtt(added, 'byRow') → handleShippCel() / handleCalcShippingFee()
    ├─ getCountryFqList(added)
    ├─ [接口] getCalcAllByCountry → handleCalcShippingFee()
    └─ endFormLinkageSaveBlock()
```

### 8.2 回车确认国家（@keyup.enter）

```
handleKeyEnter()
├─ beginFormLinkageSaveBlock()
├─ handleCallByCountry2ByEnter(countryCode)
│   ├─ creatTableRows2() → getDeclaredPriceByTable2()
│   ├─ getMatchByAtt(…, 'byWeight') → handleShippCel()
│   ├─ getCountryFqList()
│   ├─ [接口] getCalcAllByCountry
│   └─ handleCalcShippingFee() → calcBatchTotalFee()
└─ endFormLinkageSaveBlock()
```

### 8.3 移除单个国家 / 清空国家（@remove-tag / @clear）

```
handleRemoveCountryCode(val)
├─ 过滤删除 bjyf 对应行 + 删除 costListByCountryWeight
└─ countryCode 清空后 → ensureNoCountryRow() → calcNoCountryTotalFee()

handleClearCountryCode()
├─ bjyf=[]
└─ ensureNoCountryRow() → calcNoCountryTotalFee()
```

### 8.4 快捷勾选国家（上次报价 / 所有报价国家）

```
getLastQuotedCountry() / getAllQuotingCountries()
└─ applyCountryShortcut()              [社区: handleCalcShippingFee]
    ├─ beginFormLinkageSaveBlock()
    ├─ 组装国家列表 → handleCallByCountry2(false, 新列表)   ── 走差异流程
    └─ endFormLinkageSaveBlock()
```

---

## 9. 重量 / 抛重输入失焦（@blur）

```
handleGetFinalFee(weight, field, 'input')
├─ beginFormLinkageSaveBlock()
├─ [接口] 附加重量计算 → packageFinalWeight / volumeFinalWeight
├─ calcTempShippingCosts() → getCountryFqList()
├─ fetchEuProcessFeeByRows(bjyf)
├─ getDeclaredPriceByTable()
├─ removeNoCountryRows()
├─ handleCalcShippingFee()  或  handleCallByCountry(countryCode, false)
└─ endFormLinkageSaveBlock()
※ 实重/抛重取最大值作为试算重量
```

---

## 10. 运费利润率失焦（@blur）

```
handleShippingFeeProfitRateBlur()
├─ 空值/为0的 batch 同步全局 shippingFeeProfitRate
├─ beginFormLinkageSaveBlock() → endFormLinkageSaveBlock()
└─ handleCalcShippingFee() → calcBatchTotalFee() / calcNoCountryTotalFee()
```

---

## 11. 产品属性变更（checkbox @change）

```
handleProductAttrChange()
└─ getMatchByAtt()            [社区: handleCalcShippingFee]
    ├─ [接口] 按属性匹配最优渠道
    └─ handleShippCel() / handleCalcShippingFee()
```

---

## 12. 运费表格内操作（核心交互区）

### 12.1 选择发货方式（行内 el-select @change）

```
handleShippCelChangeFromTable(val, row)
├─ beginFormLinkageSaveBlock()
├─ handleShippCel(val, row)            [社区: calcBatchTotalFee]
│   ├─ ensureShippingChannelInYfFilter(row)
│   ├─ setTableLoading('shipFee','totalFee','loading')
│   ├─ autoFilled 批次 → beginDeclaredPriceApi([row])
│   │   └─ [接口] getDeclareAmountApi（申报价/税费/利润率逐批）
│   │       ├─ batch.shippingFeeProfitRate = 全局值
│   │       └─ endDeclaredPriceApi([row])
│   ├─ declareApiFlg=false 时 → getDeclaredPriceByTable2([row])
│   │   ├─ beginDeclaredPriceApi(targetRows) / endDeclaredPriceApi(targetRows)
│   │   └─ calcBatchTotalFee()
│   ├─ fetchEuProcessFeeByRow(row)     [接口] 单行欧盟操作费
│   │   └─ resolveEuProcessFee() → setRowEuProcessCost() → calcBatchTotalFee()
│   └─ calcBatchTotalFee() + totalFeeFlag
└─ endFormLinkageSaveBlock()
```

### 12.2 清空发货方式（@clear）

```
handleClearShippByRow(row)
└─ calcBatchTotalFee(batch)   ── 清渠道后按 0 运费重算
```

### 12.3 渠道选择器确认（channelSelector @changeSelectorConfirm）

```
changeSelectorConfirm($event, row, index)   [社区: changeSelectorConfirm]
├─ beginFormLinkageSaveBlock()
├─ calcFeeByItemRow(row, itemObj)   ── 写入 shippingFeeYb
├─ handleShippCel()                 ── 同 12.1
├─ simulateDocumentClick()          ── 关闭 popover
└─ endFormLinkageSaveBlock()
```

### 12.4 复制到所有（@click 图标）

```
handleSetshippingChannel(row)       [社区: calcBatchTotalFee]
├─ ensureShippingChannelInYfFilter(row)
├─ fetchEuProcessFeeByRows(bjyf)    ── 渠道变更后批量回填欧盟操作费
└─ handleCalcShippingFee()
```

### 12.5 表格内单元格编辑失焦（editableCell @hdBlur）

```
handleTbRowsBlur(val, row, batchIdx, item)
├─ $set(batch, 字段, val)
├─ getShippingFeeResult()           ── 按申报价反推/校验
├─ calcBatchTotalFee(batch)
└─ calcNoCountryTotalFee()          ── 无国家行

handleTbRowsBlurByRate(...)         ── 利润率格，同上 + handleSetShipfeeYb 反推
handleEuProcessCostBlur(val, row, batchIdx)   ── 欧盟操作费格
└─ calcBatchTotalFee() / calcNoCountryTotalFee()
```

### 12.6 运费一致（popover 内按钮）

```
handleSetShipfeeYb(item, batch, mode)   [社区: changeSelectorConfirm]
├─ 反推 shippingFeeProfitRate = (fee×汇率/原运费−1)×100
├─ handleIsagreeShippingFee()       ── 校验是否一致
├─ simulateDocumentClick()
└─ calcBatchTotalFee()
```

### 12.7 复制行 / 国家分区

```
handleCopyTableRow(row, index)      [社区: ensureNoCountryRow]
├─ JSON 深拷贝行
└─ genYfRowKey()（虚拟滚动 key 唯一）

handleCountryPartition(row, index, opts)
├─ genYfRowKey() × N（分区行）
├─ getCountryFqList()
├─ getMatchByAtt()
├─ mergeCountryPartitionIntoYfFilter()
└─ refreshYfVirtualScroll()
```

---

## 13. 商品信息操作

### 13.1 款式代号失焦 / 组合品刷新（@blur / @click）

```
handleBlurZhp(styleCode)             [社区: fetchCombosDetail]
└─ fetchCombosDetail()
    ├─ [接口] 组合品详情
    └─ applyCombosDetail()
        ├─ ensureNoCountryRow()
        ├─ getCountryFqList()
        ├─ syncRowEuProcessCostToBatches()
        ├─ handleCalcShippingFee()
        └─ refreshYfVirtualScroll()

handleResetCombos() → fetchCombosDetail()
```

### 13.2 1688 链接解析 / 失焦 / 清空

```
analyze1688(url)                     [社区: cudQuotation.vue / calcBatchTotalFee]
├─ [接口] 1688 解析 → 商品信息回填
└─ calcProductFee()

handleProductUrlBlur()               [社区: handleProductUrlBlur]
├─ getUrlBase() / is1688Url()
└─ syncRemoveCurrentProductUrl()

handleProductUrlClear() → syncRemoveCurrentProductUrl()
```

### 13.3 多采购链接管理（popover）

```
addProductUrl()  → getUrlBase()
handleChangeRadio(row)  /  handleRemovePop(row, index)  → is1688Url()
```

---

## 14. 保存 / 返回 / ESC（页面级操作）

### 14.1 保存（@click 保存按钮）

```
save()                               [社区: handleEscToBack]
├─ 校验（表单规则 + bjyf 各批次）
├─ 参数映射：quotationShippingFeeDTOList = batches；countryShippingFees = bjyf
├─ [接口] addOrUpdate → 成功提示
└─ back()                            ── 返回列表
※ saveDisabled = 关键字段聚焦中 || 运费/总费 loading || 联动 pending
```

### 14.2 返回 / ESC

```
back() → 路由返回

handleEscToBack()                    [社区: handleEscToBack]
├─ consumeUpperOverlayOnEsc()
│   ├─ hasVisibleElementOverlay() → isDomElementVisible()
│   ├─ closeVisiblePopoversOnEsc()   [社区: closeVisiblePopoversOnEsc]
│   │   ├─ closePopoverByPopperEl()
│   │   └─ blurDocumentFocus()
│   └─ closeYfFilterPanel()
└─ back()

confirmReplaceWithArchive() → back()（历史报价替换二次确认）
```

---

## 15. 运费表格筛选 / 虚拟滚动 / 吸顶（视图辅助）

```
openYfFilterPanel(col, e) → closeYfFilterPanel()
handleYfFilterConfirm() → closeYfFilterPanel() + refreshYfVirtualScroll()
handleYfFilterReset()   → closeYfFilterPanel() + refreshYfVirtualScroll()
handleScroll()          → updateStickyYfHeader()   [社区: updateStickyYfHeader]
                          ├─ syncStickyColumnWidths()
                          └─ syncStickyHorizontalScroll()
refreshYfVirtualScroll() → ensureYfRowKeys() → genYfRowKey()
```

---

## 16. MOQ 弹窗（产品费项 @click 维护MOQ）

```
openMoqDialog(item, index)           ── 打开
saveMoqData()                        [社区: saveMoqData]
├─ _validateMoqValue()               ── 校验数量/价格
├─ syncMoqRowError() → _validateMoqValue()
└─ cancelMoqDialog()
```

---

## 附：操作 → 最重链路拓扑图（主路径）

```
┌─ 初始化 handler ──────────────→ getDetailById / applyQuotationDetailPayload
│                                     │
│                                     └─ handleCallByCountry(true) ── 只回填不重建
├─ 选国家 ──── handleCallByCountry2 ──→ creatTableRows2 → getDeclaredPriceByTable2
│                │                       → getMatchByAtt → handleShippCel
│                └─ getCalcAllByCountry ─→ handleCalcShippingFee → calcBatchTotalFee
├─ 改数量 ──── quantityChange ───→ calcProductFee → calcStepCost
│                └─ fetchEuProcessFeeByRows ─ getDeclaredPriceByTable ─ handleCalcShippingFee
├─ 切币种 ──── currencyChange ───→ convertPackingFeeByCurrency → calcProductFee
│                └─ getDeclaredPriceByTable → fetchEuProcessFeeByRows
│                └─ handleCalcShippingFee → 全表 calcBatchTotalFee
├─ 切类型 ──── handleChangeLocation → applyLocationDefaults → resetToNoCountry
│                └─ applyOverseasWarehouseDefaults / applyDomesticLargeCargoDefaults
│                └─ resetAndSelectDefaultCountry → applyDefaultCountryWithPartition
├─ 选渠道 ──── handleShippCelChangeFromTable → handleShippCel
│                ├─ beginDeclaredPriceApi → getDeclareAmountApi
│                ├─ getDeclaredPriceByTable2 → calcBatchTotalFee
│                └─ fetchEuProcessFeeByRow → setRowEuProcessCost → calcBatchTotalFee
└─ 保存 ────── save → addOrUpdate → back
```

**核心结论**：所有费用联动最终都收敛到 `calcBatchTotalFee`（总费唯一出口）；所有异步链路都被 `begin/endFormLinkageSaveBlock`（保存锁）与 `begin/endDeclaredPriceApi`（税费列 loading 计数）包住；国家/数量/币种/重量四类操作共享 `handleCalcShippingFee` 这条最重链路。
