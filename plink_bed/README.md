**Readme**

These scripts were developed to make PGS calculation in AoU Workbench 2.0 accessible to a wider research community. We tried different tools and present the workflow that balances intuitiveness and resource / cost-efficiency. A single PGS ID or list of PGS IDs can be supplied for calculation.

**When to use:**

PRS can be calculated using HAIL, however, this method is not scalable to PRS with more than hundreds to a few thousand variants. With larger and larger version releases, HAIL will be less and less effective. This plink method is scalable to more and larger PGS. 

**Starting up an app:**

Recommended configuration (for V8): AoU Jupyter n2-highmem-64 64 CPUs, 512GB memory, $4.22 per hour - set AutoStop to 1 hr. Check with V9

High memory is needed for the plink bed files

**Predicted run time:**
V8: On a AoU Jupyter n2-highmem-64 64 CPUs, 512GB memory, $4.22 per hour [old code]
* ~4 hours for 1 PRS / 1,258,327 variants
* ~6 hours for 6 PRS /  ~6 hours for 6 PRS / 2,764,037 variants
* ~22 hours for 15 PRS / 16,339,291 variants
* Therefore predicted run time (using the machine described above on V8 data) can be estimated roughly as  2.49 + 1.16 * variants_millions + 0.05 * n_pgs

**New Predicted run time:**
V8: On a AoU Jupyter n1-standard-32 32 CPUs, 120GB memory, $1.55 per hour
* 3.5 hrs 2 PRS / 1,797,838 variants
* 3.66 hrs 5 PRS / 2,764,037 variants
* 

**Note:** autostop appears unreliable 

**Setting variables:**

*“000\_indicate\_PGSID.ipynb”:* We tried to make this as hands-off as possible; therefore, you only need to set the environmental variables once in *“000\_indicate\_PGSID.ipynb”* and the rest of the scripts can be run sequentially without any hard coding - using papermill to execute the notebooks. A sample ID file can be provided to indicate who to include in the PGS genetic PC adjustment training vs testing data (see 015 for more info). 

**Calculating PGS:**

Once you select your PGSID (or list of PGS IDs), you can run the papermill command in *“000\_indicate\_PGSID.ipynb”* which will run the following scripts sequentially.

 *“001\_prepare\_weights\_files\_from\_PGS\_catalog.ipynb”* aligns the PGS scoring files to the AoU bim file format. This includes loading the PGS score file from PGS catalog, preparing SNP IDs to match the .bim file, and finding allele matches from PGS score file in BIM files. Multiple PGS are saved in a wide score file for computational efficiency of the next step. This script will report QC tracking which is also saved as “{PGS\_ID}\_qc\_metrics\_GRCh38.txt.” For multiple PGS, a combined QC file is also prepared: "combined\_qc\_metrics\_{pgs\_ids}.txt"

*“005\_run\_plink\_bash.ipynb”* runs plink2 in parallel across all of the chromosomes at once; therefore, the time it takes to run chromosomes 1/2 (the largest chromosomes) should be the rate limiting step. No-mean-imputation and score sums are used. A wide score file is used to compute multiple PGS at once. 

*“010\_combine\_chr\_check\_QC.ipynb”* combines the chromosomes into a single score sum file and reports QC metrics from PRS construction.

“015\_genetic\_ancestry\_adjustment.ipynb” performs mean and variance adjustment of the PRS as a function of 16 genetic principal components, reducing technical ancestry-related differences in the score distribution – as recommended by the eMERGE consortium. The adjustment model is trained using individuals with WGS data who do not have linked EHR data. A sample_id file can also be provided in which anyone not included in the sample_id file will be used to train the adjustment model. 

**Output files:**

base\_directory: “/home/jupyter/workspace/workspace-bucket/calculate\_pgs/workflow\_runs”

| Folder / File name | Generated in | Location | Contains |
| --- | --- | --- | --- |
| PGS_calc_param.txt | 000_indicate_PGSID.ipynb | base_directory | PGS ID, chromosomes parameters |
| {PGS_ID}_hmPOS_GRCh38.txt.gz | 001_prepare_weights_files_from_PGS_catalog.ipynb | Base_directory/pgs_catalog_weights | Weights file downloaded from pgs catalog |
| {PGS_ID}_plink_score_GRCh38_prepared.txt | “ “ | Base_directory/ pgs_bim_matched_weights | Prepared weights file |
| {PGS_ID}_qc_metrics_GRCh38.txt | “ “ | “ “ | QC metrics from weight preparation step |
| combined_qc_metrics_{pgs_ids}.txt | “ “ | “ “ | Combined QC metrics from weight preparation step for multiple PGS |
| plink_bed_parallel.sh | 005_run_plink_bash.ipynb | Base_directory | Bash script for plink |
| latest_run.env | “ “ | Base_directory/plink_results | Run_id, output_path, PGS_ID |
| ${PGS}_GRCh38_${RUN_ID} | “ “ | “ “ | Folder with score files and logs per chromosome |
| ${PGS}_combined_chr1_22_${RUN_ID}.csv | 010_combine_chr_check_QC.ipynb | Base_directory/plink_results/${PGS}_GRCh38_${RUN_ID} | Person id and their score |
| variants_incorporated_by_chr_${PGS}_${RUN_ID}.csv | “ “ | “ “ | Number of variants by chr, total and percent of variants incorporated |
| percent_incorperated_${PGS_IDs}_${RUN\_ID}.csv | “ “ | “ “ | Percent of variants incorporated across multiple PGS |
| pre_adjusted_score_density.png | 015_genetic_ancestry_adjustment.ipynb | “” | Pre genetic ancestry adjustment PGS distribution colored by ancestry groups |
| post_adjusted_score_training_data_density.png | “” | “” | PGS distribution post genetic ancestry adjustment in the training data |
| post_adjusted_score_testing_data_density.png | “” | “” | PGS distribution post genetic ancestry adjustment in the testing data |
| {PGS}_final_adjusted_{RUN_ID}.csv | “” | “” | Final PGS file with person_id, ancestry_pred, raw score, and adjusted_score |

**Q&A:**

**Why not use a workflow?**
Cromwell, Nextflow, and Google Batch typically run tasks on separate worker VMs, which means the large chromosome files would need to be copied, localized, or read remotely from shared storage. That adds extra cost, transfer time, and operational complexity.

For this project, running on the existing VM is a practical workaround because the mounted/local chromosome files are already available there. The main tradeoff is that the VM could stop mid-run. However, we have tested the expected runtime and recommend configuring the app’s autostop settings accordingly to minimize this risk.

**What does a successful run look like?**

There are a few things to check.

1. Base\_directory/ pgs\_bim\_matched\_weights/{PGS\_ID}\_qc\_metrics\_GRCh38.txt
- percent\_found should be close to 100%

2. Base\_directory/plink\_results/${PGS}\_GRCh38\_${RUN\_ID}/ variants\_incorporated\_by\_chr\_${PGS}\_${RUN\_ID}.csv
- Percent incorporated should be close to 100%
3. Base\_directory/plink\_results/${PGS}\_GRCh38\_${RUN\_ID}/percent\_incorperated\_${PGS_IDs}\_${RUN\_ID}.csv
-   Percent incorporated should be close to 100% (easier to read for multiple PGS)


