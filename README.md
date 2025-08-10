# CMAQ-ISAM-Tutorial
**This tutorial provides a step-by-step guide to using the Integrated Source Apportionment Method (ISAM) in the Community Multiscale Air Quality (CMAQ) model, taking the Greater Bay Area (GBA) of China as an example.**

ISAM tracks contributions from specific emissions sources and regions to pollutant concentrations, such as ozone or PM2.5. This guide emphasizes key configurations in the run_cctm.csh script (emissions settings), creating mask files for regional tagging, the EmissCtrl_cb6r3_ae7_aq.nml file for the RegionsRegistry namelist, and the isam_control.txt file, using the cb6r3_ae7_aq mechanism as an example. The tutorial assumes basic familiarity with CMAQ. For installation, refer to the EPA CMAQ documentation (https://github.com/USEPA/CMAQ). This example uses CMAQ v5.3+ with ISAM enabled (compile with -Disam flag).
#### Prerequisites:
* Installed CMAQ with ISAM (via bldit_cctm.csh with ISAM option).
* Input data: Meteorology (from WRF/MCIP), emissions (gridded via SMOKE or similar), initial/boundary conditions.
* Tools: netCDF utilities (nco, CDO), Python with netCDF4 for mask creation.

## 1. Prepare Input Data
#### CMAQ-ISAM requires standard CMAQ inputs, plus custom tags for sources/regions.
* **Meteorology and Grid:** Process WRF outputs with MCIP to get METCRO2D, METCRO3D, etc.
* **Emissions Setup:** Emissions are tagged by sector (e.g., biogenic, mobile, power plants) and region. Use SMOKE or equivalent to generate gridded emissions files (e.g., EMIS_3D.nc). In ISAM, emissions streams are labeled (e.g., BIOG_EMIS for biogenic, HK_MV_EMIS for Hong Kong Mobile Sources, GD_PP_EMIS for Guangdong power plants). These labels must match your run_cctm.csh and isam_control.txt.

<br>

## 2. Configure the Run Script (run_cctm.csh)
####    a) Emissions Settings: Define emission file paths and labels. Example snippet for selected streams:
```
set EMISfile = emiss_MEGAN_${RUNID}_${CUR_JDATE}.ncf
setenv GR_EMIS_001 ${EMISpath}/${EMISfile}
setenv GR_EMIS_LAB_001 BIOG_EMIS
setenv GR_EM_SYM_DATE_001 F

set EMISfile = emiss_HK-Power.${RUNID}_${CUR_JDATE}.ncf
setenv GR_EMIS_002 ${EMISpath}/${EMISfile}
setenv GR_EMIS_LAB_002 HK_PP_EMIS
setenv GR_EM_SYM_DATE_002 F

# Continue for e.g. HK_IND_EMIS, HK_MV_EMIS, HK_AR_EMIS, GD_PP_EMIS, GD_IND_EMIS, GD_MV_EMIS, GD_AR_EMIS, MARINE_EMIS, MEIC_EMIS
```
* These labels (BIOG_EMIS, HK_PP_EMIS, etc.) are used in isam_control.txt to tag sources.

####    b) Enable ISAM: For the third domain (e.g., $DOMAINS_GRID == "3"), set setenv CTM_ISAM Y. Additional ISAM configs:
```
#> Integrated Source Apportionment Method (ISAM) Options
if ( "$DOMAINS_GRID" == "3" ) then
  setenv CTM_ISAM Y
  if ( $?CTM_ISAM ) then
     if ( $CTM_ISAM == 'Y' || $CTM_ISAM == 'T' ) then
        setenv SA_IOLIST ${WORKDIR}/isam_control.txt
        setenv ISAM_BLEV_ELEV " 1 1"
        setenv AISAM_BLEV_ELEV " 1 1"
 
        #> Set Up ISAM Initial Condition Flags
        if ($NEW_START == true || $NEW_START == TRUE ) then
           setenv ISAM_NEW_START Y
           setenv ISAM_PREVDAY
        else
           setenv ISAM_NEW_START N
           setenv ISAM_PREVDAY "$OUTDIR/CCTM_SA_CGRID_${RUNID}_${YES_GDATE}.nc"
        endif
 
        #> Set Up ISAM Output Filenames
        setenv SA_ACONC_1      "$OUTDIR/CCTM_SA_ACONC_${CTM_APPL}.nc -v"
        setenv SA_CONC_1       "$OUTDIR/CCTM_SA_CONC_${CTM_APPL}.nc -v"
        setenv SA_DD_1         "$OUTDIR/CCTM_SA_DRYDEP_${CTM_APPL}.nc -v"
        setenv SA_WD_1         "$OUTDIR/CCTM_SA_WETDEP_${CTM_APPL}.nc -v"
        setenv SA_CGRID_1      "$OUTDIR/CCTM_SA_CGRID_${CTM_APPL}.nc -v"
 
     endif
  endif
endif
```

<br>

## 3. Create the Mask File
#### The mask file is a NetCDF file defining binary (0/1) masks for regions. Each variable in the file represents a region (e.g., 'GZ' for Guangzhou), with 1 where the region applies and 0 elsewhere. This allows ISAM to tag emissions/concentrations by geographic area.
* File Name and Path: Name it mask_3km.nc (or similar). Store it in your input directory, e.g., `$CMAQ_DATA/masks/mask_3km.nc`.
* Reference it by adding the following line to `run_cctm.csh`:
  ```bash
  setenv MASK_FN $CMAQ_DATA/masks/mask_3km.nc
* 

<br>

## 4. Configure EmissCtrl_cb6r3_ae7_aq.nml
#### This file contains the emissions control namelist, including the RegionsRegistry for regional masks, tailored for the cb6r3_ae7_aq mechanism. 
* Store the file in your working directory (e.g., `$CMAQ_HOME/scripts/EmissCtrl_cb6r3_ae7_aq.nml`).
* Reference it by adding the following line to `run_cctm.csh`:  
   ```bash
   setenv EMISSCTRL_NML ${cwd}/EmissCtrl_${MECH}.nml
* RegionsRegistry Namelist: Define this in EmissCtrl_cb6r3_ae7_aq.nml:
```
&RegionsRegistry
 RGN_NML  =   
 !          | Region Label   | File_Label  | Variable on File
                'PRD'      ,    'MASK_FN',      'PRD',
                'SZ'       ,    'MASK_FN',      'SZ',
                'GZ'       ,    'MASK_FN',      'GZ',
                'FS'       ,    'MASK_FN',      'FS',
                'DG'       ,    'MASK_FN',      'DG',
                'ZS'       ,    'MASK_FN',      'ZS',
                'HZ'       ,    'MASK_FN',      'HZ',
                'JM'       ,    'MASK_FN',      'JM',
                'ZQ'       ,    'MASK_FN',      'ZQ',
                'ZH'       ,    'MASK_FN',      'ZH',
                'HK'       ,    'MASK_FN',      'HK',
```
* **Meaning:** Each line maps a region label (e.g., 'GZ' for Guangzhou) to a variable in the mask file (MASK_FN points to mask_3km.nc). ISAM uses these to apportion sources within regions. Ensure all regions listed here exist in mask_3km.nc.

<br>

## 5. Configure isam_control.txt
* This file defines tags for source apportionment. Store it in your working directory, e.g., $CMAQ_HOME/isam_control.txt.
* Example for GBA (Ozone Tracking): Based on your provided config, adapted for cb6r3_ae7_aq mechanism. Tracks ozone contributions from sectors (B=Biogenic, P=Power, etc.) in GBA regions (HK, GZ, etc.) and outside (OUTPRD, OPM, etc.). Update labels to match run_cctm.csh (e.g., GD_PP_EMIS instead of PRD_PP_EMIS if applicable).
```
!!! CMAQ-ISAM tag definition control file
!!! (lines begining with !!! - three exclamation marks - are ignored by the text parser)!!!
!!!
!!! Example file provided with CMAQ v5.3.2 release
!!! 26 July 2020: Sergey L. Napelenok
!!!
!!!
!!! The "TAG CLASSES" line defines the tag classes to track for the simulation. Species in NITRATE and VOC classes depend on the
!!! the chemical mechanism used. The below definitions apply for the cb6r3_ae7_aq mechanism. These species will be tracked for
!!! each user-defiend source.
!!! Choose any/all from the list of nine: SULFATE, NITRATE, AMMONIUM, EC, OC, VOC, PM25_IONS, OZONE, CHLORINE
!!! SULFATE - ASO4J, ASO4I, SO2, SULF, SULRXN
!!! NITRATE - ANO3J, ANO3I, HNO3, NO, NO2, NO3, HONO, N2O5, PNA, PAN, PANX, NTR1, NTR2, INTR, CLNO2, CLNO3
!!! AMMONIUM - ANH4J, ANH4I, NH3
!!! EC - AECJ, AECI
!!! OC - APOCI, APOCJ, APNCOMI, APNCOMJ
!!! VOC - Various species depending on mechanism. Now includes CO. (see CCTM/src/isam/SA_DEFN.F for complete list)
!!! PM25_IONS - ANAI, ANAJ, AMGJ, AKJ, ACAJ, AFEJ, AALJ, ASIJ, ATIJ, AMNJ, AOTHRI, AOTHRJ
!!! OZONE - O3, all NITRATE species, and all VOC species
!!! CHLORINE - ACLI, ACLJ, HCL

TAG CLASSES |OZONE

!!! The following are source definition text blocks in the format. Provide a 3-line block for each source you want to track.
!!! Do not assign the same source of mass in more than 1 source definition block.
!!! TAG NAME |Three character text string (unique to each source definition)
!!! REGION(S) |Keyword EVERYWHERE or variable names from the region file (multiple regions need to be comma delimited)
!!! EMIS STREAM(S) |Emissions labels (multiple labels need to be comma delimited)

!!! Target regions: HK (Hong Kong), GZ (Guangzhou), FS (Foshan), HZ (Huizhou), ZS (Zhongshan), ZQ (Zhaoqing), ZH (Zhuhai), SZ (Shenzhen), JM (Jiangmen), DG (Dongguan), OUTPRD (Outside PRD)
!!! Target sectors: B (Biogenic), P (Power Plant), I (Industry), M (Mobile), O (Other), SHP (Ship)

TAG NAME |HKB
REGION(S) |HK
EMIS STREAM(S) |BIOG_EMIS

TAG NAME |HKP
REGION(S) |EVERYWHERE
EMIS STREAM(S) |HK_PP_EMIS

...

TAG NAME        |GZB
REGION(S)       |GZ
EMIS STREAM(S)  |BIOG_EMIS

TAG NAME        |GZP
REGION(S)       |GZ
EMIS STREAM(S)  |PRD_PP_EMIS

TAG NAME        |GZF
REGION(S)       |GZ
EMIS STREAM(S)  |PRD_LAF_EMIS

...

# (Continue similarly for other streams and regions)

ENDLIST eof
```
* #### Meaning:
  * TAG CLASSES |OZONE: Tracks ozone-related species (O3, NOx, VOCs).
  * Each tag block: TAG NAME is a unique 3-letter code (e.g., 'HKB' = Hong Kong Biogenic). REGION(S) links to mask variables or 'EVERYWHERE'. EMIS STREAM(S) matches emission labels from run_cctm.csh.
  * Avoid double-tagging sources. For GBA, this tags city-specific sectors (e.g., Guangzhou power plants as 'GZP') and ships/maritime everywhere.
