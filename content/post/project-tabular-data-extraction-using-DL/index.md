---
title: "Extracting tabular data from Photovoltaic module datasheets using Table-Transformer"
date: 2024-01-17T15:03:24+01:00
author: "Swathi Thiruvengadam"
authorAvatar: "img/ada.jpg"
tags: [Table detection, Table structure recognition, Tabular data extraction]
categories: ["project"]
image: "img/project-tabular-data-extraction-using-DL/solar-panel.jpg"
draft: false
---



This project aims to extract crucial tabular data from Photovoltaic Module Datasheets using Microsoft's Table-Transformer.

<!--more-->

## Content

+ [Motivation](#motivation)
+ [Prior work](#prior-work)
    - [Introduction to Object Detectors](#introduction-to-object-detectors)
    - [Two - Stage Object Detectors](#two---stage-object-detectors)
    - [Single - Stage Object Detectors](#single---stage-object-detectors)
    - [Introduction to Existing Pipeline : Lightning-Table](#introduction-to-existing-pipeline--lightning-table)
    - [Drawbacks of existing pipeline](#drawbacks-of-existing-pipeline)
+ [Table-Transformer](#table-transformer)
    - [Introduction](#introduction)
    - [Model](#model)
    - [Canonicalization](#canonicalization)
    - [Limitations](#limitations)
    - [Training the Model](#training-the-model)
    - [Evaluation metrics ](#evaluation-metrics)
+ [Inference pipeline](#inference-pipeline)
    - [Working with PDF](#working-with-pdf)
    - [Data Extraction](#data-extraction)
    - [Table Detection](#table-detection)
    - [Table Structure Recognition](#table-structure-recognition)
+ [Results](#results)
    - [Extraction Results](#extraction-results)
    - [Table detection performance comparison](#table-detection-performance-comparison)
    - [Table structure recognition model comparison](#table-structure-recognition-models-comparison)
    - [Table extraction performance](#table-extraction-performance)
+ [Advantages of Table Transformer](#advantage-of-using-table-transformer-over-lightning-table)
+ [Drawbacks of Table Transformer](#drawbacks-of-table-transformer)
+ [Conclusion and future goals](#conclusion-and-future-goals)
+ [References](#references)

## Motivation

Solar power generation is one of the most commonly used clean and renewable sources of energy. The photovoltaic modules, commonly known as solar panels, capture solar rays and transform it into sustainable energy. The various organizations that produce these modules capture their technical specifications such as its electrical, mechanical or temperature characteristics, as well as the physical properties of these modules in a tabular form and this information is made available in a PDF format. 

The solar module datasheets contain critical information about the modules performance and they can be leveraged to advance PV module research, gain useful insights about the module performance and ultimately help produce better PV modules.  But there is no standardization in the design of these data sheets across the markets and as a result, computers cannot readily process and extract information from them. Manual data extraction from these PDF datasheets can be tedious, time consuming and prone to human errors. 

I wish to extend on some prior work in this field by creating an end-to-end pipeline that uses Deep Learning (DL) for table detection and table structure recognition before extracting the data from solar modules datasheets. 


## Prior work

### Introduction to Object Detectors

Object detection is a field of computer vision that deals with teaching the computers to detect objects of interest in image-based data. The object detectors can also be used to detect tables from images which are rendered from PV module datasheets or PDF documents.  These object detectors can be broadly classified into two types: Two-Stage object detector and Single-Stage object detectors.

#### Two - Stage Object Detectors

Two-Stage object detectors consist of a region proposal step followed by a classification step. During the region proposal step, the network proposes a set of regions in the image that might contain the object of interest. In the second stage the proposed regions are classified into one of some predefined categories.

FasterRCNN by _Ren et al, 2015 [6]_  is one of the most commonly used two-stage detectors in DL applications. Earlier networks such as RCNN by _Girshick et al [8]_  and FastRCNN by _Girshi [9]_ used a selective search methodology for region proposal which would propose around 2000 regions per image. But this proved to be a major bottleneck for speed of the network. To overcome this bottleneck, _Ren et al. [6]_ presented FasterRCNN where the region proposal task by using selective search was replaced by a convolutional network called region proposal network. This network removed the bottleneck for speed and resulted in significant performance improvement. A convolutional neural network (CNN) is then applied to the image only once to extract the features. The regions proposed by the region proposal network are then projected onto these features.

#### Single - Stage Object Detectors

In Single-Stage object detectors (SSD), both the bounding boxes for the objects as well as its class are predicted in a single step. This makes SSDs faster than the two-stage detectors but they are often less accurate in comparison.

RetinaNet is one of the best single-stage object detectors as it works well on dense and small scale objects. As SSDs do not have a separate region proposal network, the network has to evaluate around 100 regions per image resulting in a large number of negatives. This causes a class imbalance problem as the training becomes inefficient since the easy negatives do not contribute to useful learning and the easy negatives can also overwhelm the learning process, resulting in degenerate models. Hence _Lin et al. [7]_ presented a novel loss function called focal loss to narrow down the regions of interest. Easy to classify regions are weighed less to let the network focus more on the regions that are hard to learn.

### Introduction to Existing Pipeline : Lightning-Table

The existing pipeline _[2]_ aims not only to extract the tabular data present in the datasheets but also processes and reformat the extracted data according to _Figure 1_ that can be used in PV module research to gain useful insights. This pipeline can be divided into three major problem areas.  Firstly, a deep learning based object detector was used to detect tables in the datasheets. In the second stage, the raw values present in these tables were extracted using off-the-shelf table extraction packages such as Tabula _[10]_ and Camelot _[11]_. Finally, the extracted raw values are structured and reformatted using rule-based methods to get the desired output. This reformatted data is then used to visualize the latest production trends and gain insights about the module performance to help produce better PV modules.

![Figure 1 : Data extraction pipeline of Lightning Table](/img/project-tabular-data-extraction-using-DL/Picture1_extraction_pipeline.png)<p style="text-align: center;">Figure 1 : Data extraction pipeline of Lightning Table</p>
  
![Figure 2 : PDF documents are converted into images](/img/project-tabular-data-extraction-using-DL/Picture2_pdf2image.png)<p style="text-align: center;">Figure 2 : PDF documents are converted into images</p>

![Figure 3 : Table Detection : Locating all tables in the image](/img/project-tabular-data-extraction-using-DL/Picture3_table_detection.png)<p style="text-align: center;">Figure 3 : Table Detection : Locating all tables in the image</p> 

![Figure 4 : Table Recognition : Identifying rows and columns in the tables and extracting raw data](/img/project-tabular-data-extraction-using-DL/Picture4_table_recognition.png)<p style="text-align: center;">Figure 4 : Table Recognition : Identifying rows and columns in the tables and extracting raw data</p>  
         
![Figure 5 : Locating, extracting and reformatting the required information from extracted tables](/img/project-tabular-data-extraction-using-DL/Picture5_reformatting.png)<p style="text-align: center;">Figure 5 : Locating, extracting and reformatting the required information from extracted tables</p>

![Figure 6 : Gaining useful insights from extracted table](/img/project-tabular-data-extraction-using-DL/Picture6_insights.png)<p style="text-align: center;">Figure 6 : Gaining useful insights from extracted table</p>

This pipeline _[2]_ achieved 98.9% precision and 94.0% recall performance at extracting electrical and thermal characteristics for simple tables from solar cell datasheets. 

![Figure 7 : The evaluation results of Lightning table on solar cell datasheets](/img/project-tabular-data-extraction-using-DL/Picture7_LT_cells_results.jpg)<p style="text-align: center;">Figure 7 : The evaluation results of Lightning table on solar cell datasheets</p>


### Drawbacks of existing pipeline

The existing pipeline _[2]_ made use of off-the-shelf python libraries such as Camelot and Tabula which are rule-based utilities along with a baseline method to extract data from PDF files. These evaluation results were run on the electrical and thermal characteristics tables of solar cell datasheets which have rather simple table structures. However, this pipeline could not be used for extracting data from solar module datasheets which contain more complex table structure.

The following drawbacks can be observed when working on solar module datasheets:

* It failed in understanding complex table structures like merged rows and columns, canonical cells and at capturing the cell data accurately. 
* It also suffered greatly when the cell data occupied multiple lines. The corresponding extracted data was treated as different entries.
* The difference in text alignments and line spacings also hindered model performance.
* In general, it lacked context and failed to understand the relationship between data.
* Also, Camelot and Tabula could only extract data from text-based PDFs. 

As a result, some deficiencies were observed in the form of missing values in the extracted tables, incorrect merging of data and improper table structure. _Figure 8_ is the image of a table from a solar module datasheet and _Figure 9_ contains an image of the data extracted from the table in _Figure 8_ using Camelot. In this example, it can be observed from _Figure 9(a)_ that the data from the last 2 columns are merged together. Missing and misaligned values can also be seen in _Figure 9(b)_ which is due to varying line spacing. Finally, the inability of Camelot to represent merged cells can be observed in _Figure 9(c)_.


![Figure 8 : A simple table capturing the electrical specifications of a solar module](/img/project-tabular-data-extraction-using-DL/Picture8_simple_table.png)<p style="text-align: center;">Figure 8 : A simple table capturing the electrical specifications of a solar module</p>

![Figure 9 : Data extracted from Figure 8 by Lightning Table using Camelot](/img/project-tabular-data-extraction-using-DL/Picture9_camelot.png)<p style="text-align: center;">Figure 9 : Data extracted from Figure 8 by Lightning Table using Camelot</p>

Consequently, the rule-based method of data extraction and reformatting of data in the final step of this pipeline will fail as a result. Figure 10 depicts the result of the final data extraction and reformatting step. We can observe that not all data was captured and there are a lot of missing values. Improving the tabular data extraction in the previous step should improve the results significantly.  

![Figure 10 : Output of the final extraction step which should capture all specifications together (electrical, mechanical and technical)](/img/project-tabular-data-extraction-using-DL/Picture10_final_step.png)<p style="text-align: center;">Figure 10 : Output of the final extraction step which should capture all specifications together (electrical, mechanical and technical)</p> 


## Table-Transformer
                                        
### Introduction

Table Transformer(TATR) introduced by _Smock et al[1]_, is a Deep Learning based object detection model for extracting tables from unstructured documents and images. 

Unlike standard object detection models like Faster R-CNN and Mask R-CNN that rely on region proposals, non-maximum suppression procedure and anchor generation, Detection Transformer (DETR)  is a single-stage object detector that can be trained end-to-end due to the use of a combination of bipartite matching loss and prediction loss. The prediction/Hungarian loss used in DETR eliminates the need to sort ground truth data and prediction by comparing loss between two unordered sets. Also, the bipartite matching forces a 1-to-1 matching of values. This makes DETR faster than standard two-stage detectors while also ensuring accuracy.

In TATR, a table’s hierarchical structure is modeled by using six object classes: 1) table, 2) table column, 3) table row, 4) table column header, 5) table projected row header, and 6) table spanning cell as defined in _Figure 11_. The intersection of these pairs of table column and table row objects form an additional class called table grid cell. 

![Figure 11 :  Bounding box annotations of the various object classes in TATR[1]](/img/project-tabular-data-extraction-using-DL/Picture11_bb_annotations.png)<p style="text-align: center;">Figure 11 :  Bounding box annotations of the various object classes in TATR[1]</p> 

### Model

Table Transformer architecture consists of 2 Detection Transformer (DETR): one to perform the task of table detection and the other one for performing the table structure recognition. All models use a ResNet-18 backbone pretrained on ImageNet dataset with the first few layers frozen with mostly default settings to allow the data to drive the result.

### Canonicalization

In TATR, to ensure that the annotations are free of noise, the cells in the headers were canonicalized to correct over-segmentation in a table’s structure annotations. In addition, multiple quality control steps were also implemented. 

_Figure 12_ depicts the canonicalization algorithm used and it amounts to merging adjacent cells under certain conditions. The merging conditions differ for both the table header cells and cells present in the table body.

![Figure 12 : Canonicalization algorithm used in TATR[1]](/img/project-tabular-data-extraction-using-DL/Picture12_canonicalization.png)<p style="text-align: center;">Figure 12 : Canonicalization algorithm used in TATR[1]</p>

### Limitations

The algorithm described in _Figure 12_ was designed to perform canonicalization for annotations in the PMCOA dataset which predominantly consists of vertical tables. Hence, it can only observe the merging of cells in the table column header and the table projected row header. Canonicalizing tables from other datasets with horizontal tables will require additional assumptions and was not covered in the scope of TATR. 

### Training the Model

The TATR model was trained on the PubTables-1M dataset which is a large, detailed and high quality dataset for training and evaluating models on the tasks of table detection, table structure recognition, and functional analysis. The model was trained on PNG images and the corresponding annotations were in the form of a JSON file. 

Compared to prior datasets, PubTables-1M dataset contains richer annotation information, including annotations for projected row headers and bounding boxes for all rows, columns, and cells, including blank cells. It also includes annotations on their original source documents, which supports multiple input modalities and enables a wide range of potential model architectures, thereby proving that improvements to the ground truth alone can have a positive impact on the model performance as successfully demonstrated by _Smock et al [1]_.

### Evaluation metrics

Standard object detection metrics like AP, AP50, AR and AP75 are used to evaluate table-transformer for the task of table detection. Average Precision (AP) measures the area under the precision-recall curve which is obtained by varying the confidence threshold for predicted bounding boxes. AP50 and AP75 are variants of AP where precision is computed when the IoU (Intersection over Union) between predicted and ground truth bounding boxes are 50% and 75% respectively. Average Recall is computed by averaging the recall values at different IoU thresholds. It measures the ability of the detector to recall objects across a range of IoU thresholds.

In addition to this, _Smock et al [1]_ also computed grid table similarity (GriTS) for the evaluation of table structure recognition. It is a measure of table cell correctness and is defined as the average correctness of each cell averaged over all tables. 

_Figure 13_ and _Figure 14_ depict the evaluation results of Table-Transformer on the PubTables-1M dataset. Using an 80/10/10 split, PubTables-1M dataset was randomly split into train, validation, and test sets. As a result, 57,115 samples were used in the evaluation of TATR for the task of table detection and 93,834 samples for the task of table structure recognition. These samples include a mix of simple and complex tables.

![Figure 13: Evaluation metrics for table detection on the PubTables-1M dataset](/img/project-tabular-data-extraction-using-DL/Picture13_eval_TD.png)<p style="text-align: center;">Figure 13: Evaluation metrics for table detection on the PubTables-1M dataset</p>

![Figure 14: Evaluation metrics for table structure recognition on the PubTables-1M dataset](/img/project-tabular-data-extraction-using-DL/Picture14_eval_TSR.png)<p style="text-align: center;">Figure 14: Evaluation metrics for table structure recognition on the PubTables-1M dataset</p>

Leveraging this model instead of Lightning-table _[2]_ would preserve the table structure. Furthermore, this model should improve the extraction of tabular data from tables with complex structure. 

## Inference pipeline

### Working with PDF

The solar model datasheets are typically available as PDF files. They can be either in text-based or image-based formats. The PDF files need to be converted into images as a first step of the inference pipeline. 

To achieve this, the python library _pdf2image [12]_ is used which can convert a PDF document into an PIL (Python Imaging Library) image object as can be seen in _Figure 15_. Given a multi-page PDF file, this library outputs multiple images corresponding to the individual PDF pages by default.

![Figure 15 : Table extraction pipeline using Table Transformer](/img/project-tabular-data-extraction-using-DL/Picture15_TATR_pipeline.png)<p style="text-align: center;">Figure 15 : Table extraction pipeline using Table Transformer</p>

### Data Extraction

TATR is trained to recognize tables from image input as well as identify its structure. However, to capture/include text in the resulting HTML or CSV output file, we need to provide an additional optical character recognition (OCR) input along with the input images files. OCR is used to transform an image of text, either machine printed or handwritten text from its image representation into machine-readable text.

In this application, Pytesserct  _[13]_, a python wrapper for Google’s Tesseract-OCR Engine was used for character recognition since it is open-source, versatile and can read all image types supported by Pillow,  a Python Imaging Library (PIL) fork that adds image processing capabilities to the Python interpreter. This initially produced poor OCR results when input image quality was poor or when input images contain complex backgrounds/watermarks. The results significantly improved by removing noise and improving DPI (dots per inch) of the input image, that is images with better resolution. 

It is important to mention that when the tables lacked a clear structure (missing vertical lines),  text alignment and spacing is used to distinguish between cells, rows, and columns. The text position and content data provided by Pytesseract will help the TATR model to infer the table structure. But faulty OCR data generated by Pytesseract can caues the recognition model to fail resulting in merged text across columns. Furthermore, the vertical lines used to distinguish columns were often captured by Pytesseract as the ‘|’ character.

### Table Detection

Table detection is the process of identifying the tables present in the images and cropping them for further processing. While cropping the detected tables from the PDF image, maintaining the resolution, especially if the cropped region is small is a challenging yet crucial step for the table structure recognition and data extraction step. 

Using a good interpolation method like bicubic interpolation instead of simpler methods like bilinear interpolation for resizing and cropping along with an appropriate amount of padding can lead to higher quality cropped images. But if the region to be cropped is small, then too much padding can also cause loss of image quality.  Additionally, sharpening the image after cropping can enhance the details and help with data extraction in the next stage.


### Table Structure Recognition

Table structure recognition is the process of identifying the rows and columns present in the previously detected tables. The intersection of these rows and columns form the grid cells. In addition to this, TATR can also identify spanning cells that extend across multiple columns in the table. Understanding and correctly handling spanning cells is crucial as it contributes to accurately representing the layout and organization of tabular data, especially when dealing with complex tables that include merged cells for headers, subheadings, or other structural elements.

## Results

### Extraction Results

Unlike table detection packages like Tabula and Camelot which were rule-based, Table Transformer not only extracts the tabular data, but it also captures the context/relationship between data. This feature greatly improves the ability of the network to efficiently extract data from tables with complex structures.  

The key features of table transformer are:
* For a vertical table, if the column is characterized by multiple headings (that is, a column heading along with some subheadings), then the corresponding heading data is combined together to maintain the standard table structure. 
* Data captured in merged cells of the table body are extrapolated across all the table headers within the scope of the merged cell.

_Figure 16_ depicts a table structure recognition example using TATR where the model not only identified the merged cells in the header, but also this data was extrapolated across both its sub-headers during the table extraction process as can be seen in _Figure 17_. On the other hand, _Figure 18_ shows the data extracted using Lightning table for the same example.

![Figure 16 : Table structure recognition using Table Transformer](/img/project-tabular-data-extraction-using-DL/Picture16_TSR_output.png)<p style="text-align: center;">Figure 16 : Table structure recognition using Table Transformer</p>

![Figure 17 : Data extracted from Figure 16 by using Table Transformer](/img/project-tabular-data-extraction-using-DL/Picture17_dataExtracted.png)<p style="text-align: center;">Figure 17 : Data extracted from Figure 16 by using Table Transformer</p>

![Figure 18 : Data extracted from Figure 16 by using Lightning Table](/img/project-tabular-data-extraction-using-DL/Picture18_dataExtracted_LT.png)<p style="text-align: center;">Figure 18 : Data extracted from Figure 16 by using Lightning Table</p>

### Table detection performance comparison

A solar cell dataset and a solar modules dataset is created for evaluating TART and LT models on the task of the table detection. Both datasets consist of 10 PDF files each. These PDF files were then converted into images using the open-source software pdf2image [12] in Python.  

For generating the ground truth, LabelImg a free and open-source program was used for labeling the images. The bounding boxes were drawn manually around the table regions in the images. As a result, slight variations were observed while indicating the boundaries of the tables. Additionally, due to varying data sheet designs, the title of the table sometimes appeared separately from the table. A judgement call was made on a case-by-case basis whether the title of the table should be included in the boundary of the table or not. The title was omitted from the table region if it appeared far away from the table.

Lightning table used the FasterRCNN v2 architecture for table detection since it outperformed single-stage detectors like RetinaNet as established by Malik in [2] and the various model architectures performance results can be seen in _Figure 19_. To allow the network to have a good starting point and increase the training efficiency, weights trained on the Microsoft COCO dataset were used initially. It was then fine-tuned on a custom solar cell dataset consisting of 5896 tables. The results of this model on the task of detecting tables in solar cell dataset can be seen in _Figure 20_. 

![Figure 19 : Detection model architecture comparison in Lightning Table](/img/project-tabular-data-extraction-using-DL/Picture19_detectionmodel_LT.PNG)<p style="text-align: center;">Figure 19 : Detection model architecture comparison in Lightning Table</p>

![Figure 20 : Table detection evaluation results on solar cell dataset using Lightning Table](/img/project-tabular-data-extraction-using-DL/Picture20_LT_detection_cells.JPG)<p style="text-align: center;">Figure 20 : Table detection evaluation results on solar cell dataset using Lightning Table</p>

The detection transformer used for detecting tables in TATR was trained on the PubTables-1M dataset and the corresponding results of this model for the task of table detection on the above mentioned solar cell and solar module datasets can be seen in _Figure 21(a)_ and _Figure 21(b)_ respectively. This model has an average precision of 80.5% and 74.1% at 50% IoU on the solar cell and solar module datasets respectively. _Figure 22_ compares the precision and recall results of Table-transformer with Lightning-Table on the task of detecting table in solar cell and solar module datasheets. This detection accuracy is significantly less in comparison to the evaluations performed on the Pubtables-1M dataset which yielded an average precision of 99.5%. This difference is mainly attributed to the omission of the table headers since they typically have different colour, font, size etc as compared to the table body in solar module datasheets, and the inability of the model to distinguish various tables that are in close proximity like in _Figure 23_.

![Figure 21(a) : Table detection evaluation results on solar cell dataset using Table Transformer](/img/project-tabular-data-extraction-using-DL/Picture21(a)_TATR_detection_cells.PNG)<p style="text-align: center;">Figure 21(a) : Table detection evaluation results on solar cell dataset using Table Transformer</p>

![Figure 21(b) : Table detection evaluation results on solar module dataset using Table Transformer](/img/project-tabular-data-extraction-using-DL/Picture21(b)_TATR_detection_modules.PNG)<p style="text-align: center;">Figure 21(b) : Table detection evaluation results on solar module dataset using Table Transformer</p>

![Figure 22 : Detection performance comparision of Table transformer with Lightning Table](/img/project-tabular-data-extraction-using-DL/Picture22_detection_comparision.PNG)<p style="text-align: center;">Figure 22 : Detection performance comparision of Table transformer with Lightning Table</p>

![Figure 23 : Inability of TATR to distinguish between tables in close proximity](/img/project-tabular-data-extraction-using-DL/Picture23_TATR_detection_drawback.PNG)<p style="text-align: center;">Figure 23 : Inability of TATR to distinguish between tables in close proximity</p>

### Table structure recognition models comparison

The TATR-v1.0 model is a DETR model with an ResNet-18 backbone and it was trained on the PubTables-1M dataset.  On the other hand, the TATR-v1.1 model was trained with additional image cropping and on more epochs.  It works best on tightly cropped table images. This model was further trained on 3 datasets giving rise to the following models :  TATR-v1.1-Pub (trained on PubTables-1M dataset), TATR-v1.1-Fin (trained on FinTabNet.c dataset) and TATR-v1.1-All (trained on both PubTables-1M and FinTabNet.c datasets). Figure 24 depicts the precision scores of these models on the task of Table structure recognition. 

![Figure 24 : Performance comparison of TSR models](/img/project-tabular-data-extraction-using-DL/Picture24_TSR_model_Comparision.PNG)<p style="text-align: center;">Figure 24 : Performance comparison of TSR models</p>

### Table extraction performance

For evaluating the tabular data extraction performance of TATR, the previously created solar cell and solar module datasets were used while also ensuring that these documents were diverse and included different layouts/styles and complex table structures. The tabular data was manually extrated from the PDF files to form the ground-truth data. The data present in merged cells in the 'table body' were extrapolated across the merged columns and the data present in merged cells in the 'table header' was concatinated with all its sub-headings as described in _Figure 17_. This data was then saved as a YAML file for easy access and data manipulation where the key corresponds to the table column header (if available) or the first table column and the values corresponds to the content in that row.

An attempt was made to match cells individually, but this is based on the structure provided by the ground truth and the cells position in the table. The primary challenge was matching predicted rows and columns with ground truth keys, especially if the structure is complex. However, a perfect cells content match was not expected while calculating the precision scores. Instead Levenshtein distance metric was used to allow some tolerance since the data extracted from tables using Pytesseract was not accurate and contains misrecognized characters. A string similarity of upto 2 character differences was allowed. 

The detection transformer used to recognize the table structure in TATR was trained on the PubTables-1M dataset. TATR uses the object classes ‘table column header’ and 'table project row header' in its hierarchical structure definition to account for horizontal tables. Consequently, these parameters can be used to handle merged cells  in the table's column header that span across multiple columns using the canonicalization algorithm as seen in _Figure 12_.  No measures were taken to handle vertical tables with Row headings and merged cells that span across multiple rows.  Whereas, the solar module datasets predominantly consists of a large number of vertical tables. As a result the task of Table structure recognition yields only 67.4% and 55.3% Average Precision scores on the solar cell and solar module datasets respectively. The precision and recall scores were computed for every PDf file individually and averaged over the dataset to get these values. This can be observed in _Figure 25_ and _Figure 26_ respectively.  

![Figure 25 : Table extraction from solar cell datasheets](/img/project-tabular-data-extraction-using-DL/Picture25_TE_solarCell_TATR.PNG)<p style="text-align: center;">Figure 25 : Table extraction from solar cell datasheets</p>

![Figure 26 : Table extraction from solar module datasheets](/img/project-tabular-data-extraction-using-DL/Picture26_TE_solarModule_TATR.PNG)<p style="text-align: center;">Figure 26 : Table extraction from solar module datasheets</p>

## Advantage of using Table Transformer over Lightning Table

* Table Transformer can extract data from text-based PDFs as well as image-based PDFs.
* Unlike Lightning table, Table transformer can understand context and relationship between data thereby accurately capturing data from rows with variable heights or cells spanning across multiple rows. _Figure 28_ and _Figure 29_ depict the data extracted from a table described in _Figure 27_ using Lightning Table and Table Transformer respectively. 
![Figure 27 : Table detected from solar module datasheets with variable height rows](/img/project-tabular-data-extraction-using-DL/Picture27_context.PNG)<p style="text-align: center;">Figure 27 : Table detected from solar module datasheets with variable height rows</p>

![Figure 28 : Data extracted from Figure 27 using Lightning Table](/img/project-tabular-data-extraction-using-DL/Picture28_context_LT.PNG)<p style="text-align: center;">Figure 28 : Data extracted from Figure 27 using Lightning Table</p>

![Figure 29 : Data extracted from Figure 27 using Table Transformer](/img/project-tabular-data-extraction-using-DL/Picture29_context_TATR.PNG)<p style="text-align: center;">Figure 29 : Data extracted from Figure 27 using Table Transformer</p>

* Recognize the structure of complex tables and extrapolate the data across merge cells using its canonicalization algorithm. This example can be observed in _Figure 16_ and _Figure 17_.

## Drawbacks of Table Transformer

* When the detected tables did not have clear rows and columns defined by bounding lines, TATR failed to recognize merged cells. In _Figure 30_, it can be observed that instead of a single merged cell, the last four rows in the table were considered to have individual cells following the same structure as the previous rows. As a result, the corresponding values were not extrapolated across the various module types during data extraction.
![Figure 30 : Failure to recognize merged cells](/img/project-tabular-data-extraction-using-DL/Picture30_fail_spanning.PNG)<p style="text-align: center;">Figure 30 : Failure to recognize merged cells</p>

* TATR fails to recognise the table structure and extract data when the detected tables have complex backgrounds or patterns. _Figure 31_ shows how varying background colors cause the recognition model to fail. 
![Figure 31 : Table with varying background colors fail](/img/project-tabular-data-extraction-using-DL/Picture31_fail_background.PNG)<p style="text-align: center;">Figure 31 : Table with varying background colors fail</p>

* Error in recognizing the cells accurately due to difference in alignment, line spacing, size, font etc. of text within the same row or column. In the example shown in Figure 32(a), we can observe that the alignment of data in the first column is different. Also the data is not aligned vertically across the rows in Figure 32(b).  
![Figure 32 : Failure due to varying text alignments](/img/project-tabular-data-extraction-using-DL/Picture32_fail_alignment.PNG)<p style="text-align: center;">Figure 32 : Failure due to varying text alignments</p>

* TART was only trained on vertical tables and the canonicalization algorithm used in this model only accounts for table columns header and table projected row header classes and hence it cannot be used to recognise the structure of horizontal tables. This is depicted in Figure 25 where the first column is the table's header.  
![Figure 33 : Failure to recognize structure of vertical tables](/img/project-tabular-data-extraction-using-DL/Picture33_fail_vertical_tables.PNG)<p style="text-align: center;">Figure 33 : Failure to recognize structure of vertical tables</p>

* TATR fails to localize and accurately detect individual tables that are in close proximity. Often times this leads to merging multiple tables during data extraction. This can be seen in _Figure 23_.

## Conclusion and future goals
As illustrated, the main advantage of using Table transformer over Lightning table for the extraction of tabular data from solar cell and solar module datasheets is that it can understand the structure of complex tables and extrapolate the data across merged cells using the canonicalization algorithm, thereby creating a simple table which could greatly aid in post-processing the extracted data to gain useful insights.

In the process of development and evaluation, some areas were identified that can be explored in the future:
* Improving the table detection model to better segment the tables in close proximity.
* Improving the table structure recognition model by applying transfer learning concepts to fine-tune the model on a solar cell and solar module datasets.
* Extend the canonicalization algorithm to account for vertical tables and merged cells across several rows.

## References

* [1] Smock, B., Pesala, R., Abraham, R.: Pubtables-1m: Towards comprehensive table extraction from unstructured documents. In: Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 4634–4642 (2022)
* [2] https://ad-publications.cs.uni-freiburg.de/theses/Master_Moeez_Malik_2023.pdf
* [3] Smock, B., Pesala, R., Abraham, R.: “Aligning benchmark datasets for table structure recognition”, International Conference on Document Analysis and Recognition- ICDAR 2023
* [4] Smock, B., Pesala, R., Abraham, R.: “GriTS: Grid table similarity metric for table structure recognition”
* [5] https://github.com/microsoft/table-transformer
* [6] S. Ren, K. He, R. Girshick, and J. Sun, “Faster r-cnn: Towards real-time object detection with region proposal networks,” Advances in neural information processing systems, vol. 28, 2015.
* [7] T.Y. Lin, P. Goyal, R. Girshick, K. He, and P. Dollár, “Focal loss for dense object detection,” in Proceedings of the IEEE international conference on computer vision, pp. 2980–2988, 2017.
* [8] R. Girshick, J. Donahue, T. Darrell and J. Malik, "Region-Based Convolutional Networks for Accurate Object Detection and Segmentation," in IEEE Transactions on Pattern Analysis and Machine Intelligence
* [9] R. Girshick; “Fast R-CNN”, Proceedings of the IEEE International Conference on Computer Vision (ICCV), 2015.
* [10] https://tabula-py.readthedocs.io/en/latest/getting_started.html#
* [11] https://github.com/atlanhq/camelot
* [12] https://pypi.org/project/pdf2image/
* [13] https://pypi.org/project/pytesseract/ 
