# GRAB

### How to install and load this package

Rtools (https://cran.rstudio.com/bin/windows/Rtools/) should be installed before installing this package.

```{r}      
library(devtools)  # author version: 2.3.0, use install.packages("devtools") first
install_github("GeneticAnalysisinBiobanks/GRAB", INSTALL_opts=c("--no-multiarch"))  # The INSTALL_opts is required in Windows OS.
library(GRAB)
```

### Replicate the genotype simulation in the package
```{r}   
set.seed(12345)
OutList = GRAB.SimuGMatCommon(nSub = 500, nFam = 50, FamMode = "10-members", nSNP = 10000)
GenoMat = OutList$GenoMat 
MissingRate = 0.05
indexMissing = sample(length(GenoMat), MissingRate * length(GenoMat))
GenoMat[indexMissing] = -9
extDir = system.file("extdata", package = "GRAB")
extPrefix = paste0(extDir, "/simuPLINK")
GRAB.makePlink(GenoMat, extPrefix)

setwd(extDir)
system("plink --file simuPLINK --make-bed --out simuPLINK")
system("plink --bfile simuPLINK --recode A --out simuRAW")
system("plink2 --bfile simuPLINK --export bgen-1.2 bits=8 ref-first --out simuBGEN  # UK Biobank use 'ref-first'")
system("bgenix -g simuBGEN.bgen -index")
```

### Replicate the sparse GRM generation in the package
```{r}
GenoFile = system.file("extdata", "simuPLINK.bed", package = "GRAB")
PlinkFile = tools::file_path_sans_ext(GenoFile)  # remove file extension
nPartsGRM = 2
for(partParallel in 1:nPartsGRM){
  getTempFilesFullGRM(PlinkFile, nPartsGRM, partParallel) # this function only supports Linux
}
SparseGRMFile = system.file("SparseGRM", "SparseGRM.txt", package = "GRAB")
getSparseGRM(PlinkFile, nPartsGRM, SparseGRMFile)
```

### Replicate the phenotype simulation in the package
```{r}
FamFile = system.file("extdata", "simuPLINK.fam", package = "GRAB")
FamData = read.table(FamFile)
IID = FamData$V2  # Individual ID
n = length(IID)
set.seed(678910)
## The below is just to demonstrate the functions of GRAB package
Pheno = data.frame(IID = IID, Cova1 = rnorm(n), Cova2 = rbinom(n, 1, 0.5), 
                   binary = rbinom(n, 1, 0.5),
                   ordinal = rbinom(n, 3, 0.3),
                   quantitative = rnorm(n),
                   time = runif(n),
                   event = rbinom(n, 1, 0.2))
PhenoFile = system.file("extdata", "simuPHENO.txt", package = "GRAB")
write.table(Pheno, PhenoFile, row.names = F, quote = F, sep = "\t")
```

### Step 1: Fit a null model using the sparse GRM and phenotype data
```{r}
GenoFile = system.file("extdata", "simuPLINK.bed", package = "GRAB")
PhenoFile = system.file("extdata", "simuPHENO.txt", package = "GRAB")
Pheno = read.table(PhenoFile, header = T)
SparseGRMFile = system.file("SparseGRM", "SparseGRM.txt", package = "GRAB")
objNullFile = system.file("results", "objNull.RData", package = "GRAB")

objNull = GRAB.NullModel(as.factor(ordinal) ~ Cova1 + Cova2, 
                         data = Pheno, 
                         subset = (event==0), 
                         subjData = Pheno$IID, 
                         method = "POLMM", 
                         traitType = "ordinal", 
                         GenoFile = GenoFile,
                         SparseGRMFile = SparseGRMFile)

objNull$tau    # 0.5655751
                         
save(objNull, file = objNullFile)
```

### Step 2: Perform a genome-wide analysis for marker-level and region-level
```{r}
objNullFile = system.file("results", "objNull.RData", package = "GRAB")
load(objNullFile)

OutputFile = system.file("results", "simuOUTPUT.txt", package = "GRAB")
GenoFile = system.file("extdata", "simuPLINK.bed", package = "GRAB")

# If 'OutputFile' and 'OutputFileIndex' have existed, users might need to remove them first.
GRAB.Marker(objNull, 
            GenoFile = GenoFile,
            OutputFile = OutputFile)
            
## Check 'OutputFile' for the analysis results
```

### NOTE for BGEN file input

The current version of GRAB package only supports BGEN v1.2 using 8 bits compression (faster than using 16 bits). For example, if plink2 is used to make BGEN file, please refer to https://www.cog-genomics.org/plink/2.0/data#export
```
plink2 --bfile input --out output --export bgen-1.2 bits=8
```

The index file for BGEN file is also required (filename extension is ".bgen.bgi"). Please refer to https://enkre.net/cgi-bin/code/bgen/wiki/bgenix 


### More detailed information for package developers

For SPACox method, we should create the following files

```{r}
/src/SPACox.cpp
/src/SPACox.hpp
/R/SPACox.R
```
and modify the following codes

```{r}
/src/Main.cpp  
# static SPACox::SPACoxClass* ptr_gSPACoxobj = NULL;
# void setSPACoxobjInCPP()
/R/GRAB_Null_Model.R
/R/GRAB_Marker.R
/R/GRAB_Region.R
```
