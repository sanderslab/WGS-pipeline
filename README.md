# Category-based burden test (Werling et al. 2018)

This README contains the run command for the category-based burden test in the Werling et al. (2017). The scripts and materials are distributed in Amazon Machine Images and you can deploy the workflow using Amazon Web Services (AWS). The AMI ID is ami-1eefb064, name 'Werling 2018 to share', and has 80 Gib size. For the instance setting, we recommend m4.large or m4.xlarge.


## Step 1. Annotation

De novo variants will be annotated for genomic regions, functional regions, and functional/conservation scores.

Script: run_annotation.py

Required arguments:
-i, --inputfile = path to file listing variants. Input file format described below.
-o, --outputfile = path and name for output file.
-t, --number_threads = number of threads to use

Input file format:
Variant list file with 8 columns: Chr (chromosome), Start (bp starting coordinate, hg19), End (bp ending coordinate, hg19), Ref (reference allele), Alt (alternative allele), Fam (family ID), Pheno (2 for case and 1 for control), SampleID (sample ID)

```bash
# Example
python run_annotation.py \
-i /home/ec2-user/input/input.DNVs.sscWgs519.txt \
-o /home/ec2-user/annotation/table.DNVs.sscWgs519.txt \
-t 2
```

## Step 2. Variant categorization

The annotated variants will be grouped by annotation categories and individuals. You will find the output file in the categorisation/ folder, named categorisation_mutationSum_final.your_output_tag.txt

Script: run_categorizeVars.py

Required arguments:
-i, --inputfile = path to file listing annotated variants. Input file format described below.
-t, --number_threads = number of threads to use

Optional arguments:
-o, --output_tag = text string that will be used for naming output files, e.g. analysis name or date. Default is 'out'.

Input file format:
Annotated variant list, output from run_annotation.py, above.

```bash
python run_categorizeVars.py \
-i /home/ec2-user/annotation/table.DNVs.sscWgs519.txt \
-o test \
-t 4
```

## Step 3. Filter variant categories

To speed up burden testing in the following step, categories with 0 variants will be removed from the file. Users can also provide a list of category names to remove (-r), e.g. redundant categories based on the annotation scheme. This is optional.

You will find the output file in the categorisation/ folder, named categorisation_mutationSum_final.your_output_tag1.trimmed.your_output_tag2.txt

Script: run_filterCategories.py

Required arguments:
-i, --inputfile = path to file with sample-categorized variants. Input file format described below.

Optional arguments:
-o, --output_tag = text string that will be used for naming output files, e.g. analysis name or date. Default is 'out'.
-n, --number_categories = number of categories in annotation combinations. Default is 66402 under our default annotation scheme.
-r, --remove_cat_file = path to file listing categories to remove from the analysis, e.g. categories known to be redundant to other categories in the current annotation scheme. Default is text string 'no', which will bypass this step and will proceed by only removing categories with 0 variants.

Input file format:
File listing the number of variants per sample that belong to each annotation category. Output file from run_categorizeVars.py, above. One row per sampleID, 3 columns of sample information (SampleID, FamilyID, Pheno [2 for cases, 1 for controls]), and one additional column for each annotation category. Under this default annotation scheme, there are 66402 total combinations of annotation categories.

```bash
python run_filterCategories.py \
-i /home/ec2-user/categorisation/categorisation_mutationSum_final.test.txt \
-n 66402 \
-r /home/ec2-user/reference/categorisation/trimAnnotCombos_170629.txt \
-o test
```

## Step 4. Run burden (binomial) test on annotation categories

Each remaining category after the trimming step 3 will be subject to burden test using the binomial test on total variant counts per category between cases and unaffected controls. You will find the result of burden test in the burden_matrix/ folder, named burdenByAnnotCombo_adj_noPerm_your_output_tag.txt

Script: run_burdenTest.py

Required arguments:
-i, --inputfile = path to file with filtered categories (or the non-filtered version of this file). Input file format described below.
-n, --number_categories = Number of annotation categories included in the input file. Should be the number of columns in the file, minus 3.

Optional arguments:
-o, --output_tag = text string that will be used for naming output files, e.g. analysis name or date. Default is 'out'.
-a, --adjust_file = file specifying adjustment to variant rate, e.g. from regression vs. covariates. Adjustment file format described below. Default is text string 'no', which will bypass this adjustment step.
-p, --perm_option = 'yes' or 'no' to permute within-sibship case-control labels. Default is text string 'no'.

Input file format:
File listing the number of variants per sample that belong to each annotation category, optionally with 0-variant and redundant or user-specified categories removed (trimming these additional categories will speed up run time for the remaining steps). Output file from run_filterCategories.py (or from run_categorizeVars.py), above. One row per sampleID, 3 columns of sample information (SampleID, FamilyID, Pheno [2 for cases, 1 for controls]), and one additional column for each annotation category.

Adjustment file format:
File with 1 row per sampleID listing adjustment factor (e.g. from regression) to multiply by total variant counts. Requires columns named "SampleID" and "AdjustFactor"

```bash
python run_burdenTest.py \
-i /home/ec2-user/categorisation/categorisation_mutationSum_final.trimmed.test.txt \
-n 24494 \
-a /home/ec2-user/reference/burden_matrix/sscWgs519_adjustFile-170927.txt \
-p no \
-c 519 \
-u 519 \
-o test
```

## Step 5. Plot volcano plot of burden results

Volcano plot to display the relative risk for each variant category (x-axis) by the significance of the binomial test (y-axis). You will find a PDF of this plot in the figures/ folder, named volcanoBurden_your_output_tag.pdf

Script: run_volcanoPlot.py

Required arguments:
-i, --inputfile = path to file with burden test results. Input file format described below.

Optional arguments:
-o, --output_tag = text string that will be used for naming output files, e.g. analysis name or date. Default is 'out'.
-m, --min_var = minimum number of variants per category required for plotting of that category. Categories with fewer variants will not be plotted. Default is 10.
-e, --effective_tests = estimated number of effective tests, which will define the genome-wide significance threshold. Threshold is plotted with a red line. Default is 4123, as estimated from analysis of 51801 non-redundant categories (out of 66402) and 519 quartet families.
-p, --pval_col = name of the column listing p-values per category to be used for y-axis of volcano plot. Default is "Binom_p"

Input file format:
File with results from burden test. Output from run_burdenTest.py, above. One row per tested annotation category, with columns named Annotation_combo (annotation category name), Expected_prob (expected proportion of case variants = n cases/n samples), Total_count_raw (number of total variants in each category), Case_count_raw (number of case variants in each category), Con_count_raw (number of control variants in each category), Binom_p_raw (p-value from 2-sided binomial test on case vs. control raw variant counts), Binom_p_1side_raw (p-value from 1-sided binomial test on case vs. control raw variant counts), Unadjusted_relative_risk (ratio of case to control raw variant counts), Total_count_adj (number of total variants in each category, adjusted for covariates), Case_count_adj (number of case variants in each category, adjusted for covariates), Con_count_adj (number of control variants in each category, adjusted for covariates), Binom_p (p-value from 2-sided binomial test on case vs. control adjusted variant counts), Binom_p_1side (p-value from 2-sided binomial test on case vs. control adjusted variant counts), Adjusted_relative_risk (ratio of case to control adjusted variant counts)

```bash
python run_volcanoPlot.py \
-i /home/ec2-user/burden_matrix/burdenByAnnotCombo_adj_noPerm_test.txt \
-m 10 \
-e 4123 \
-p Binom_p \
-o test
```
