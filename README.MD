# Automated Koos Classification of Vestibular Schwannoma

This repository provides jupyter-notebooks for the classifcation of Vestibular Schwannoma based on 3D T1, T2 or both (T1+T2) MRI images.

These scripts were used to create results in the publication:

- Kujawa, A., Dorent, R., Connor, S., Oviedova, A., Okasha, M., Grishchuk, D., Ourselin, S., Paddick, I., Kitchen, N., Vercauteren, T. and Shapey, J., 2022. Automated Koos classification of Vestibular Schwannoma. Frontiers in Radiology, p.4.

Currently, a subset of the MRI images and ground truth Koos grades is kept private because it is used as the test set in the [crossMoDA challenge](https://crossmoda-challenge.ml/). [2]

Therefore, the repository currently includes only a couple of example parcellations and ground truth grades from the crossMoDA training set.

## Content of the repository

### Parcellation files
The starting point of the training pipeline are the MRI images and corresponding manual segmentations of Vestibular Schwannoma. The repository contains the manual segmentations for a few cases in the folder [vs_manual_segmentations](vs_manual_segmentations). The corresponding images, as well as more images and manual segmentations of 242 patients can be found [here](https://doi.org/10.7937/TCIA.9YTJ-5Q73) on TCIA) [3]

Images and segmentations were used as input for the GIF algorithm. GIF is available to the public via a [web interface](http://niftyweb.cs.ucl.ac.uk/program.php?p=GIF), however, currently, the web interface does not allow to provide a tumour mask along with the MRI image to improve registration accuracy by focussing on healthy areas. 

Hence, the original GIF parcellation output is provided in this repository in the folder [parcellation_data/GIF_output](parcellation_data/GIF_output).

Although the tumour mask is provided to GIF, the GIF output does not include a label for the tumour. Therefore, the manual tumour segmentation has to be merged with the GIF output which can be achieved with this jupyter-notebook: [combine_GIF_parcellation_with_tumour_segmentation.ipynb](combine_GIF_parcellation_with_tumour_segmentation.ipynb)

The tumour label will be 300. The merged parcellation with tumour segmentation will be saved in [parcellation_data/GIF_with_tumour_all_classes](parcellation_data/GIF_with_tumour_all_classes).

Since only 9 labels were analysed in the subsequent pipeline steps, the following jupyter-notebook was used to remove all but those 9 labels: [create_multi_class_segmentation_for_unet_input_mergedLabels.ipynb](create_multi_class_segmentation_for_unet_input_mergedLabels.ipynb)

The resulting files will be stored in [parcellation_data/GIF_with_tumour_9classes](parcellation_data/GIF_with_tumour_9classes). The remaining labels are the following:

    0: background
    1: tumour (previously label 300)
    2: pons (previously label 35)
    3: brainstem (previously label 36)
    4: vermal lobules I-V (previously label 72)
    5: vermal lobules VI-VII (previously label 73)
    6: vermal lobules VIII-X (previously label 74)
    7: Right Cerebellum (previously label 39 and 41)
    8: Left Cerebellum (previously label 40 and 42)

Note that label 7 and 8 each are a union of 2 other labels.

In the inference path of the pipeline, GIF is not used at all because it takes too long. Instead, a U-Net trained within the nnU-Net framework [4] is used to predict the 9-class segmentations. Currently, the trained model weights are not available in the repository, hence the predicted output of the 3 models (one for each type of input modality) are provided in the folders [parcellation_data/GIF_UNet_prediction_9_classes_T1](parcellation_data/GIF_UNet_prediction_9_classes_T1), [parcellation_data/GIF_UNet_prediction_9_classes_T2](parcellation_data/GIF_UNet_prediction_9_classes_T2) and [parcellation_data/GIF_UNet_prediction_9_classes_T1T2](parcellation_data/GIF_UNet_prediction_9_classes_T1T2).

### Extraction of handcrafted features for random forest

jupyter-notebooks and data that are specific to the random forest branch of the pipeline are in the folder [random_forest_classification](random_forest_classification).

The first step of the random forest branch is to extract handcrafted features from the parcellations. During training this is done based on the GIF output, whereas at inference the fast nnU-Net output is used. The following jupyter-notebook is used to extract features:
[GIF_feature_extraction_by_path_9class.ipynb](random_forest_classification/GIF_feature_extraction_by_path_9class.ipynb)

For each of the labels, 3 features are extracted: volume, distance to the tumour, contact surface with the tumour. A subset of these features is used in the random forest. The extracted features are saved as CSV files in the folder [features](random_forest_classification/features).

At the beginning of the notebook, the input path to the parcellations and the output path of the CSV files has to be defined. For training, the input path should be [parcellation_data/GIF_with_tumour_9classes](parcellation_data/GIF_with_tumour_9classes) and the output path should be [random_forest_classification/features/extracted_from_GIF_output](random_forest_classification/features/extracted_from_GIF_output). For inference, the input path should be [parcellation_data/GIF_UNet_prediction_9_classes_T1](parcellation_data/GIF_UNet_prediction_9_classes_T1) and the output path [random_forest_classification/features/extracted_from_nnUNet_prediction/T1](random_forest_classification/features/extracted_from_nnUNet_prediction/T1) for the parcellations predicted based on T1 images, and modified accordingly for T2 and T1+T2 input modalities.

### Random forest training/inference
Once features are extracted, the following jupyter-notebook can be used to train and infer with the random forest:
[random_forest_classification/koos_classification_by_random_forest.ipynb](random_forest_classification/koos_classification_by_random_forest.ipynb)

The notebook relies on the ground truth Koos grades provided in the CSV file [Koos_grades_groundtruth.csv](Koos_grades_groundtruth.csv) for training and evaluation of predictions. Furthermore, it relies on [TumorLR.csv](TumorLR.csv) which contains information on whether the tumour is on the left or right. This information can easily be extracted from the parcellations for example as described in the above paper. This information is used to convert left/right labels into ipsilateral/contralateral (with respect to the tumour) labels. 

At the beginning of the notebook, the path to the training set (features extracted from GIF output) and the test sets (features extracted from the three nnU-Net predictions) are defined. The test sets are define by which CSV files are available in the defined test set path. Note that cases that are available in both the training set path and the test set path will be removed from the training set.

A new random forest is trained and inference run for each of the test sets. Moreover, for each test set a confusion matrix and classification report will be printed to the console.

## References
1. Cardoso MJ, Modat M, Wolz R, Melbourne A, Cash D, Rueckert D, Ourselin S. Geodesic Information Flows: Spatially-Variant Graphs and Their Application to Segmentation and Fusion. IEEE Trans Med Imaging. 2015 Sep;34(9):1976-88. doi: 10.1109/TMI.2015.2418298. Epub 2015 Apr 14. PMID: 25879909.
2. Dorent, R., Kujawa, A., Ivory, M., Bakas, S., Rieke, N., Joutard, S., Glocker, B., Cardoso, J., Modat, M., Batmanghelich, K. and Belkov, A., 2022. CrossMoDA 2021 challenge: Benchmark of Cross-Modality Domain Adaptation techniques for Vestibular Schwnannoma and Cochlea Segmentation. arXiv preprint arXiv:2201.02831.
3. Shapey, J., Kujawa, A., Dorent, R., Wang, G., Dimitriadis, A., Grishchuk, D., Paddick, I., Kitchen, N., Bradford, R., Saeed, S. R., Bisdas, S., Ourselin, S., & Vercauteren, T. (2021). Segmentation of vestibular schwannoma from MRI, an open annotated dataset and baseline algorithm. In Scientific Data (Vol. 8, Issue 1). Springer Science and Business Media LLC. https://doi.org/10.1038/s41597-021-01064-w
4. Isensee, F., Jaeger, P.F., Kohl, S.A.A. et al. nnU-Net: a self-configuring method for deep learning-based biomedical image segmentation. Nat Methods 18, 203–211 (2021). https://doi.org/10.1038/s41592-020-01008-z