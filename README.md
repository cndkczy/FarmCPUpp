## Readme

FarmCPUpp provides an efficient reimplementation of the FarmCPU R scripts found [here](http://zzlab.net/FarmCPU/index.html). Through the use of the `bigmemory`, `Rcpp`, `RcppEigen`, and `RcppParallel` packages, FarmCPUpp decreases the memory usage of the original FarmCPU scripts and improves the runtime through parallelization. FarmCPUpp will provide the greatest benefits on large datasets and on machines with many CPU cores.

### Installation

FarmCPUpp relies on multiple HPC packages for efficient memory usage and parallel processing. To install FarmCPUpp itself, you will need the `devtools` package. Run this code to ensure that you have the appropriate packages installed and updated:

```
packages <- c("Rcpp", "RcppEigen", "RcppParallel", "Rmpi", "snow", "doSNOW",
              "foreach", "bigmemory", "biganalytics", "devtools")
install.packages(packages)
devtools::install_github(repo = "amkusmec/FarmCPUpp")
```

Following installation the package can be loaded for use with

```
library(bigmemory)
library(FarmCPUpp)
```

#### Installing `Rmpi` on Mac OS X

Apple no longer bundles OpenMPI with Mac OS X which causes problems when installing `Rmpi`. In order to install `Rmpi`, follow these steps:

1. Install Xcode 7 or later from the App Store.
2. Open the Terminal application and install the Xcode Command Line Tools.
```
xcode-select --install
```
3. Install Homebrew.
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew doctor # To check for installation problems
```
If you run into problems with file permissions, change the ownership on the Homebrew directory:
```
sudo chown -R $(whoami):admin /usr/local
```
4. Install gcc.
```
brew install gcc
```
5. Install OpenMPI.
```
brew install open-mpi
```
6. The last line of output from installing OpenMPI will tell you where Homebrew installed it. This should be a directory of the form `/usr/local/Cellar/open-mpi/x.x.x`, where `x.x.x` is the version number.
7. Install `Rmpi` in R or RStudio:
```
install.packages("Rmpi", type = "source",
                 configure.arges = paste(
                    "--with-Rmpi-include=/usr/local/Cellar/open-mpi/x.x.x/include",
                    "--with-Rmpi-libpath=/usr/local/Cellar/open-mpi/x.x.x/lib",
                    "--with-Rmpi-type=OPENMPI"
                 ))
```

### Data Formats

FarmCPUpp requires three data files with two optional files. Please note that the order of data in these files is important; results obtained will be incorrect if files are not ordered properly. Please see section [Data Dependencies](#data-dependencies) for more information.

#### Phenotype Data

Phenotype data should be provided as a two column dataframe where the first column contains sample names and the second column contains numeric phenotypic values. Missing values are allowed and should be specified as `NA`. The column name of the phenotypic data will be used for output file names. A dataframe with more than two columns may be provided to FarmCPUpp, but only the phenotype in the second column will be analyzed. The code below loads a sample phenotype file.

```
myY <- read.table(system.file("extdata", "mdp_traits_validation.txt",
                              package = "FarmCPUpp"),
                  header = TRUE, stringsAsFactors = FALSE)
```

#### Genotype Data

Genotype data should be provided in numerical format where 0 indicates no copies of the minor allele, 1 indicates a single copy of the minor allele, and 2 indicates two copies of the minor allele. Any value in the range [0,2] is accepted. The file contains marker scores in the columns and samples in the rows. The file should also include a header line, and the first column should contain taxa names. Note that the marker IDs from the genotype information file are used to redefine the header line. Because genotype files will often be very wide and large, the `bigmemory` package is used to import the data:

```
myGD <- read.big.matrix(system.file("extdata", "mdp_numeric.txt",
                                    package = "FarmCPUpp"),
                        type = "double", sep = "\t", header = TRUE,
                        col.names = myGM$SNP, ignore.row.names = FALSE,
                        has.row.names = TRUE)
```

If the same genotype file will be used in multiple GWASs, it may be advantageous to create a file-backed big matrix that can be reattached, since creating a big matrix from a large genotype file can be time consuming. The following code demonstrates how to create the file-backed big matrix and access it later.

```
# Load the data into a big.matrix.
# Note the use of the backingfile and descriptorfile arguments.
myGD <- read.big.matrix(system.file("extdata", "mdp_numeric.txt",
                                    package = "FarmCPUpp"),
                        type = "double", sep = "\t", header = TRUE,
                        col.names = myGM$SNP, ignore.row.names = FALSE,
                        has.row.names = TRUE, backingfile = "mdp_numeric.bin",
                        descriptorfile = "mdp_numeric.desc")

# Save the pointer for access later
dput(describe(myGD), "mdp_numeric_pointer.desc")

# The big.matrix can be reattached in a different R session using
desc <- dget("mdp_numeric_pointer.desc")
myGD <- attach.big.matrix(desc)
```

#### Genotype Information

Genotype information should be provided as a three column dataframe with colum names. The first column, SNP, should contain unique IDs for each marker. The second column, Chromosome, should contain integer or numeric IDs for each chromosome. The third column, Position, shoudl contain the integer base-pair positions of each marker. The code below loads a sample genotype information file.

```
myGM <- read.table(system.file("extdata", "mdp_SNP_information.txt",
                               package = "FarmCPUpp"),
                   header = TRUE, stringsAsFactors = FALSE)
```

#### Covariates

User-specified covariates can also be included in the GWAS. These may include principal components of the genotype data, results from programs such as STRUCTURE, or other experiment-related covariates that may be important for the GWAS. These data should be provided as a matrix with column names specifying the names of the covariates and row names specifying the sample names.

#### Genotype Prior

FarmCPUpp also accepts a dataframe of prior probabilities that a marker may be selected as a pseudo-QTN. These should be provided in a dataframe with the same format and information as the [genotype information](#genotype-information) with a fourth column named Probability that contains numeric probabilities.

#### Data Dependencies

FarmCPUpp assumes that the numbers of samples and markers are the same across all input data. Therefore, the number of rows in the genotype information should be equal to the number of rows of the prior and the number of columns of the genotype data, and all three should be sorted in the same order. Additionally, the number of rows of the phenotype data, covariates, and genotype data should be equal and sorted in the same order. This means that there may be samples in the genotype table that are not present in the phenotype table, for example. These samples should be added and missing values entered as appropriate. Missing values will be removed as necessary by FarmCPUpp.

### GWAS

GWAS is performed through one main function called `farmcpu`. This function requires phenotype data, genotype information, and genotype data. The simplest GWAS is run using

```
myResults <- farmcpu(Y = myY, GD = myGD, GM = myGM)
```

Please see the documentation (`?farmcpu`) for more information on the various arguments and options. By default FarmCPUpp will run on one core. The user may specify different numbers of cores to use for single-marker regressions and bin selection.

### Working with the Results

FarmCPUpp returns a results list. If `iteration.output = TRUE`, the list will contain one element for each iteration plus a final element with the same name as the phenotype that contains the final results. Each element of the results list is itself a list and may contain up to two elements. The first element, named GWAS, contains the GWAS results, including the information for each marker along with its effect estimate, standard error, t-statistic, and p-value. The second element, named CV, will contain a matrix with the covariate effect estimates if covariates were provided. The results object may be conveniently written to disk by

```
write_results(myResults)
```

FarmCPUpp also contains a function for creating manhattan plots from a single GWAS results table. The plotting colors and y-axis label may be customized. The user may also input a desired significance threshold. For example, to create a manhattan plot for the EarDia example data at a 0.01 significance threshold, use the following code.

```
manhattan_plot(myResults$EarDia$GWAS, cutoff = 0.01)
```
