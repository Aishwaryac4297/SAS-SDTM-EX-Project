# 💊 SDTM-EX (Exposure) Dataset Creation Guide

This document outlines the SAS programming steps to build a **CDISC-compliant SDTM-EX (Exposure)** dataset. It includes derivation logic, variable formatting, and integration with the Demographics (DM) domain for reference dates.

---

## 📂 Input Files

| File Name                                             | Description              |
|------------------------------------------------------|--------------------------|
| `SAS Project SDTM -EX Raw data.xlsx`                 | Exposure raw dataset     |
| `SAS Project SDTM_Demography Raw data.xlsx`          | Demographic (DM) data    |

---

## 🔧 Step-by-Step SAS Workflow

### ✅ Step 1: Import Exposure Raw Data

```sas
PROC IMPORT DATAFILE="/home/SAS Project SDTM -EX Raw data.xlsx"
    OUT=work.EX DBMS=XLSX REPLACE;
RUN;

DATA EX1;
    SET work.EX;
    IF STUDY = ' ' THEN DELETE;
RUN;
```
### ✅ Step 2: Build Exposure (EX) Domain
```sas
DATA EX1;
SET EX1;
    STUDYID   = STRIP(STUDY);
    DOMAIN    = "EX";
    SITEID    = STRIP(SITE);
    SUBJID    = STRIP(SUBJID);
    USUBJID   = STRIP(STUDYID) || "-" || STRIP(SITEID) || "-" || STRIP(SUBJID);

    EXTRT     = STRIP(TRT);
    EXDOSE    = INPUT(SUBSTR(DOSE,1,2), BEST.);
    EXDOSU    = LOWCASE(SUBSTR(DOSE,3));
    EXDOSTXT  = DOSEN;
    EXDOSFRM  = DOSTP;
    EXDOSFRQ  = UPCASE(FREQ);
    EXROUTE   = UPCASE(ROUTE);
    EXFAST    = UPCASE(SUBSTR(FAST,1,1));
    EPOCH     = "TREATMENT";

    DSDTN     = INPUT(DSDT, MMDDYY10.);
    EXSTDTC   = PUT(DSDTN, IS8601DA.) || "T" || PUT(DSDTM, TOD8.);
    EXSTDTN   = DATEPART(INPUT(EXSTDTC, IS8601DT.));
    FORMAT EXSTDTN DATE9.;

    EXENDTC   = PUT(DSDTN, IS8601DA.) || "T" || PUT(DSDTM, TOD8.);
    EXENDTN   = DATEPART(INPUT(EXENDTC, IS8601DT.));
    FORMAT EXENDTN DATE9.;
RUN;
```
### ✅ Step 3: Import and Prepare Demographics Data for Reference Start Date
```sas
PROC IMPORT DATAFILE="/home/SAS Project SDTM_Demography Raw data.xlsx"
    OUT=WORK.DM DBMS=XLSX REPLACE;
RUN;

DATA DM1;
    SET WORK.DM;
    IF STUDY = ' ' THEN DELETE;
RUN;

DATA DM1;
    SET DM1;
    STUDYID   = STRIP(STUDY);
    DOMIN     = "DM";
    SUBJID    = STRIP(SUBJID);
    SITEID    = STRIP(SITE);
    USUBJID   = STRIP(STUDYID) || "-" || STRIP(SITEID) || "-" || STRIP(SUBJID);
    ENRDTN    = INPUT(ENTDT, MMDDYY10.);
    RFSTDTC   = PUT(ENRDTN, IS8601DA.) || "T" || PUT(ENRTM, TOD8.);
    RFSTDTN   = DATEPART(INPUT(RFSTDTC, IS8601DT.));
    FORMAT RFSTDTN DATE9.;
    KEEP USUBJID RFSTDTN RFSTDTC;
RUN;
```
### ✅ Step 4: Merge Exposure and Demographics
```sas
PROC SORT DATA=EX1; BY USUBJID; RUN;
PROC SORT DATA=DM1; BY USUBJID; RUN;

DATA DM_EX;
    MERGE DM1 (IN=A) EX1 (IN=B);
    BY USUBJID;
    IF A AND B;
RUN;
```
### ✅ Step 5: Derive Study Days and Exposure Duration
```sas
DATA EX2;
    SET DM_EX;

    IF EXSTDTN > . AND RFSTDTN > . THEN DO;
        IF EXSTDTN >= RFSTDTN THEN EXSTDY = EXSTDTN - RFSTDTN + 1;
        ELSE EXSTDY = EXSTDTN - RFSTDTN;
    END;

    IF EXENDTN > . AND RFSTDTN > . THEN DO;
        IF EXENDTN >= RFSTDTN THEN EXENDY = EXENDTN - RFSTDTN + 1;
        ELSE EXENDY = EXENDTN - RFSTDTN;
    END;

    EXDUR1 = EXENDY - EXSTDY + 1;
    EXDUR  = COMPRESS('P' || PUT(EXDUR1, BEST.) || 'D');
RUN;
```
### ✅ Step 6: Assign Sequence Number
```sas
DATA EX3;
    SET EX2;
    BY USUBJID EXSTDTN;
    IF FIRST.USUBJID THEN EXSEQ = 1;
    ELSE EXSEQ + 1;
RUN;
```
### ✅ Step 7: Final EX Dataset with Required Variables
```sas
DATA EX4;
SET EX3;
KEEP 
    STUDYID DOMAIN USUBJID EXSEQ EXTRT EXDOSE EXDOSTXT EXDOSU EXDOSFRM EXDOSFRQ 
    EXROUTE EXFAST EPOCH EXSTDTC EXENDTC EXSTDY EXENDY EXDUR;
RUN;
```
### 📊 Final SDTM-EX Dataset
```sas
PROC SQL;
CREATE TABLE EX_FINAL AS
SELECT
    STUDYID      "Study Identifier"                   LENGTH=8,
    DOMAIN       "Domain Abbreviation"                LENGTH=2,
    USUBJID      "Unique Subject Identifier"          LENGTH=50,
    EXSEQ        "Sequence Number"                    LENGTH=8,
    EXTRT        "Name of Treatment"                  LENGTH=200,
    EXDOSE       "Dose"                               LENGTH=8,
    EXDOSTXT     "Dose Description"                   LENGTH=50,
    EXDOSU       "Dose Units"                         LENGTH=40,
    EXDOSFRM     "Dose Form"                          LENGTH=50,
    EXDOSFRQ     "Dose Frequency"                     LENGTH=50,
    EXROUTE      "Route of Administration"            LENGTH=40,
    EXFAST       "Fasting Status"                     LENGTH=2,
    EPOCH        "Epoch"                              LENGTH=40,
    EXSTDTC      "Start Date/Time of Treatment"       LENGTH=25,
    EXENDTC      "End Date/Time of Treatment"         LENGTH=25,
    EXSTDY       "Study Day of Treatment Start"       LENGTH=8,
    EXENDY       "Study Day of Treatment End"         LENGTH=8,
    EXDUR        "Treatment Duration (ISO)"           LENGTH=8
FROM EX4;
QUIT;
```
### 💾 Export SDTM-EX to Dataset and XPT
```sas
LIBNAME SDTM "/home/SDTM";
DATA SDTM.EX(LABEL="EXPOSURE");
    SET EX_FINAL;
RUN;

LIBNAME XPT XPORT "/home/SDTM/EX.XPT";
DATA XPT.EX;
    SET EX_FINAL;
RUN;
```
### 📌 Notes
- EXSTDTC and EXENDTC are ISO 8601 compliant timestamps (YYYY-MM-DDTHH:MM:SS)

- EXSEQ resets per subject to enumerate exposure records.

- EXDUR is formatted using ISO 8601 duration syntax (e.g., P1D, P7D).

- All variables are labeled and length-defined to align with CDISC SDTM IG v3.2.

### ✅ Output
- Final SDTM dataset: SDTM.EX

- XPT transport file: SDTM/EX.XPT
