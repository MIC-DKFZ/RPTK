
<img src="img/rptk_logo.png" align="right" width="150"/>

[![Multi Section](https://img.shields.io/badge/RPTK-v1.0-purple?logo=github)](https://git.dkfz.de/mic/personal/group3/jonasb/rptk/)
[![Logo Tool](https://img.shields.io/badge/Python-3.9_%7C_3.10_%7C_3.11-blue?)](https://github.com/username/mytool)
[![Logo Tool](https://img.shields.io/badge//Operating_Systems-Linux-re?)](https://github.com/username/mytool)


<br />
<br />

# RPTK (Radiomics Processing Toolkit)

**RPTK** is a comprehensive, standardized pipeline for radiomics data processing. This toolkit is designed to facilitate the end-to-end workflow for radiomics analysis, from image transformation and segmentation to feature extraction, stability analysis, and model application. By consolidating multiple radiomics processing stages into a single framework, RPTK streamlines the generation of high-quality, reproducible radiomics features that are ready for downstream analyses and model development.

## Key Features

1. **Image Transformation**  
   RPTK standardizes images and prepares them for radiomics analysis, ensuring compatibility and consistency across different datasets and sets the same data basis for the applied radiomics feature extractors.

2. **Segmentation Perturbation and Segmentation Filtering**  
   RPTK handles segmentation variability, enabling robust feature extraction by adjusting segmentation parameters and examining the impact on radiomics features. Filter segmentations for artifacts by identifying and removing additional connected components.

3. **Feature Extraction**  
   RPTK utilizes **PyRadiomics** and **MIRP** (Medical Image Radiomics Platform) to extract high-dimensional radiomics features, capturing texture, intensity, and shape-based information from medical images.

4. **Feature Stability Analysis**  
   RPTK evaluates the stability of extracted features across perturbations and transformations to ensure reliable, reproducible metrics that can be confidently used in further analysis.

5. **Feature Selection**  
   RPTK implements different feature selection methods, focusing on identifying and retaining the most robust features to enhance model performance and reduce overfitting. By default, sequential feature selection with a random forest classifier selecting the best 10 features forward and backward will be applied.

6. **Automated Model Training and Optimization**  
   RPTK includes six different machine learning models that can be applied directly to the processed radiomics data, providing a streamlined path from feature extraction to predictive modeling. Model pretraining, cross-validation-based optimization and model ensembling ensure stable performance.
   
7. **Automated Model Evaluation**  
   RPTK selects the best model based on the best mean validation AUROC and performs 1000x bootstrapping to calculate 95CI Test AUROC to measure the convidence. Decision thresholds get automatically optimized by youden index calculation to show optimal sensitivity and specificity performance. 

## Getting Started

To get started with RPTK, clone this repository and follow the installation instructions in `docs/INSTALL.md`.
Alternatively we are offering a docker image to run independently on your machine. For docker execution please see `docs/Docker/README.md` for detailed instructions.

## Usage

RPTK is built to handle every stage of radiomics processing, providing researchers and clinicians with a powerful, standardized toolkit. By leveraging RPTK, users can quickly progress from raw medical images to trained models, minimizing the complexities of radiomics processing. Take a look at `docs/USAGE.md` for more. For example scripts and usable scripts with example input data configuration, please look at the `example/` folder. If you want to compare your radiomics tool against deep learning model perforance check the `example/DeepLearning` folder. For application of RPTK, we recommend to use `run-rptk.py` or `example/run_rptk.py` script including argpaser for better application of RPTK.

We recommend using datasets with a **minimum of 120 samples** for application.  
The required data size can vary depending on the heterogeneity of the data and the complexity of the task.  

To estimate whether the dataset size is sufficient, we recommend using empirical [Learning Curve Analysis](https://scikit-learn.org/stable/auto_examples/model_selection/plot_learning_curve.html).  
A learning curve shows how model performance (e.g., accuracy, error, Dice score) changes as the training set size increases.  
If the curve plateaus early, additional samples may not significantly improve performance.  
If the curve is still rising, more data will likely be beneficial.  

If you want to use this tool, please cite us:

`Bohn, J.R. et al. (2023). RPTK: The Role of Feature Computation on Prediction Performance, Medical Image Computing and Computer Assisted Intervention – MICCAI 2023 Workshops. MICCAI 2023. Lecture Notes in Computer Science, vol 14394. Springer, Cham. DOI: 10.1007/978-3-031-47425-5_11`

---

## Input CSV Configuration

To ensure smooth processing with RPTK, your input data file must be configured in a specific format. This CSV file will contain both mandatory metadata and optional fields for each sample, enabling RPTK to handle images, segmentations, feature data, and additional annotations. Here’s the required structure:

### Required Columns

These columns include the required information for the pipeline to run. Please also follow the format recommendations to avoid format-related errors.

1. **ID**  
   - A unique identifier for each sample.
   - Must be a string or number.
   - Forbidden characters: `_` (underscore).

2. **Image**  
   - Path to the image file for each sample.
   - This should be the full path or a relative path accessible by RPTK.
   - Files should be in .nii.gz, .nii, or .nrrd format

3. **Mask**  
   - Path to the segmentation mask file corresponding to the Region of Interest (ROI) in each sample.
   - Must align with the same file structure as the `Image` column.
   - Files should be in .nii.gz, .nii, or .nrrd format

4. **Modality**  
   - Specifies the imaging modality.
   - Accepted values:
     - `"CT"` for Computed Tomography images.
     - `"MR"` for Magnetic Resonance images.

5. **Prediction_Label**  
   - The target label used for prediction tasks.
   - Should contain the ground truth label or class information for each sample, required for supervised model training.
   - Currently only binary preditions can be performed.

### Optional Columns

These columns include information that is only required if the label in the mask is not always the same (if the label is not always 1), if you have logitudinal data, or if you have segmentations from different segmentators. 

1. **ROI_Label**  
   - Use this column if your segmentation includes multiple labels.
   - Specifies the specific region of interest label within the segmentation for each sample.
   - **Note**:  RPTK assumes to take label 1 in the segmentation if noting is specified

2. **Timepoint**  
   - Use this column for time series data.
   - Specifies the time step associated with each sample, allowing RPTK to analyze temporal changes.
   - If you are considering calculating delta radiomics features, you should set the timepoint as a number
   - RPTK handles the prediction label between time points by default as follows: if the labels between the time points defer, the label of the later timepoint is set as the label for the delta calculation
   - If you want to consider temporal changes from one state to the other, set take_label_changes to True. If the label changes over time from 0 to 1, the label is -1 if the label changes from 1 to 0 the label is 2. If the label stays, it is considered as a single label.

3. **Rater**  
   - Use this column if your samples have been segmented by multiple raters.
   - Specifies the identifier for each rater, enabling feature stability filtering across different segmentations.
   - **Note**: For ground truth samples, leave the `Rater` column empty, and provide only the segmentation for the chosen ground truth rater. Add separate rows for each additional rater's segmentation file paired with the same image.
   
4.  **Additional Clinical Parameters**  
   - Additional measurements per sample can be easily included in the corresponding row with the correct ID
   - `Important`: Parameter names should not contain "index" as this can lead to confusion within the pipeline, and the parameter might get ignored

### Example Configuration

For a detailed description on how to use RPTK and configure the input CSV file `docs/USAGE.md`. 

Your input CSV might look something like this:

| ID    | Image                    | Mask                    | Modality | Prediction_Label | ROI_Label | Timepoint | Rater |
|-------|---------------------------|-------------------------|----------|------------------|-----------|-----------|-------|
| ID-001   | /path/to/image1.nii.gz    | /path/to/mask1.nii.gz  | CT       | 1                | 1     | 0         |       |
| ID-001   | /path/to/image1.nii.gz    | /path/to/rater2_mask1.nii.gz | CT | 1 | 1 | 0 | Rater2 |
| ID-001   | /path/to/image1.nii.gz    | /path/to/rater3_mask1.nii.gz | CT | 1 | 1 | 0 | Rater3 |
| ID-001   | /path/to/image1.nii.gz    | /path/to/rater4_mask1.nii.gz | CT | 1 | 1 | 0 | Rater4 |
| ID-002   | /path/to/image2.nii.gz    | /path/to/mask2.nii.gz  | CT       | 0                | 1    | 0         |       |
| ID-002   | /path/to/image2.nii.gz    | /path/to/rater3_mask2.nii.gz  | CT       | 0         | 1    | 0         |Rater3 |
| ID-002   | /path/to/image2.nii.gz    | /path/to/rater4_mask2.nii.gz  | CT       | 0         | 1    | 0         |Rater4 |
| ...   | ...                       | ...                     | ...      | ...              | ...       | ...       | ...   |


### Guidelines

- Ensure paths in `Image` and `Mask` columns are valid and accessible.
- Each entry in `Modality` must correspond to a supported modality.
- `Prediction_Label` should be specified for all samples to support model training and evaluation.
- **Optional Columns** (`ROI_label`, `Timepoint`, `Rater`) should only be included as needed for your dataset, but if used, must be filled consistently across samples.

---

This expanded configuration guide ensures that users can take full advantage of RPTK's capabilities, including multi-rater comparisons, time series analysis, and ROI-specific feature extraction.

## References

1. van Griethuysen, J. J. M., Fedorov, A., Parmar, C., *et al.* (2017).
   Computational radiomics system to decode the radiographic phenotype.
   *Cancer Research*, 77(21), e104–e107. https://doi.org/10.1158/0008-5472.CAN-17-0339 (**PyRadiomics**)

2. Zwanenburg, A., Leger, S., Agolli, L., Pilz, K., Troost, E. G. C., Richter, C., & Löck, S. (2019).
   Assessing robustness of radiomic features by image perturbation.
   *Scientific Reports*, 9, 614. https://doi.org/10.1038/s41598-018-36938-4 (**MIRP**)

## Acknowlegement

RPTK is developed and maintained by the [Division of Medical Image Computing](https://www.dkfz.de/en/mic/index.php) at the 
[German Cancer Research Center (DKFZ)](https://www.dkfz.de/en/index.html).
