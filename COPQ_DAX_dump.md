# DAX in COPQ_Report_r1_local.pbix

## Measures (38)

### [Repulp] Repulp %
```DAX
DIVIDE(SUM(Repulp[Slab Qty Tons]), CALCULATE(sum(Production[Net Production]),FILTER(all(Production),Production[Machine] in VALUES(Repulp[MACH_ID]))))*100
```

### [Stocklot] Stocklot %
```DAX
DIVIDE(SUM(Stocklot[Hold Qty Tons]), CALCULATE(sum(Production[Net Production]),FILTER(all(Production),Production[Machine] in VALUES(Stocklot[PROD_MACH_ID]))))*100
```

### [Receipt_Summary] Excess Manufacturing %
```DAX
DIVIDE(SUM(Receipt_Summary[Excess due to manufacturing]), CALCULATE(sum(Production[Net Production]),FILTER(all(Production),Production[Machine] in VALUES(Receipt_Summary[Machine]))))*100
```

### [Receipt_Summary] J Del Tons
```DAX
sum(Receipt_Summary[Sum of Wgt scaled - Hold])/1000
```

### [TrimSheet_v1] Trim Plan (Tons)
```DAX
sum(TrimSheet_v1[WGT_PLAN])/1000
```

### [TrimSheet_v1] Wound (Tons)
```DAX
sum(TrimSheet_v1[WGT_PROD])/1000
```

### [CoPQ_cat_Qty] Salable x Jumbo Diversions
```DAX
SUM('CoPQ_cat_Qty'[Jumbo Diversions])/SUM('CoPQ_cat_Qty'[Salable])
```

### [CoPQ_cat_Qty] CoPQ_jumbo_diversion_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[Jumbo Diversions])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[Jumbo Diversions]),(sum(CoPQ_cat_Qty[Jumbo Diversions])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[Jumbo Diversions]*RELATED(CoPQ_Cost[Jumbo Diversions])),sum(CoPQ_cat_Qty[Jumbo Diversions])))),0)
```

### [CoPQ_cat_Qty] CoPQ_jumbo_transition_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[Transition])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[Transition]),(sum(CoPQ_cat_Qty[Transition])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[Transition]*RELATED(CoPQ_Cost[Transition])),sum(CoPQ_cat_Qty[Transition])))),0)
```

### [CoPQ_cat_Qty] CoPQ_FH_repulp_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[Finishing loss repulp quantity])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[Finishing loss repulp quantity]),(sum(CoPQ_cat_Qty[Finishing loss repulp quantity])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[Finishing loss repulp quantity]*RELATED(CoPQ_Cost[Finishing loss repulp quantity])),sum(CoPQ_cat_Qty[Finishing loss repulp quantity])))),0)
```

### [CoPQ_cat_Qty] CoPQ_oss_qlt_repulp_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[OSS- Quality Repulp qty])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[OSS- Quality Repulp qty]),(sum(CoPQ_cat_Qty[OSS- Quality Repulp qty])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[OSS- Quality Repulp qty]*RELATED(CoPQ_Cost[OSS- Quality Repulp qty])),sum(CoPQ_cat_Qty[OSS- Quality Repulp qty])))),0)
```

### [CoPQ_cat_Qty] CoPQ_qsc_qlt_repulp_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[QSC- Quality Repulp qty])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[QSC- Quality Repulp qty]),(sum(CoPQ_cat_Qty[QSC- Quality Repulp qty])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[QSC- Quality Repulp qty]*RELATED(CoPQ_Cost[QSC- Quality Repulp qty])),sum(CoPQ_cat_Qty[QSC- Quality Repulp qty])))),0)
```

### [CoPQ_cat_Qty] CoPQ_stocklot_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[CoPQ_stocklot_div])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[Off grade Generation]),(sum(CoPQ_cat_Qty[CoPQ_stocklot_div])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[CoPQ_stocklot_div]*RELATED(CoPQ_Cost[Off grade Generation])),sum(CoPQ_cat_Qty[CoPQ_stocklot_div])))),0)
```

