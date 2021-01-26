Tutorial for multiclassPairs R package
================

Created: 26/01/2021

<!-- badges: start -->
[![CRAN
status](https://www.r-pkg.org/badges/version/multiclassPairs)](https://www.r-pkg.org/badges/version/multiclassPairs)
[![Lifecycle:
stable](https://img.shields.io/badge/lifecycle-stable-brightgreen.svg)](https://www.tidyverse.org/lifecycle/#stable)
[![CRAN RStudio mirror
downloads](https://cranlogs.r-pkg.org/badges/multiclassPairs)](https://cran.r-project.org/package=multiclassPairs)
<!-- badges: end -->
Build single sample pairs-based (rule-based) classifiers using top-score pairs or random forest for multi-class problems.

Description: A toolbox to train a single sample classifier that uses in-sample feature relationships. The relationships are represented as feature1 < feature2 (e.g. gene1 < gene2). We provide two options to go with. The first is based on 'switchBox' package which uses Top-score pairs algorithm. Second is a novel implementation based on random forest algorithm. For simple problems, we recommend to use one-vs-rest using TSP option due to its simplicity and for being easy to interpret.  For complex problems, RF performs better.  Both lines filter the features first then combine the filtered features to make the list of all the possible rules (i.e. rule1: feature1 < feature2, rule2: feature1 < feature3, etc...).  Then the list of rules will be filtered and the most important and informative rules will be kept. The informative rules will be assembled in a one-vs-rest model or an RF model.  We provide a detailed description with each function in this package to explain the filtration and training methodology in each line. 

author:
  - name: "By Nour-al-dain Marzouka"
    email: "nour-al-dain.marzouka@med.lu.se"
	
In this tutorial we show how to use the different functions and options
in `multiclassPairs` package (v0.4.1), including training a pair-based
classifier, predict classes in test data, and visualize the prediction
scores and rules.

# Installation

`multiclassPairs` package is available on
[CRAN](https://cran.r-project.org/package=multiclassPairs) and
[GitHub](https://github.com/NourMarzouka/multiclassPairs). You can use
the following code to install `multiclassPairs` package from
[CRAN](https://cran.r-project.org/package=multiclassPairs) and
[GitHub](https://github.com/NourMarzouka/multiclassPairs) and its
dependencies from Bioconductor.

``` r
# Install the released version from CRAN using
if (!requireNamespace("multiclassPairs", quietly = TRUE)) {
  install.packages("multiclassPairs")
}

# Or install the dev version from GitHub using
# if (!requireNamespace("multiclassPairs", quietly = TRUE)) {
#  if (!requireNamespace("devtools", quietly = TRUE)) {
#    install.packages("devtools")
#  }
#  library(devtools) # this package is needed to install from GitHub
#  install_github("NourMarzouka/multiclassPairs", build_vignettes = TRUE)
#}

# Install the dependencies from Bioconductor
# BiocManager, Biobase, and switchBox packages from Bioconductor are needed
if (!requireNamespace("BiocManager", quietly = TRUE)) {
  install.packages("BiocManager")
}
if (!requireNamespace("Biobase", quietly = TRUE)) {
  BiocManager::install("Biobase")
}
if (!requireNamespace("switchBox", quietly = TRUE)) {
  BiocManager::install("switchBox")
}

# load multiclassPairs library
library(multiclassPairs)
```

# Workflow

The workflow in `multiclassPairs` is summarized in the next figure:

<div class="figure" style="text-align: center">

<img src="images/workflow_v0_3.png" alt="Workflow in multiclassPairs R package: Functions are colored in green." width="1565" />
<p class="caption">
Workflow in multiclassPairs R package: Functions are colored in green.
</p>

</div>

The workflow always starts with `ReadData` function. Then you have two
schemes to train your pair-based classifier:

-   First option is a one-vs-rest scheme that assemble one-vs-rest
    binary classifiers built by ‘switchBox’ package which uses Top-score
    pairs (TSP) algorithm.
-   The second option is a scheme based on a novel implementation of the
    random forest (RF) algorithm.

For more complex predictions, RF performs better. For less complex
predictions, we recommend to use one-vs-rest due to its straightforward
application and interpretation.

Both workflows begin by filtering the features, then combining the
filtered features to make the list of all the possible rules
(i.e. rule1: feature1 &lt; feature2, rule2: feature1 &lt; feature3,
etc…). Then the list of rules will be filtered and the most important
and informative rules will be kept. The informative rules will be
assembled in an one-vs-rest model or in an RF model. We provide a
detailed description for the methodologies for each step in the help
files in the package.

# Create data object

`ReadData` takes the data (data.frame/matrix/ExpressionSet) and class
labels for the samples, then generates a data object to be used in the
downstream steps, such as genes filtering, training, and visualization.

Optionally, `ReadData` accepts additional platform/study labels and
includes it in the data object when more than one platform/study are
involved, this helps in performing the downstream steps in
platform/study wise manner, where the filtering of the genes/rules is
performed in the samples of each platform/study separately, then the top
genes/rules in all of the platforms/studies are selected.

Here is an example of creating data object using a matrix:

``` r
library(multiclassPairs)

# example of creating data object from matrix
# we will generate fake data in this example
# matrix with random values
Data <- matrix(runif(100000), 
               nrow=100, 
               ncol=100, 
               dimnames = list(paste0("G",1:100), 
                               paste0("S",1:100)))
# class labels
L1 <- sample(x = c("A","B","C"), size = 100, replace = TRUE)

# platform/study labels
P1 <- sample(x = c("P1","P2"), size = 100, replace = TRUE)

# create the data object
object <- ReadData(Data = Data,
                   Labels = L1,
                   Platform = P1,
                   verbose = FALSE)
object
```

    ## multiclassPairs object
    ## *Data:
    ##    Number of samples: 100 
    ##    Number of genes/features: 100 
    ## *Labels and Platforms:
    ##    Classes: A C B 
    ##    Platforms/studies: P2 P1 
    ## *Samples count table:    
    ##       A  B  C
    ##   P1 21 10 13
    ##   P2 19 16 21

Here is an example using
[`leukemiasEset`](https://bioconductor.org/packages/release/data/experiment/html/leukemiasEset.html)
data from Bioconductor packages, which is an `Expressionset` containing
gene expression data from 60 bone marrow samples of patients with one of
the four main types of leukemia (ALL, AML, CLL, CML) or non-leukemia.
Note that this example is just to show the functions and the options in
the `multiclassPairs` package. For a better comparison between the
workflows and other tools see the comparison session in the end of this
tutorial.

``` r
library(multiclassPairs, quietly = TRUE)

# install Leukemia cancers data
if (!requireNamespace("BiocManager", quietly = TRUE)){
  install.packages("BiocManager")
}
if (!requireNamespace("leukemiasEset", quietly = TRUE)){
  BiocManager::install("leukemiasEset")
}

# load the data package
library(leukemiasEset, quietly = TRUE)
data(leukemiasEset)
```

``` r
# check the Expressionset
leukemiasEset
```

    ## ExpressionSet (storageMode: lockedEnvironment)
    ## assayData: 20172 features, 60 samples 
    ##   element names: exprs, se.exprs 
    ## protocolData
    ##   sampleNames: GSM330151.CEL GSM330153.CEL ... GSM331677.CEL (60 total)
    ##   varLabels: ScanDate
    ##   varMetadata: labelDescription
    ## phenoData
    ##   sampleNames: GSM330151.CEL GSM330153.CEL ... GSM331677.CEL (60 total)
    ##   varLabels: Project Tissue ... Subtype (5 total)
    ##   varMetadata: labelDescription
    ## featureData: none
    ## experimentData: use 'experimentData(object)'
    ## Annotation: genemapperhgu133plus2

``` r
# explore the phenotypes data
knitr::kable(head(pData(leukemiasEset)))
```

|               | Project | Tissue     | LeukemiaType | LeukemiaTypeFullName         | Subtype                            |
|:--------------|:--------|:-----------|:-------------|:-----------------------------|:-----------------------------------|
| GSM330151.CEL | Mile1   | BoneMarrow | ALL          | Acute Lymphoblastic Leukemia | c\_ALL/Pre\_B\_ALL without t(9 22) |
| GSM330153.CEL | Mile1   | BoneMarrow | ALL          | Acute Lymphoblastic Leukemia | c\_ALL/Pre\_B\_ALL without t(9 22) |
| GSM330154.CEL | Mile1   | BoneMarrow | ALL          | Acute Lymphoblastic Leukemia | c\_ALL/Pre\_B\_ALL without t(9 22) |
| GSM330157.CEL | Mile1   | BoneMarrow | ALL          | Acute Lymphoblastic Leukemia | c\_ALL/Pre\_B\_ALL without t(9 22) |
| GSM330171.CEL | Mile1   | BoneMarrow | ALL          | Acute Lymphoblastic Leukemia | c\_ALL/Pre\_B\_ALL without t(9 22) |
| GSM330174.CEL | Mile1   | BoneMarrow | ALL          | Acute Lymphoblastic Leukemia | c\_ALL/Pre\_B\_ALL without t(9 22) |

``` r
# We are interested in LeukemiaType
knitr::kable(table(pData(leukemiasEset)[,"LeukemiaType"]))
```

| Var1 | Freq |
|:-----|-----:|
| ALL  |   12 |
| AML  |   12 |
| CLL  |   12 |
| CML  |   12 |
| NoL  |   12 |

``` r
# split the data
# 60% as training data and 40% as testing data
n <- ncol(leukemiasEset)
set.seed(1234)
training_samples <- sample(1:n,size = n*0.6)

train <- leukemiasEset[1:1000,training_samples]
test  <- leukemiasEset[1:1000,-training_samples]

# just to be sure there are no shared samples between the training and testing data
sum(sampleNames(test) %in% sampleNames(train)) == 0
```

    ## [1] TRUE

``` r
# create the data object
# when we use Expressionset we can use the name of the phenotypes variable 
# ReadData will automatically extract the phenotype variable and use it as class labels
# the same can be used with the Platform/study labels
# in this example we are not using any platform labels, so leave it NULL
object <- ReadData(Data = train, 
                   Labels = "LeukemiaType", 
                   Platform = NULL, 
                   verbose = FALSE)
object
```

    ## multiclassPairs object
    ## *Data:
    ##    Number of samples: 36 
    ##    Number of genes/features: 1000 
    ## *Labels and Platforms:
    ##    Classes: CLL AML NoL CML ALL 
    ##    Platforms/studies: NULL
    ## *Samples count table:
    ## ALL AML CLL CML NoL 
    ##   8   6   7   8   7

# One-vs-rest scheme

One-vs-rest scheme is composed from binary individual classifiers for
each class (i.e. each class versus others). Each binary classifier votes
(i.e. give a score) for the predicted sample, and the sample’s class is
predicted based on the highest score.

<div class="figure" style="text-align: center">

<img src="images/one_vs_rest_scheme.png" alt="One-vs-rest scheme" width="1664" />
<p class="caption">
One-vs-rest scheme
</p>

</div>

## Gene filtering

For building a pair-based classifier with a one-vs-rest scheme, we start
by selecting top differentially expressed genes using the
`filter_genes_TSP` function. This reduces the number of gene
combinations (rules) in the next steps. This function can perform the
filtering in different ways and return the top differential expressed
genes for each class.

`filter_genes_TSP` function provides two options for gene filtering.
Both options begins by ranking the data (i.e. in-sample ranking). The
first option performs one-vs-rest comparison using Wilcoxon test and
then selects a number of the top genes. Wilcoxon test is done separately
for each class. The second option performs one-vs-one comparison using
Dunn’s test, and then selects a number of the top genes. Dunn’s test is
performed for all classes together.

<div class="figure" style="text-align: center">

<img src="images/gene_filtering_TSP.png" alt="Gene filtering options" width="1962" />
<p class="caption">
Gene filtering options
</p>

</div>

Previous filtering options can be done for samples from each
platform/study separately, then the top genes in all platforms/studies
will be selected.

<div class="figure" style="text-align: center">

<img src="images/platform_wise_gene_filtering_TSP.png" alt="Platform-wise gene filtering" width="2102" />
<p class="caption">
Platform-wise gene filtering
</p>

</div>

The reason for using one-vs-one gene filtering is to give more weight to
small classes. The reason for using platform-wise gene filtering is to
give more weight to platforms with small sample size, and to select
genes that are important in all platforms/studies. However, one-vs-one
and platform-wise options does not guarantee better results but they are
valid options to be considered during the training process.

More details about the filtering process is mentioned in the
documentation of `filter_genes_TSP` function.

Here we will run a simple gene filtering example on the
[`leukemiasEset`](https://bioconductor.org/packages/release/data/experiment/html/leukemiasEset.html)
data.

``` r
# let's go with gene filtering using one_vs_one option
# for featureNo argument, a sufficient number of returned features is 
# recommended if large number of rules is used in the downstream training steps.
filtered_genes <- filter_genes_TSP(data_object = object,
                                   filter = "one_vs_one",
                                   platform_wise = FALSE,
                                   featureNo = 1000,
                                   UpDown = TRUE,
                                   verbose = TRUE)
filtered_genes
```

    ## sorted genes for One-vs-rest Scheme:
    ##   Object contains:
    ##       - filtered_genes 
    ##          - class: CLL : 554 genes
    ##          - class: AML : 241 genes
    ##          - class: NoL : 381 genes
    ##          - class: CML : 440 genes
    ##          - class: ALL : 378 genes
    ##       - calls 
    ##             filter_genes_TSP(data_object = object,
    ##              filter = "one_vs_one",
    ##              platform_wise = FALSE,
    ##              featureNo = 1000,
    ##              UpDown = TRUE,
    ##              verbose = TRUE)

We asked for 1000 filtered genes per class (i.e. `featureNo` argument)
but the filtered genes object has less than 1000 genes per class. This
is because the function did not find enough genes matching the top genes
criteria (i.e. high/low in the class and the opposite in the other
classes).

To skip the step of filtering genes, one can create the filtering genes
object with all available genes as in the next chunk. However, this
significantly increases the training time.

``` r
# using the object that is generated by ReadData
# we can create genes object with all genes to skip filtering step

# Get the class names
classes <- unique(object$data$Labels)

# create empty genes object
genes_all <- list(OnevsrestScheme = list(filtered_genes = NULL,
                                         calls = c()))
class(genes_all) <- "OnevsrestScheme_genes_TSP"

# prepare the slots for each class
tmp <- vector("list", length(classes))
names(tmp) <- classes

genes_all$OnevsrestScheme$filtered_genes <- tmp
genes_all$OnevsrestScheme$calls <- c()
genes_all

# fill the gene object in each slot
for (i in classes) {
  genes_all$OnevsrestScheme$filtered_genes[[i]] <- rownames(object$data$Data)
}

# This is the gene object with all genes
genes_all
```

## Model training

After filtering the genes, we can train our model using
`train_one_vs_rest_TSP` function. This function combines the filtered
genes as binary rules (GeneA &lt; GeneB, GeneA &lt; GeneC, etc.) then
rules are sorted based on their score, and the optimal number of rules
among the input range (i.e. `k_range` argument) is selected. The optimal
number of rules is determined internally through the `SWAP.Train.KTSP`
function from `switchBox` which uses Variance Optimization (VO) method
as in (Afsari et al. 2014).

Rules scoring is performed by the traditional scoring method [(Geman et
al. 2004)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC1989150/).
Briefly, the score for a given rule equals the percentage of samples
those are TRUE for the rule in the class minus the percentage of the
samples those are TRUE for the rule in another group(s).
`train_one_vs_rest_TSP` function gives two scoring options: one-vs-rest
and one-vs-one. In one-vs-rest scoring option, all the rest classes are
grouped in one ‘rest’ group. In one-vs-one scoring option, the score is
calculated as the mean of one-vs-one rule scores. The reason for using
one-vs-one option is to give more weight to small classes.

Scores can also be calculated in a platform-wise manner where the score
is calculated in each platform/study separately, then the mean of these
scores will be the final score for the rule.

The `k_range` argument defining the candidate number of top rules from
which the algorithm chooses to build the binary classifier for each
class. Note that during prediction process (i.e. class voting) binary
classifiers with low number of rules tie more than binary classifiers
with higher number of rules. Because of that, it is recommended to use
sufficient number of rules to avoid the score ties. Sufficient number of
rules depends on how many classes we have and on how much these classes
are distinct from each other.

Rules have two sides (i.e. GeneA &lt; GeneB), usually rules are composed
from two differently expressed genes (i.e. filtered genes). However,
invariant genes (called pivot genes) can be involved in the rules
through `include_pivot` argument. If `include_pivot` argument is TRUE,
then the filtered genes will be combined with all genes in the data
allowing rules in form of ‘filtered genes &lt; pivot gene’ and ‘pivot
gene &lt; filtered genes’. This may increase the training time due to
increase the number of the possible rules.

``` r
# Let's train our model
classifier <- train_one_vs_rest_TSP(data_object = object,
                                    filtered_genes = filtered_genes,
                                    k_range = 5:50,
                                    include_pivot = FALSE,
                                    one_vs_one_scores = TRUE,
                                    platform_wise_scores = FALSE,
                                    seed = 1234,
                                    verbose = FALSE)
classifier
```

    ## multiclassPairs - One vs rest Scheme
    ## *Classifier:
    ##   contains binary classifiers:
    ##      - Class: CLL ... 24 rules
    ##      - Class: AML ... 8 rules
    ##      - Class: NoL ... 5 rules
    ##      - Class: CML ... 5 rules
    ##      - Class: ALL ... 7 rules

So, in our trained model we have 5 binary classifiers for the classes:
CLL, AML, NoL, CML, ALL, with 24,8,5,5,7 rules, respectively.

## Prediction

Now, let’s apply our trained model on the training and testing data to
assess its performance. We can do that through `predict_one_vs_rest_TSP`
function.

Note that, in some cases the testing data miss some features or genes,
`predict_one_vs_rest_TSP` can deal with this through ignoring the rules
with missed genes. However, if a lot of rules are ignored the prediction
may be affected to large extend.

`predict_one_vs_rest_TSP` returns a data.frame with class scores, score
ties, and final prediction based on the max score.

``` r
# apply on the training data
# To have the classes in output in specific order, we can use classes argument
results_train <- predict_one_vs_rest_TSP(classifier = classifier,
                                         Data = object,
                                         tolerate_missed_genes = TRUE,
                                         weighted_votes = TRUE,
                                         classes = c("ALL","AML","CLL","CML","NoL"),
                                         verbose = TRUE)

# apply on the testing data
results_test <- predict_one_vs_rest_TSP(classifier = classifier,
                                        Data = test,
                                        tolerate_missed_genes = TRUE,
                                        weighted_votes = TRUE,
                                        classes=c("ALL","AML","CLL","CML","NoL"),
                                        verbose = TRUE)
# get a look over the scores in the testing data
knitr::kable(head(results_test))
```

|               |       ALL |       AML |       CLL |       CML |       NoL | max\_score | tie\_flag |
|:--------------|----------:|----------:|----------:|----------:|----------:|:-----------|:----------|
| GSM330151.CEL | 0.2857157 | 0.1219990 | 0.4999986 | 0.0000000 | 0.0000000 | CLL        | NA        |
| GSM330178.CEL | 0.5714304 | 0.1219990 | 0.0416651 | 0.2012551 | 0.1916289 | ALL        | NA        |
| GSM330186.CEL | 1.0000000 | 0.0000000 | 0.0000000 | 0.0000000 | 0.0000000 | ALL        | NA        |
| GSM330195.CEL | 0.8571444 | 0.1219951 | 0.0416651 | 0.0000000 | 0.0000000 | ALL        | NA        |
| GSM330532.CEL | 0.2857131 | 0.8695936 | 0.0000000 | 0.0000000 | 0.1938418 | AML        | NA        |
| GSM330571.CEL | 0.0000000 | 0.8780049 | 0.0416656 | 0.0000000 | 0.3854708 | AML        | NA        |

Let’s check the accuracy in the training and testing data.

``` r
# Confusion Matrix and Statistics on training data
caret::confusionMatrix(data = factor(results_train$max_score, 
                                     levels = unique(object$data$Labels)),
                       reference = factor(object$data$Labels, 
                                          levels = unique(object$data$Labels)),
                       mode="everything")
```

    ## Confusion Matrix and Statistics
    ## 
    ##           Reference
    ## Prediction CLL AML NoL CML ALL
    ##        CLL   7   0   0   0   0
    ##        AML   0   6   0   0   0
    ##        NoL   0   0   7   0   0
    ##        CML   0   0   0   8   0
    ##        ALL   0   0   0   0   8
    ## 
    ## Overall Statistics
    ##                                      
    ##                Accuracy : 1          
    ##                  95% CI : (0.9026, 1)
    ##     No Information Rate : 0.2222     
    ##     P-Value [Acc > NIR] : < 2.2e-16  
    ##                                      
    ##                   Kappa : 1          
    ##                                      
    ##  Mcnemar's Test P-Value : NA         
    ## 
    ## Statistics by Class:
    ## 
    ##                      Class: CLL Class: AML Class: NoL Class: CML Class: ALL
    ## Sensitivity              1.0000     1.0000     1.0000     1.0000     1.0000
    ## Specificity              1.0000     1.0000     1.0000     1.0000     1.0000
    ## Pos Pred Value           1.0000     1.0000     1.0000     1.0000     1.0000
    ## Neg Pred Value           1.0000     1.0000     1.0000     1.0000     1.0000
    ## Precision                1.0000     1.0000     1.0000     1.0000     1.0000
    ## Recall                   1.0000     1.0000     1.0000     1.0000     1.0000
    ## F1                       1.0000     1.0000     1.0000     1.0000     1.0000
    ## Prevalence               0.1944     0.1667     0.1944     0.2222     0.2222
    ## Detection Rate           0.1944     0.1667     0.1944     0.2222     0.2222
    ## Detection Prevalence     0.1944     0.1667     0.1944     0.2222     0.2222
    ## Balanced Accuracy        1.0000     1.0000     1.0000     1.0000     1.0000

``` r
# Confusion Matrix and Statistics on testing data
caret::confusionMatrix(data = factor(results_test$max_score, 
                                     levels = unique(object$data$Labels)),
                       reference = factor(pData(test)[,"LeukemiaType"], 
                                          levels = unique(object$data$Labels)),
                       mode="everything")
```

    ## Confusion Matrix and Statistics
    ## 
    ##           Reference
    ## Prediction CLL AML NoL CML ALL
    ##        CLL   5   1   0   0   1
    ##        AML   0   5   0   0   0
    ##        NoL   0   0   5   0   0
    ##        CML   0   0   0   4   0
    ##        ALL   0   0   0   0   3
    ## 
    ## Overall Statistics
    ##                                         
    ##                Accuracy : 0.9167        
    ##                  95% CI : (0.73, 0.9897)
    ##     No Information Rate : 0.25          
    ##     P-Value [Acc > NIR] : 9.084e-12     
    ##                                         
    ##                   Kappa : 0.8952        
    ##                                         
    ##  Mcnemar's Test P-Value : NA            
    ## 
    ## Statistics by Class:
    ## 
    ##                      Class: CLL Class: AML Class: NoL Class: CML Class: ALL
    ## Sensitivity              1.0000     0.8333     1.0000     1.0000     0.7500
    ## Specificity              0.8947     1.0000     1.0000     1.0000     1.0000
    ## Pos Pred Value           0.7143     1.0000     1.0000     1.0000     1.0000
    ## Neg Pred Value           1.0000     0.9474     1.0000     1.0000     0.9524
    ## Precision                0.7143     1.0000     1.0000     1.0000     1.0000
    ## Recall                   1.0000     0.8333     1.0000     1.0000     0.7500
    ## F1                       0.8333     0.9091     1.0000     1.0000     0.8571
    ## Prevalence               0.2083     0.2500     0.2083     0.1667     0.1667
    ## Detection Rate           0.2083     0.2083     0.2083     0.1667     0.1250
    ## Detection Prevalence     0.2917     0.2083     0.2083     0.1667     0.1250
    ## Balanced Accuracy        0.9474     0.9167     1.0000     1.0000     0.8750

So we got 100% overall accuracy in the training data and 92% overall
accuracy in the testing data.

## Visualization

Finally it is recommended to visualize the rules in the training and
testing data. This will give you an idea to which level the rules and
predictions are good.

We can plot binary heatmap plots through `plot_binary_TSP` function as
follows:

``` r
# plot for the rules and scores in the training data
plot_binary_TSP(Data = object, # we are using the data object here
                classifier = classifier, 
                prediction = results_train, 
                classes = c("ALL","AML","CLL","CML","NoL"),
                #margin = c(0,5,0,10),
                title = "Training data")
```

<img src="vignettes/figure/unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

``` r
# plot for the rules and scores in the testing data
plot_binary_TSP(Data = test, # ExpressionSet
                ref = "LeukemiaType", # ref label names in pData
                classifier = classifier, 
                prediction = results_test, 
                classes = c("ALL","AML","CLL","CML","NoL"),
                title = "Testing data"#, 
                #margin = c(0,5,0,10)
                )
```

<img src="vignettes/figure/unnamed-chunk-15-2.png" style="display: block; margin: auto;" />

# Random Forest scheme

In Random Forest (RF) scheme, all steps of gene filtering/sorting, rule
filtering/sorting, and final model training are performed using the RF
algorithm.

The `ranger` package is used to run the RF algorithm, and arguments can
be passed to `ranger` function if any changes are needed. For example,
increasing the number of trees. Check the documentation of
`multiclassPairs` for more details.

## Gene sorting

To reduce the number of the genes, `sort_genes_RF` function sorts the
genes based on their importance and take the top important genes. Gene
importance is determined based on the ability of the gene to split the
classes better than other genes. Check
[`ranger`](https://cran.r-project.org/package=ranger) package for more
details.

To get the important (i.e. informative) genes `sort_genes_RF` function
performs two types of sorting genes, first type is “altogether” which
runs the RF algorithm to sort the genes based on their importance in all
classes from each other, this generates one list of sorted genes for the
whole data. The second type is “one\_vs\_rest” which runs the RF
algorithm to sort the genes based on their importance in splitting one
class from the rest, this generates sorted list of genes for each class.
It is recommended (default) to run both, particularly in case of class
imbalance in the training data.

By default, all genes are sorted and returned, and the user can specify
how many top genes will be used in the downstream functions. However,
the user can specify how many genes should be returned using
`featureNo_altogether` and `featureNo_one_vs_rest` arguments, and if one
of these arguments is 0, then that sorting type will be skipped.

Like One-vs-rest scheme, platform-wise option is available for the
Random Forest scheme where the genes and rules can be sorted in each
platform/study individually then the top genes in all of them will be
taken. This gives more weight to small platforms/studies and try to get
genes that function equally well in all platforms/studies.

It is recommended to run all function in the RF scheme with seeds to get
reproducible results each time.

Let us run gene sorting function for the previous leukemia training
data. Note that the data is not ranked by default here, but you can rank
the data (ex. in case you are using non-normalized data) before sorting
the genes using `rank_data`, the genes then will be ranked within each
sample.

``` r
# (500 trees here just for fast example)
genes_RF <- sort_genes_RF(data_object = object,
                          # featureNo_altogether, it is better not to specify a number here
                          # featureNo_one_vs_rest, it is better not to specify a number here
                          rank_data = TRUE,
                          platform_wise = FALSE,
                          num.trees = 500, # more features, more tress are recommended
                          seed=123456, # for reproducibility
                          verbose = TRUE)
genes_RF # sorted genes object
```

    ## multiclassPairs - Random Forest scheme
    ##   Object contains:
    ##       - sorted_genes 
    ##          - class: all : 1000 genes
    ##          - class: CLL : 1000 genes
    ##          - class: AML : 1000 genes
    ##          - class: NoL : 1000 genes
    ##          - class: CML : 1000 genes
    ##          - class: ALL : 1000 genes
    ##       - RF_classifiers 
    ##       - calls 
    ##             sort_genes_RF(data_object = object,
    ##              rank_data = TRUE,
    ##              platform_wise = FALSE,
    ##              num.trees = 500,
    ##              verbose = TRUE,
    ##              seed = 123456)

## Rule sorting

After we sorted the genes, we need to take the top genes and combine
them as binary rules and then sort these rules. This can be performed
using `sort_rules_RF` function. Here, we need to determine how many top
genes to use.

Like sorting genes options, rules sorting can be performed in
“altogether”, “one\_vs\_rest”, and “platform\_wise” options. By default,
both “altogether” and “one\_vs\_rest” rule sorting are performed.

**Parameter optimization:**

The `summary_genes_RF` function is an optional function in the workflow.
`summary_genes_RF` function gives an idea of how many genes you need to
use to generate a specific number of rules.

The `summary_genes_RF` function checks different values of
`genes_altogether` and `genes_one_vs_rest` arguments in `sort_rules_RF`
function. `summary_genes_RF` works as follows: take the first element in
`genes_altogether` and `genes_one_vs_rest` arguments, then bring this
number of top genes from altogether slot and one\_vs\_rest slots (this
number of genes will be taken from each class), respectively, from the
sorted\_genes\_RF object. Then pool the extracted genes and take the
unique genes. Then calculate the number of the possible combinations.
Store the number of unique genes and rules in first row in the output
data.frame then pick the second element from the `genes_altogether` and
`genes_one_vs_rest` and repeat the steps again. NOTE: gene replication
in rules is not considered, because the rules are not sorted yet in the
sorted\_genes\_RF object.

Another way, but computationally more extensive, is to use large and
sufficient number of genes, this will produce large number of rules,
then we can use `run_boruta=TRUE` argument in the training function
(i.e. `train_RF` function), this will remove the uninformative rules
before training the final model.

Let us run an example for `summary_genes_RF` function to get an idea of
how many genes we will use, then we run `sort_rules_RF` function.

``` r
# to get an idea of how many genes we will use
# and how many rules will be generated
summary_genes <- summary_genes_RF(sorted_genes_RF = genes_RF,
                                  genes_altogether = c(10,20,50,100,150,200),
                                  genes_one_vs_rest = c(10,20,50,100,150,200))
knitr::kable(summary_genes)
```

| genes\_altogether\_in\_object | genes\_1\_vs\_r\_in\_object | n\_classes | from\_altogether | from\_one\_vs\_rest | n\_unique\_genes | n\_rules |
|------------------------------:|----------------------------:|-----------:|-----------------:|--------------------:|-----------------:|---------:|
|                          1000 |                        1000 |          5 |               10 |                  10 |               50 |     1225 |
|                          1000 |                        1000 |          5 |               20 |                  20 |              100 |     4950 |
|                          1000 |                        1000 |          5 |               50 |                  50 |              229 |    26106 |
|                          1000 |                        1000 |          5 |              100 |                 100 |              422 |    88831 |
|                          1000 |                        1000 |          5 |              150 |                 150 |              579 |   167331 |
|                          1000 |                        1000 |          5 |              200 |                 200 |              695 |   241165 |

``` r
# 50 genes_altogether and 50 genes_one_vs_rest seems 
# to give enough number of  rules and unique genes for our classes
# (500 trees here just for fast example)
# Now let's run sort_rules_RF to create the rules and sort them
rules_RF <- sort_rules_RF(data_object = object, 
                          sorted_genes_RF = genes_RF,
                          genes_altogether = 50,
                          genes_one_vs_rest = 50, 
                          num.trees = 500,# more rules, more tress are recommended 
                          seed=123456,
                          verbose = TRUE)
rules_RF # sorted rules object
```

    ## multiclassPairs - Random Forest scheme
    ##   Object contains:
    ##       - sorted_genes 
    ##          - class: all : 50 genes
    ##          - class: CLL : 50 genes
    ##          - class: AML : 50 genes
    ##          - class: NoL : 50 genes
    ##          - class: CML : 50 genes
    ##          - class: ALL : 50 genes
    ##       - sorted_rules 
    ##          - class: all : 26106  sorted rules
    ##          - class: CLL : 26106  sorted rules
    ##          - class: AML : 26106  sorted rules
    ##          - class: NoL : 26106  sorted rules
    ##          - class: CML : 26106  sorted rules
    ##          - class: ALL : 26106  sorted rules
    ##       - RF_classifiers 
    ##       - calls 
    ##             sort_rules_RF(data_object = object,
    ##              sorted_genes_RF = genes_RF,
    ##              genes_altogether = 50,
    ##              genes_one_vs_rest = 50,
    ##              num.trees = 500,
    ##              verbose = TRUE,
    ##              seed = 123456)

## Model training

Now, we have the rules sorted based on their importance. Now we can
train our final RF model. We can go with the default settings in the
`train_RF` function directly or we can check some different parameters
to optimize the training process by running `optimize_RF` as in the next
example:

``` r
# prepare the simple data.frame for the parameters I want to test
# names of arguments as column names
# this df has three sets (3 rows) of parameters
parameters <- data.frame(
  gene_repetition=c(3,2,1),
  rules_one_vs_rest=c(2,3,10),
  rules_altogether=c(2,3,10),
  run_boruta=c(FALSE,"make_error",TRUE), # I want to produce error in the 2nd trial
  plot_boruta = FALSE,
  num.trees=c(100,200,300),
  stringsAsFactors = FALSE)

# parameters
# for overall and byclass possible options, check the help files
para_opt <- optimize_RF(data_object = object,
                        sorted_rules_RF = rules_RF,
                        parameters = parameters,
                        test_object = NULL,
                        overall = c("Accuracy","Kappa"), # wanted overall measurements 
                        byclass = c("F1"), # wanted measurements per class
                        verbose = TRUE)

para_opt # results object
# para_opt$summary # the df of with summarized information
knitr::kable(para_opt$summary)
```

Here, the 3rd trial pronounced good accuracy.

It is recommend to run Boruta option (`run_boruta=TRUE`) to remove the
uniformative rules, thus use less genes and rules in the model. Note:
When `run_boruta=TRUE` the training process may take long time if the
number of rules is large.

It is recommended to train the final model with `probability=TRUE`, this
will allow the model to give additional scores for each class instead of
class prediction without scores.

We recommend that you visit the documentation of `train_RF` and `ranger`
functions and to take a look over the different arguments and options
there.

``` r
# train the final model
# it is preferred to increase the number of trees and rules in case you have
# large number of samples and features
# for quick example, we have small number of trees and rules here
# based on the optimize_RF results we will select the parameters
RF_classifier <- train_RF(data_object = object,
                          sorted_rules_RF = rules_RF,
                          gene_repetition = 1,
                          rules_altogether = 10,
                          rules_one_vs_rest = 10,
                          run_boruta = TRUE, 
                          plot_boruta = FALSE,
                          probability = TRUE,
                          num.trees = 300,
                          boruta_args = list(),
                          verbose = TRUE)
```

    ## Classes
    ## ALL AML CLL CML NoL 
    ##   8   6   7   8   7

**Training data - quality check:** From the trained model we can extract
a proximity matrix which is based on the fraction of times any two given
out-of-bag samples receive the same predicted label in each tree. This
gives us a better overview of the predictor performance in the training
data, and the behavior of our reference labels. We can generate heatmap
for the proximity matrix by `proximity_matrix_RF` function as follows:

``` r
# plot proximity matrix of the out-of-bag samples
# Note: this takes a lot of time if the data is big
proximity_matrix_RF(object = object,
             classifier = RF_classifier, 
             plot = TRUE,
             return_matrix = FALSE, # if we need to get the matrix itself
             title = "Leukemias",
             cluster_cols = TRUE)
```

<img src="vignettes/figure/unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

## Training Accuracy

For the training accuracy, we do not apply the RF model on the training
data, because this gives 100% accuracy (almost always) which is not
reliable. Instead we use the out-of-bag predictions, which can be
obtained as follows:

``` r
# training accuracy
# get the prediction labels from the trained model
# if the classifier trained using probability   = FALSE
training_pred <- RF_classifier$RF_scheme$RF_classifier$predictions
if (is.factor(training_pred)) {
  x <- as.character(training_pred)
}

# if the classifier trained using probability   = TRUE
if (is.matrix(training_pred)) {
  x <- colnames(training_pred)[max.col(training_pred)]
}

# training accuracy
caret::confusionMatrix(data =factor(x),
                       reference = factor(object$data$Labels),
                       mode = "everything")
```

    ## Confusion Matrix and Statistics
    ## 
    ##           Reference
    ## Prediction ALL AML CLL CML NoL
    ##        ALL   8   0   0   0   0
    ##        AML   0   6   0   0   0
    ##        CLL   0   0   7   0   0
    ##        CML   0   0   0   8   0
    ##        NoL   0   0   0   0   7
    ## 
    ## Overall Statistics
    ##                                      
    ##                Accuracy : 1          
    ##                  95% CI : (0.9026, 1)
    ##     No Information Rate : 0.2222     
    ##     P-Value [Acc > NIR] : < 2.2e-16  
    ##                                      
    ##                   Kappa : 1          
    ##                                      
    ##  Mcnemar's Test P-Value : NA         
    ## 
    ## Statistics by Class:
    ## 
    ##                      Class: ALL Class: AML Class: CLL Class: CML Class: NoL
    ## Sensitivity              1.0000     1.0000     1.0000     1.0000     1.0000
    ## Specificity              1.0000     1.0000     1.0000     1.0000     1.0000
    ## Pos Pred Value           1.0000     1.0000     1.0000     1.0000     1.0000
    ## Neg Pred Value           1.0000     1.0000     1.0000     1.0000     1.0000
    ## Precision                1.0000     1.0000     1.0000     1.0000     1.0000
    ## Recall                   1.0000     1.0000     1.0000     1.0000     1.0000
    ## F1                       1.0000     1.0000     1.0000     1.0000     1.0000
    ## Prevalence               0.2222     0.1667     0.1944     0.2222     0.1944
    ## Detection Rate           0.2222     0.1667     0.1944     0.2222     0.1944
    ## Detection Prevalence     0.2222     0.1667     0.1944     0.2222     0.1944
    ## Balanced Accuracy        1.0000     1.0000     1.0000     1.0000     1.0000

## Prediction

We predict the class in the test samples by running the `predict_RF`
function. Note that you can use `impute = TRUE` to handle missed genes
in the test data. This is done by filling the missed rule values based
on the closest samples in the training data. RF models are good enough
to handle large percent of missed rules (up to 50% in some cases)
without large effect on the accuracy of the prediction.

``` r
# apply on test data
results <- predict_RF(classifier = RF_classifier, 
                      Data = test,
                      impute = TRUE) # can handle missed genes by imputation

# get the prediction labels
# if the classifier trained using probability   = FALSE
test_pred <- results$predictions
if (is.factor(test_pred)) {
  x <- as.character(test_pred)
}

# if the classifier trained using probability   = TRUE
if (is.matrix(test_pred)) {
  x <- colnames(test_pred)[max.col(test_pred)]
}

# training accuracy
caret::confusionMatrix(data = factor(x),
                       reference = factor(pData(test)[,"LeukemiaType"]),
                       mode = "everything")
```

    ## Confusion Matrix and Statistics
    ## 
    ##           Reference
    ## Prediction ALL AML CLL CML NoL
    ##        ALL   3   0   0   0   0
    ##        AML   0   4   0   0   0
    ##        CLL   1   1   5   0   0
    ##        CML   0   1   0   4   0
    ##        NoL   0   0   0   0   5
    ## 
    ## Overall Statistics
    ##                                           
    ##                Accuracy : 0.875           
    ##                  95% CI : (0.6764, 0.9734)
    ##     No Information Rate : 0.25            
    ##     P-Value [Acc > NIR] : 2.032e-10       
    ##                                           
    ##                   Kappa : 0.8435          
    ##                                           
    ##  Mcnemar's Test P-Value : NA              
    ## 
    ## Statistics by Class:
    ## 
    ##                      Class: ALL Class: AML Class: CLL Class: CML Class: NoL
    ## Sensitivity              0.7500     0.6667     1.0000     1.0000     1.0000
    ## Specificity              1.0000     1.0000     0.8947     0.9500     1.0000
    ## Pos Pred Value           1.0000     1.0000     0.7143     0.8000     1.0000
    ## Neg Pred Value           0.9524     0.9000     1.0000     1.0000     1.0000
    ## Precision                1.0000     1.0000     0.7143     0.8000     1.0000
    ## Recall                   0.7500     0.6667     1.0000     1.0000     1.0000
    ## F1                       0.8571     0.8000     0.8333     0.8889     1.0000
    ## Prevalence               0.1667     0.2500     0.2083     0.1667     0.2083
    ## Detection Rate           0.1250     0.1667     0.2083     0.1667     0.2083
    ## Detection Prevalence     0.1250     0.1667     0.2917     0.2083     0.2083
    ## Balanced Accuracy        0.8750     0.8333     0.9474     0.9750     1.0000

So we got 100% overall accuracy in the training data and 88% overall
accuracy in the testing data.

## Visualization

`plot_binary_RF` can be used to plot binary heatmaps for the rules in
training and testing datasets, as follow:

``` r
#visualize the binary rules in training dataset
plot_binary_RF(Data = object,
               classifier = RF_classifier,
               prediction = NULL, 
               as_training = TRUE, # to extract the scores from the model
               show_scores = TRUE,
               top_anno = "ref",
               show_predictions = TRUE, 
               #margin = c(0,5,0,8),
               title = "Training data")
```

<img src="vignettes/figure/unnamed-chunk-25-1.png" style="display: block; margin: auto;" />

``` r
# visualize the binary rules in testing dataset
plot_binary_RF(Data = test,
               ref = "LeukemiaType", # Get ref labels from the test ExpressionSet
               classifier = RF_classifier,
               prediction = results, 
               as_training = FALSE, 
               show_scores = TRUE,
               top_anno = "ref",
               show_predictions = TRUE,
               title = "Testing data")
```

<img src="vignettes/figure/unnamed-chunk-25-2.png" style="display: block; margin: auto;" />

# Disjoint rules

`multiclassPairs` allows the user to select if the rules should be
disjoint or not in the final model. This means if the gene is allowed to
be repeated in the rules or not. For the one-vs-rest scheme, we pass the
`disjoint` argument to switchBox package to allow/prevent the gene
repetition in the rules. In the random forest scheme, we give the user
more control over the gene repetition where the `gene_repetition`
argument in the `train_RF` function allows the user to determine how
many times the gene is allowed to be repeated in the rules. When
`gene_repetition` = 1 the produced classifier will have disjointed
rules.

# Comparison

Here, we compare the runtime and accuracy of the two schemes in
`multiclassPairs` (one-vs-rest and random forest) with the multiclass
pair-based decision tree (DT) implementation in `Rgtsp` package
(Popovici et al., 2011).

We used the breast cancer training data ran the three approaches
subsetting different training cohort sizes: 50, 100, 200, 500, 1000, and
1500 samples. The used settings in the three approaches are shown in the
next chunk. We repeated the training process 5 times and recorded the
training time for the full pipeline (i.e. gene and rule filtering and
model training) and the accuracy. We used two different computational
settings: 4 cores/64GB of RAMs and 24 cores/128GB of RAMs. The used test
data contained (n=1634 samples).

We found that the DT models had the lowest accuracy and the highest
training time regardless of the training dataset size (Figure 5). RF
models with training sample sizes above 50 outperformed both one-vs-rest
and the DT models (Figure 5). One-vs-rest models showed the lowest
training time with the small training datasets, while RF approach showed
the lowest training time in the model with 1500 training samples (Figure
6).

``` r
# For one-vs-rest scheme settings: 
# one_vs_rest gene filtering with k_range of 10:50
object <- ReadData(Data = training_data, 
                   Labels = Train_Classes)

filtered_genes <- filter_genes_TSP(data_object = object,
                                   featureNo = 1000, UpDown = TRUE,
                                   filter = "one_vs_rest")

classifier_1_vs_r <- train_one_vs_rest_TSP(data_object = object, 
                                           filtered_genes = filtered_genes,
                                           k_range = 10:50, disjoint = TRUE,
                                           include_pivot = FALSE, 
                                           platform_wise_scores = FALSE,
                                           one_vs_one_scores = FALSE)                                          

# For RF scheme settings: 
# Settings represents a large model to show calculation time with high number of genes/rules
object <- ReadData(Data = training_data, 
                   Labels = Train_Classes)

sorted_genes  <- sort_genes_RF(object)

sorted_rules  <- sort_rules_RF(data_object = object, 
                               sorted_genes_RF = sorted_genes, 
                               genes_altogether = 200, 
                               genes_one_vs_rest = 200)

classifier_RF <- train_RF(data_object = object, 
                          sorted_rules_RF = sorted_rules, 
                          rules_one_vs_rest = 200,
                          rules_altogether = 200, 
                          run_boruta = FALSE, gene_repetition = 1)

# For DT scheme from Rgtsp package: 
# default settings with min.score=0.6 (to avoid running out of rules with the default setting 0.75)
#devtools::install_github("bioinfo-recetox/Rgtsp")
library(Rgtsp)
classifier_DT <- mtsp(X = t(as.matrix(training_data)),
                      y = Train_Classes,
                      min.score=0.60)
```

<div class="figure" style="text-align: center">

<img src="vignettes/figure/Accuracy_final.png" alt="Overall accuracy for the three pair-based approaches. X-axis show the performance of different models trained on training datasets with different sizes." width="5729" />
<p class="caption">
Overall accuracy for the three pair-based approaches. X-axis show the
performance of different models trained on training datasets with
different sizes.
</p>

</div>

<div class="figure" style="text-align: center">

<img src="vignettes/figure/Both_CPUs.png" alt="Average of the overall training time including the gene and rules filtering and model training. Training repeated 5 times for each model and the bars show the average time." width="6929" height="90%" />
<p class="caption">
Average of the overall training time including the gene and rules
filtering and model training. Training repeated 5 times for each model
and the bars show the average time.
</p>

</div>

# Session info

``` r
sessionInfo()
```

    ## R version 4.0.3 (2020-10-10)
    ## Platform: x86_64-w64-mingw32/x64 (64-bit)
    ## Running under: Windows 10 x64 (build 19042)
    ## 
    ## Matrix products: default
    ## 
    ## locale:
    ## [1] LC_COLLATE=English_United States.1252 
    ## [2] LC_CTYPE=English_United States.1252   
    ## [3] LC_MONETARY=English_United States.1252
    ## [4] LC_NUMERIC=C                          
    ## [5] LC_TIME=English_United States.1252    
    ## 
    ## attached base packages:
    ## [1] parallel  stats     graphics  grDevices utils     datasets  methods  
    ## [8] base     
    ## 
    ## other attached packages:
    ## [1] leukemiasEset_1.26.0  Biobase_2.50.0        BiocGenerics_0.36.0  
    ## [4] multiclassPairs_0.4.1 knitr_1.30            BiocStyle_2.18.1     
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_1.0.6           lubridate_1.7.9.2    lattice_0.20-41     
    ##  [4] png_0.1-7            gtools_3.8.2         class_7.3-17        
    ##  [7] assertthat_0.2.1     digest_0.6.27        ipred_0.9-9         
    ## [10] foreach_1.5.1        R6_2.5.0             ranger_0.12.1       
    ## [13] plyr_1.8.6           switchBox_1.26.0     stats4_4.0.3        
    ## [16] evaluate_0.14        e1071_1.7-4          highr_0.8           
    ## [19] ggplot2_3.3.3        pillar_1.4.7         gplots_3.1.1        
    ## [22] rlang_0.4.10         caret_6.0-86         data.table_1.13.6   
    ## [25] rpart_4.1-15         Matrix_1.3-2         rmarkdown_2.6       
    ## [28] splines_4.0.3        gower_0.2.2          stringr_1.4.0       
    ## [31] munsell_0.5.0        compiler_4.0.3       xfun_0.20           
    ## [34] pkgconfig_2.0.3      Boruta_7.0.0         htmltools_0.5.1.1   
    ## [37] nnet_7.3-14          tidyselect_1.1.0     tibble_3.0.5        
    ## [40] prodlim_2019.11.13   codetools_0.2-18     dunn.test_1.3.5     
    ## [43] crayon_1.3.4         dplyr_1.0.3          withr_2.4.1         
    ## [46] bitops_1.0-6         MASS_7.3-53          recipes_0.1.15      
    ## [49] ModelMetrics_1.2.2.2 grid_4.0.3           nlme_3.1-151        
    ## [52] gtable_0.3.0         lifecycle_0.2.0      DBI_1.1.1           
    ## [55] magrittr_2.0.1       pROC_1.17.0.1        scales_1.1.1        
    ## [58] KernSmooth_2.23-18   stringi_1.5.3        reshape2_1.4.4      
    ## [61] timeDate_3043.102    ellipsis_0.3.1       generics_0.1.0      
    ## [64] vctrs_0.3.6          rdist_0.0.5          lava_1.6.8.1        
    ## [67] iterators_1.0.13     tools_4.0.3          glue_1.4.2          
    ## [70] purrr_0.3.4          survival_3.2-7       yaml_2.2.1          
    ## [73] colorspace_2.0-0     BiocManager_1.30.10  caTools_1.18.1
