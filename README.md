**Readme**

These scripts were developed to make PGS calculation in AoU Workbench 2.0 accessible to a wider research community. We tried different tools and present the workflow that balances intuitiveness and resource / cost-efficiency.

**When to use:**

PRS can be calculated using HAIL, however, this method is not scalable to PRS with more than hundreds to a few thousand variants. This plink method is highly scalable. The rate limiting step is loading the genotype files, so the number of variants in your PRS shouldn’t really affect the run time or memory.

**Starting up an app:**

Recommended configuration (for V8): AoU Jupyter n2-highmem-64 64 CPUs, 512GB memory, $4.22 per hour - set AutoStop to 1 hr. Check with V9

High memory is needed for the plink bed files

**Predicted run time:**

**Setting variables:**

*“000\_indicate\_PGSID.ipynb”:* We tried to make this as hands-off as possible; therefore, you only need to set the environmental variables once in *“000\_indicate\_PGSID.ipynb”* and the rest of the scripts can be run sequentially without any hard coding using papermill to execute the notebooks.

**Calculating PGS:**

Once you select your PGSID, you can run the papermill command in *“000\_indicate\_PGSID.ipynb”* which will run the following scripts sequentially.

 *“001\_prepare\_weights\_files\_from\_PGS\_catalog.ipynb”* aligns the PGS scoring files to the AoU bim file format. This includes loading the PGS score file from PGS catalog, lifting over the score file to GRCh38 if needed, preparing SNP IDs to match the .bim file, and finding allele matches from PGS score file in BIM files. This script will report QC tracking which is also saved as “{PGS\_ID}\_qc\_metrics\_{BUILD}.txt.”

*“005\_run\_plink\_bash.ipynb”* runs plink2 in parallel across all of the chromosomes at once; therefore, the time it takes to run chromosomes 1/2 (the largest chromosomes) should be the rate limiting step.

*“010\_combine\_chr\_check\_QC.ipynb”* combines the chromosomes into a single score file and reports QC metrics from PRS construction.

“015\_genetic\_ancestry\_adjustment.ipynb” performs mean and variance adjustment of the PRS as a function of 16 genetic principal components, reducing technical ancestry-related differences in the score distribution – as recommended by the eMERGE consortium. The adjustment model is trained using individuals with WGS data who do not have linked EHR data.

**Output files:**

base\_directory: “/home/jupyter/workspace/workspace-bucket/calculate\_pgs/workflow\_runs”

| Folder / File name | Generated in | Location | Contains |
| --- | --- | --- | --- |
| PGS_calc_param.txt | 000_indicate_PGSID.ipynb | base_directory | PGS ID, genome build, chromosomes parameters |
| {PGS_ID}_hmPOS_{BUILD}.txt.gz | 001_prepare_weights_files_from_PGS_catalog.ipynb | Base_directory/pgs_catalog_weights | Weights file downloaded from pgs catalog |
| {PGS_ID}_plink_score_{BUILD}_prepared.txt | “ “ | Base_directory/ pgs_bim_matched_weights | Prepared weights file |
| {PGS_ID}_qc_metrics_{BUILD}.txt | “ “ | “ “ | QC metrics from weight preparation step |
| plink_bed_parallel.sh | 005_run_plink_bash.ipynb | Base_directory | Bash script for plink |
| latest_run.env | “ “ | Base_directory/plink_results | Run_id, output_path, PGS_ID, genome build |
| ${PGS}_${BUILD}_${RUN_ID} | “ “ | “ “ | Folder with score files and logs per chromosome |
| ${PGS}_combined_chr1_22_${RUN_ID}.csv | 010_combine_chr_check_QC.ipynb | Base_directory/plink_results/${PGS}_${BUILD}_${RUN_ID} | Person id and their score |
| variants_incorporated_by_chr_${PGS}_${RUN_ID}.csv | “ “ | “ “ | Number of variants by chr, total and percent of variants incorporated |
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

1.        Base\_directory/ pgs\_bim\_matched\_weights/{PGS\_ID}\_qc\_metrics\_{BUILD}.txt

a.        percent\_found should be close to 100%

2.        Base\_directory/plink\_results/${PGS}\_${BUILD}\_${RUN\_ID}/ variants\_incorporated\_by\_chr\_${PGS}\_${RUN\_ID}.csv

a.        Percent incorporated should be close to 100%