### [CoPQ_cat_Qty] CoPQ_repulp_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[CoPQ_repulp_div])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[Machine stage Repulp- Overall]),(sum(CoPQ_cat_Qty[CoPQ_repulp_div])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[CoPQ_repulp_div]*RELATED(CoPQ_Cost[Machine stage Repulp- Overall])),sum(CoPQ_cat_Qty[CoPQ_repulp_div])))),0)
```

### [CoPQ_cat_Qty] CoPQ_excess_rs/T
```DAX
IFERROR(if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[CoPQ_excess_div])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ])))* sum(CoPQ_Cost[Excess Deliveries]),(sum(CoPQ_cat_Qty[CoPQ_excess_div])/ sum(CoPQ_cat_Qty[Salable]))*(DIVIDE(sumx(CoPQ_cat_Qty,CoPQ_cat_Qty[CoPQ_excess_div]*RELATED(CoPQ_Cost[Excess Deliveries])),sum(CoPQ_cat_Qty[CoPQ_excess_div])))),0)
```

### [Receipt_Summary] Excess Trim
```DAX
sum(Receipt_Summary[Excess due to Trimqty])
```

### [Receipt_Summary] Excess wound over Trim Qty
```DAX
sum(Receipt_Summary[Excess due to Wound qty])
```

### [Receipt_Summary] Excess Wound over Pattern Qty
```DAX
sum(Receipt_Summary[Excess due to Pat qty])
```

### [Receipt_Summary] Excess_Stocklot
```DAX
sum(Receipt_Summary[Excess due to Stocklot])
```

### [Receipt_Summary] Excess_Inventory Transfer
```DAX
sum(Receipt_Summary[Excess due to Inventory Transfer])
```

### [Receipt_Summary] Excess_Weight/Dia variation
```DAX
sum(Receipt_Summary[Excess due to Weight/Dia variation])
```

### [CoPQ_cat_Qty] Copq_FH loss qty
```DAX
sum(CoPQ_cat_Qty[Finishing loss repulp quantity])
```

### [CoPQ_cat_Qty] Copq_jumbo diversio qty
```DAX
sum(CoPQ_cat_Qty[Jumbo Diversions])
```

### [CoPQ_cat_Qty] Copq_jumbo transistin qty
```DAX
sum(CoPQ_cat_Qty[Transition])
```

### [CoPQ_cat_Qty] Copq_oss_qlty_repulp qty
```DAX
sum(CoPQ_cat_Qty[OSS- Quality Repulp qty])
```

### [CoPQ_cat_Qty] Copq_qsc_qlty_repulp qty
```DAX
sum(CoPQ_cat_Qty[QSC- Quality Repulp qty])
```

### [CoPQ_cat_Qty] Copq_excess_del_qty
```DAX
sum(CoPQ_cat_Qty[CoPQ_excess_div])
```

### [CoPQ_cat_Qty] Copq_repulp_qty
```DAX
sum(CoPQ_cat_Qty[CoPQ_repulp_div])
```

### [CoPQ_cat_Qty] Copq_stocklot_qty
```DAX
sum(CoPQ_cat_Qty[CoPQ_stocklot_div])
```

### [TrimSheet_v1] Weight per Unit
```DAX
TrimSheet_v1[Trim Plan (Tons)]/SUM(TrimSheet_v1[UNITS_PLAN])
```

### [CoPQ Cat filter] CoPQ Nov23_Mar24
```DAX
CALCULATE(CoPQ_cat_Qty[CoPQ_excess_rs/T]+CoPQ_cat_Qty[CoPQ_FH_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_jumbo_diversion_rs/T]+CoPQ_cat_Qty[CoPQ_jumbo_transition_rs/T]+CoPQ_cat_Qty[CoPQ_oss_qlt_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_qsc_qlt_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_stocklot_rs/T], Filter( all(CoPQ_cat_Qty[Month]),CoPQ_cat_Qty[Month] >= Date(2023,11,1) && CoPQ_cat_Qty[Month] <= Date(2024,3,1)))
```

### [CoPQ Cat filter] Contribution(Cr)
```DAX
'CoPQ Cat filter'[CoPQ Nov23_Mar24]*CALCULATE(sum(CoPQ_cat_Qty[Salable]))
```

### [CoPQ Cat filter] CoPQ Target
```DAX
'CoPQ Cat filter'[CoPQ Baseline]-('CoPQ Cat filter'[CoPQ Baseline] * 0.20)
```

### [CoPQ Cat filter] CoPQ Baseline
```DAX
CALCULATE(CoPQ_cat_Qty[CoPQ_excess_rs/T]+CoPQ_cat_Qty[CoPQ_FH_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_jumbo_diversion_rs/T]+CoPQ_cat_Qty[CoPQ_jumbo_transition_rs/T]+CoPQ_cat_Qty[CoPQ_oss_qlt_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_qsc_qlt_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_repulp_rs/T]+CoPQ_cat_Qty[CoPQ_stocklot_rs/T], Filter( all(CoPQ_cat_Qty[Month]),CoPQ_cat_Qty[Month] >= Date(2023,4,1) && CoPQ_cat_Qty[Month] <= Date(2023,10,1)))
```

### [Stocklot] Stocklot % Baseline
```DAX
CALCULATE(Stocklot[Stocklot %],Filter( all(Stocklot[Date_Machine]),Stocklot[Date_Machine] >= Date(2023,4,1) && Stocklot[Date_Machine] <= Date(2023,10,31)))
```

### [Repulp] Repulp % Baseline
```DAX
CALCULATE(Repulp[Repulp %],Filter( all(Repulp[REP_DATE]),Repulp[REP_DATE] >= Date(2023,4,1) && Repulp[REP_DATE] <= Date(2023,10,31)))
```

### [Receipt_Summary] Excess Manf % Baseline
```DAX
CALCULATE([Excess Manufacturing %],Filter( all(Receipt_Summary[Date]),Receipt_Summary[Date] >= Date(2023,4,1) && Receipt_Summary[Date] <= Date(2023,10,31)))
```

### [CoPQ_cat_Qty] CoPQ_compensation_rs/T
```DAX
if (HASONEVALUE(CoPQ_cat_Qty[Machine ]),(sum(CoPQ_cat_Qty[Compensation in Rs])/CALCULATE(sum(CoPQ_cat_Qty[Salable]),CoPQ_cat_Qty[Machine ]=VALUES(CoPQ_cat_Qty[Machine ]))),(sum(CoPQ_cat_Qty[Compensation in Rs])/ sum(CoPQ_cat_Qty[Salable])))
```

## Calculated Columns (80)

### [DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8] Year
```DAX
YEAR([Date])
```

### [DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8] MonthNo
```DAX
MONTH([Date])
```

### [DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8] Month
```DAX
FORMAT([Date], "MMMM")
```

### [DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8] Day
```DAX
DAY([Date])
```

### [LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c] Day
```DAX
DAY([Date])
```

### [LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23] Day
```DAX
DAY([Date])
```

### [LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442] Day
```DAX
DAY([Date])
```

### [LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323] Day
```DAX
DAY([Date])
```

### [LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975] Day
```DAX
DAY([Date])
```

### [LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8] Day
```DAX
DAY([Date])
```

### [Repulp] month-year_c
```DAX
Repulp[Month-Year]
```

### [LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814] Day
```DAX
DAY([Date])
```

### [CoPQ_cat_Qty] CoPQ_stocklot_div
```DAX
IFERROR(LOOKUPVALUE(stocklot_summary[Hold],stocklot_summary[Month-Year_Machine],CoPQ_cat_Qty[Month],stocklot_summary[PROD_MACH_ID],CoPQ_cat_Qty[Machine ]),0)
```

### [LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698] Day
```DAX
DAY([Date])
```

### [CoPQ_cat_Qty] CoPQ_repulp_div
```DAX
IFERROR(LOOKUPVALUE(Repulp_summary[Repulp],Repulp_summary[Month-Year],CoPQ_cat_Qty[Month],Repulp_summary[MACH_ID],CoPQ_cat_Qty[Machine ]),0)
```

### [LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae] Day
```DAX
DAY([Date])
```

### [CoPQ_cat_Qty] CoPQ_excess_div
```DAX
IFERROR(LOOKUPVALUE(Excess_summary[Manuf Excess],Excess_summary[Month-Year],CoPQ_cat_Qty[Month],Excess_summary[Machine],CoPQ_cat_Qty[Machine ]),0)
```

### [Receipt_Summary] Export Flag
```DAX
If(Receipt_Summary[Export Sum] > 0,"Export","Domestic")
```

### [LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88] Day
```DAX
DAY([Date])
```

### [Receipt_Summary] Order_70%
```DAX
Receipt_Summary[Order Tolerance]*0.70
```

### [Receipt_Summary] Order_completed_70%
```DAX
Receipt_Summary[Good Scaled]>= Receipt_Summary[Order_70%]
```

### [Receipt_Summary] Order_completion%
```DAX
(Receipt_Summary[Good Scaled]/Receipt_Summary[Order Tolerance])*100
```

### [LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea] Year
```DAX
YEAR([Date])
```

### [LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea] MonthNo
```DAX
MONTH([Date])
```

### [LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea] Month
```DAX
FORMAT([Date], "MMMM")
```

### [LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea] QuarterNo
```DAX
INT(([MonthNo] + 2) / 3)
```

### [LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea] Quarter
```DAX
"Qtr " & [QuarterNo]
```

### [LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea] Day
```DAX
DAY([Date])
```

## Calculated Tables (20)

### DateTableTemplate_99936823-91f9-4708-8050-53543358a4e8
```DAX
Calendar(Date(2015,1,1), Date(2015,1,1))
```

### LocalDateTable_9ce4abd3-940d-46b2-b37a-4ac317ba678c
```DAX
Calendar(Date(Year(MIN('Stocklot'[Date])), 1, 1), Date(Year(MAX('Stocklot'[Date])), 12, 31))
```

### LocalDateTable_ffed7e93-ff4b-4fd8-a44b-037f0bcd9e23
```DAX
Calendar(Date(Year(MIN('Repulp'[REP_DATE])), 1, 1), Date(Year(MAX('Repulp'[REP_DATE])), 12, 31))
```

### LocalDateTable_4ce52da8-48f7-4c89-aedd-66a0bf623442
```DAX
Calendar(Date(Year(MIN('Production'[Month_1])), 1, 1), Date(Year(MAX('Production'[Month_1])), 12, 31))
```

### LocalDateTable_5d00b887-98f9-4945-95a7-d48ed230e323
```DAX
Calendar(Date(Year(MIN('Stocklot'[Date_Machine])), 1, 1), Date(Year(MAX('Stocklot'[Date_Machine])), 12, 31))
```

### LocalDateTable_e5c35c4a-85d5-466b-a45e-396933c96975
```DAX
Calendar(Date(Year(MIN('TrimSheet_v1'[CREATED])), 1, 1), Date(Year(MAX('TrimSheet_v1'[CREATED])), 12, 31))
```

### LocalDateTable_d92fb9d2-12e0-4ae8-84c8-7e18650120b8
```DAX
Calendar(Date(Year(MIN('Production'[Month])), 1, 1), Date(Year(MAX('Production'[Month])), 12, 31))
```

### LocalDateTable_585a7aa3-fb43-49cb-a099-42060de58814
```DAX
Calendar(Date(Year(MIN('CoPQ_Saleable'[Month])), 1, 1), Date(Year(MAX('CoPQ_Saleable'[Month])), 12, 31))
```

### stocklot_summary
```DAX
SUMMARIZE(Stocklot,Stocklot[PROD_MACH_ID],Stocklot[Month-Year_Machine],"Hold",sum(Stocklot[Hold Qty Tons]))
```

### LocalDateTable_3b0c4b78-8b1d-4e4a-a10a-4ae1357a1698
```DAX
Calendar(Date(Year(MIN('stocklot_summary'[Month-Year_Machine])), 1, 1), Date(Year(MAX('stocklot_summary'[Month-Year_Machine])), 12, 31))
```

### Repulp_summary
```DAX
SUMMARIZE(Repulp,Repulp[MACH_ID],Repulp[Month-Year],"Repulp",sum(Repulp[Slab Qty Tons]))
```

### LocalDateTable_fb3dc207-317f-45fd-beee-ba429a4ad4ae
```DAX
Calendar(Date(Year(MIN('Repulp_summary'[Month-Year])), 1, 1), Date(Year(MAX('Repulp_summary'[Month-Year])), 12, 31))
```

### Excess_summary
```DAX
SUMMARIZE(Receipt_Summary,Receipt_Summary[Machine],Receipt_Summary[Month-Year],"Manuf Excess",CALCULATE(sum(Receipt_Summary[Excess due to manufacturing]),Receipt_Summary[Excess Flag] = "Excess",Receipt_Summary[Corp Flag]="Non Corp"))
```

### CoPQ Cat filter
```DAX
{
    ("CoPQ_excess_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_excess_rs/T]), 0),("CoPQ_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_repulp_rs/T]), 1),
    ("CoPQ_stocklot_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_stocklot_rs/T]), 2),
    ("CoPQ_FH_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_FH_repulp_rs/T]), 3),
    ("CoPQ_jumbo_diversion_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_jumbo_diversion_rs/T]), 4),
    ("CoPQ_jumbo_transition_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_jumbo_transition_rs/T]), 5),
    ("CoPQ_oss_qlt_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_oss_qlt_repulp_rs/T]), 6),
    ("CoPQ_qsc_qlt_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_qsc_qlt_repulp_rs/T]), 7)
    
}
```

### Excess Category
```DAX
{
    ("Excess Trim", NAMEOF('Receipt_Summary'[Excess Trim]), 0),
    ("Excess wound over Trim Qty", NAMEOF('Receipt_Summary'[Excess wound over Trim Qty]), 1),
    ("Excess Wound over Pattern Qty", NAMEOF('Receipt_Summary'[Excess Wound over Pattern Qty]), 2),
    ("Excess_Inventory Transfer", NAMEOF('Receipt_Summary'[Excess_Inventory Transfer]), 3),
    ("Excess_Stocklot", NAMEOF('Receipt_Summary'[Excess_Stocklot]), 4),
    ("Excess_Weight/Dia variation", NAMEOF('Receipt_Summary'[Excess_Weight/Dia variation]), 5)
}
```

### CoPQ categories_Qty_filter
```DAX
{
    ("Copq_excess_del_qty", NAMEOF('CoPQ_cat_Qty'[Copq_excess_del_qty]), 0),
    ("Copq_repulp_qty", NAMEOF('CoPQ_cat_Qty'[Copq_repulp_qty]), 1),
    ("Copq_stocklot_qty", NAMEOF('CoPQ_cat_Qty'[Copq_stocklot_qty]), 2),
    ("Copq_FH loss qty", NAMEOF('CoPQ_cat_Qty'[Copq_FH loss qty]), 3),
    ("Copq_jumbo diversio qty", NAMEOF('CoPQ_cat_Qty'[Copq_jumbo diversio qty]), 4),
    ("Copq_jumbo transistin qty", NAMEOF('CoPQ_cat_Qty'[Copq_jumbo transistin qty]), 5),
    ("Copq_oss_qlty_repulp qty", NAMEOF('CoPQ_cat_Qty'[Copq_oss_qlty_repulp qty]), 6),
    ("Copq_qsc_qlty_repulp qty", NAMEOF('CoPQ_cat_Qty'[Copq_qsc_qlty_repulp qty]), 7)
}
```

### CoPQ Cat_Other_Qty_Filter
```DAX
{
    ("Copq_FH loss qty", NAMEOF('CoPQ_cat_Qty'[Copq_FH loss qty]), 0),
    ("Copq_jumbo transistin qty", NAMEOF('CoPQ_cat_Qty'[Copq_jumbo transistin qty]), 1),
    ("Copq_jumbo diversio qty", NAMEOF('CoPQ_cat_Qty'[Copq_jumbo diversio qty]), 2),
    ("Copq_qsc_qlty_repulp qty", NAMEOF('CoPQ_cat_Qty'[Copq_qsc_qlty_repulp qty]), 3),
    ("Copq_oss_qlty_repulp qty", NAMEOF('CoPQ_cat_Qty'[Copq_oss_qlty_repulp qty]), 4)
}
```

### LocalDateTable_6cd1ddd4-fa32-42b3-9172-98985cfc2a88
```DAX
Calendar(Date(Year(MIN('Excess_summary'[Month-Year])), 1, 1), Date(Year(MAX('Excess_summary'[Month-Year])), 12, 31))
```

### CoPQ Cat filter_Compen
```DAX
{
    ("CoPQ_excess_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_excess_rs/T]), 0),
    ("CoPQ_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_repulp_rs/T]), 1),
    ("CoPQ_stocklot_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_stocklot_rs/T]), 2),
    ("CoPQ_jumbo_diversion_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_jumbo_diversion_rs/T]), 3),
    ("CoPQ_jumbo_transition_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_jumbo_transition_rs/T]), 4),
    ("CoPQ_FH_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_FH_repulp_rs/T]), 5),
    ("CoPQ_oss_qlt_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_oss_qlt_repulp_rs/T]), 6),
    ("CoPQ_qsc_qlt_repulp_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_qsc_qlt_repulp_rs/T]), 7),
    ("CoPQ_compensation_rs/T", NAMEOF('CoPQ_cat_Qty'[CoPQ_compensation_rs/T]), 8)
}
```

### LocalDateTable_a30116cf-522d-4ebb-840b-8b90a8c4f8ea
```DAX
Calendar(Date(Year(MIN('Receipt_Summary'[Date])), 1, 1), Date(Year(MAX('Receipt_Summary'[Date])), 12, 31))
```
