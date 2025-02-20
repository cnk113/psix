# Psix

Psix is a computational tool for identifying cell-state associated alternative splicing events in single cell RNA-seq (scRNA-seq) data.

Inspired by autocorrelation approaches, Psix will tell you if an exon's splicing is significantly associated with a cell metric that shows the relationships between single cells. In practice, this map could be a low-dimensional representation of the gene expression of a single cell population. Psix also identified modules of potentially co-regulated exons.

[Coverage dependent biases](https://elifesciences.org/articles/54603) add unwanted technical variation to splicing observations in single cells. Psix uses a probabilistic approach to fit two models for each exon: 

* Model 1: exon usage is cell-state associated. Under this model, each cell's <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> is more likely to be similar to the average of its neighbors, than to the global average.
* Model 2: exon usage is independent of cell state. Under this model, each cell's <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> is equally likely to be similar to the average of its neighbors, than to the global average.

By comparing the probability of the observations given each model, Psix estimates a score. The higher the score of an exon, the more confident we are that the exon is cell-state associated.

#### A few examples of running Psix on smart-seq2 data:

* [Mouse midbrain development](https://github.com/lareaulab/analysis_psix/blob/main/midbrain_development/midbrain_development.ipynb)
* [Mouse midbrain development (STARsolo)](https://github.com/lareaulab/analysis_psix/blob/main/midbrain_development/midbrain_development_STARsolo.ipynb)
* [iPS cells neurogenesis](https://github.com/lareaulab/analysis_psix/blob/main/ipsc_neurogenesis/ipsc_neurogenesis.ipynb)
* [mES cells neurogenesis](https://github.com/lareaulab/analysis_psix/blob/main/mesc_neurogenesis/mesc_neurogenesis.ipynb)
* [Tabula Muris, Brain (in progress...)](https://github.com/lareaulab/analysis_psix/blob/main/tabula_muris_brain/tabula_muris_brain.ipynb)

## Installation

Psix is a Python module and it requires Python version 3.6 or higher. Psix is installed directly from github using the following command:

```
pip install git+https://github.com/lareaulab/psix.git
```

Missing package dependencies will be automatically installed.

## Psix input

##### 1. SJ.out.tab files OR STARsolo SJ features

For calculating the matrix of exon <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}">. We recommend mapping raw scRNA-seq reads using STAR version ≥ 2.5.3a. Psix uses the ```SJ.out.tab``` files from the STAR aligner. Individual files from each single cell should be stored in the same directory with the following naming format: ```cellID.SJ.out.tab```. The files can be gzipped (ending with ```.gz```), or uncompressed. Additionally, Psix can also work with the output of STARsolo when passing the argument ```--soloFeatures SJ```.

##### 2. Cassette exon annotation

This consists of a table specifying the location (chromosome, start and end) of splice junctions. Splice junctions are annotated as supporting the inclusion of a cassette exon (\_I1 and \_I2), supporting its exclusion (\_SE), or constitutive (\_CI). You can download ready-to-use mouse (mm10) and human (hg38) annotations [here](https://github.com/lareaulab/psix/tree/master/annotation). For creating your own cassette exon annotation from GTF files, see [GTF2psi](https://github.com/cfbuenabadn/sc_splicing_tools).

##### 3. TPM matrix (for smart-seq2 only)

For estimating the number of mRNA molecules captured per observation, using the normalization described in [Qiu et al., 2017](https://www.nature.com/articles/nmeth.4150) and in [Buen Abad Najar et al., 2020](https://elifesciences.org/articles/54603). The TPM matrix of **gene** expression can be obtained running different methods. We use RSEM version ≥ 1.2.31 because it can be run using STAR as the aligner. Other methods such as Kallisto can also be used to generate this matrix. This is only required for smart-seq2 data. Go to **Running Psix in UMI data** to see the details about running Psix on UMI data.

##### 4. Low-dimensional cell space

To obtain a metric of phenotypic similarity between single cells. A low-dimensional cell space is provided by the user, since Psix does not perform dimensionality reduction. In principle Psix can run with any metric. However, we recommend using interpretable dimensionality reduction methods such as PCA over the normalized gene expression, while avoiding non-interpretable methods such as tSNE except for visualization. 

For small smart-seq2 datasets (fewer than 5000 cells), we recommend using SCONE to select the best normalization method before applying a linear dimensionality reduction such as PCA. Alternatively, other methods such as ZINB-Wave can be used on this data. For larger datasets, we recommend using the latent space of scVI as the low-dimensional cell space.

## Getting started

This section was written with smart-seq2 data in mind, but running Psix on UMI data is not much different. For the specifics on running Psix on UMI-based scRNA-seq data, go to **Running Psix in UMI data**.

Psix requires four inputs from the user:
* A directory containing SJ.out.tab files from STAR. This is used to calculate the exon's observed <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}">.
* A matrix of gene expression in transcripts per million (TPM; for smart-seq2 data only). This is used to estimate the captured mRNA molecules per observation.
* A low-dimensional cell space; e.g., a PCA projection of normalized gene expression. This is used to define cell neighborhoods.
* A cassette exon annotation of splice junctions. Ready-to-use mouse (mm10) and human (hg38) annotations are provided [here](http://github.com/laeraulab/psix/annotations/).

### Creating a Psix object with smart-seq2 data

You can import Psix and create a Psix object by running:

```python
import psix
psix_object = psix.Psix()
```

This will create an empty Psix object. To run Psix, we have to first calculate the observed <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> and estimate the number of mRNA molecules captured per observation.

#### From cellID.SJ.out.tab files

If we have a directory with individual SJ.out.tab files for each cell (each the result of running STAR individually on each cell), we can simply run:


```python
psix_object.junctions2psi(
        sj_dir='/path/to/SJ_files/directory/',
        intron_file='/path/to/cassette_exon_annotation.tab',
        tpm_file='/path/to/gene_expression/tpm_file.tab',
        save_files_in='psix_output/'
    )
```

The optional argument ```save_files_in``` will create a directory where Psix will store the <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> and mRNA counts matrices for future use. 

#### From STARsolo features

If you are aligning the raw reads using [STARsolo](https://github.com/alexdobin/STAR/blob/master/docs/STARsolo.md), make sure to pass the argument ```--soloFeatures SJ```. You can create a Psix object directly from the raw output of STARsolo by passing the argument ```solo=True``` when running ```junctions2psi```: 

```python
psix_object.junctions2psi(
        sj_dir='/path/to/STARsolo_output/Solo.out/SJ/raw/',
        intron_file='/path/to/cassette_exon_annotation.tab',
        tpm_file='/path/to/gene_expression/tpm_file.tab',
        save_files_in='psix_output/',
        solo = True
    )
```

```/path/to/STARsolo_output/Solo.out/SJ/raw/``` should contain the output of runnign STARsolo with the argument ```--soloFeatures SJ```. This will contain three files generated by STARsolo: ```features.tsv```, ```barcodes.tsv``` and ```matrix.mtx```.

#### From PSI and mRNA per exon tables

Saving the <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> and mRNA counts matrices with the optional argument ```save_files_in``` allows us to skip this step after running ```junctions2psi``` for the first time. This is done by specifying the location of the <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> and mRNA counts matrices when creating a Psix object:

```python
from psix import Psix
psix_object = Psix(psi_table = 'psix_output/psi.tab.gz',
                   mrna_table = 'psix_output/mrna.tab.gz')
```

### Getting cell-state associated exons

After creating a Psix object, we can obtain the Psix scores of each exon by running:

```python
psix_object.run_psix(latent='/path/to/low_dimensional_space.tab', 
                     n_random_exons=2000, 
                     n_neighbors=100
                     )
```


By default, Psix divides the exons into 25 sets according to their variance (five bins) and averge (five bins) <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> to calculate the empirical p-values. Estimating the empirical p-values of exons is the most time consuming step of Psix, specially in large datasets. To speed things up, you can run Psix in parallel simply by specifying the number of threads _t_ by passing the argument ```n_jobs=t```.

The results of Psix can be found at ```psix_object.psix_results``` in the form of a dataframe with the following information:


Exon | psix_score | pvals | qvals
---- | ---- | ---- | ---- 
Mapt_1 | 2.709381 | 0.0005 | 0.001879
Ndrg4_1 | 2.359093 | 0.0005 | 0.001879
Dbn1_1 | 2.302729 | 0.0005 | 0.001879
Mapt_3 | 1.964202 | 0.0005 | 0.001879
Gabrg2_1 | 1.896363 | 0.0005 | 0.001879
... | ... | ... | ...


Notice that the empirical p-values are estimated with exon permutations. For this reason, the Psix score is a better value for ranking exons with very low p-values, than the p-values themselves.

### Modules of correlated exons

Psix can find modules of correlated exons by using the neighbor average <img src="https://render.githubusercontent.com/render/math?math=\bar{\Psi}"> previously used for fitting the model in which an exon is cell-state associated. Using the neighbor average greatly reduces unwanted noise from the raw data. We can obtain these modules as follows:

```python
psix_object.compute_modules(plot = True)
```

<img src="docs/_images/midbrain_modules.png" width="500"/>

The modules for each cell-state associated exon can be found at ```psix_object.modules```. This is a pandas Series that maps each exon to the number of the module to which they belong. Each module is assigned an integer. Exons that are not assigned to any module are labeled with -1.

We can also visualize the average normalized splicing of the labeled modules in the single cell population as follows:

```python
psix_object.plot_modules(save_plots='plots/')
```

Here we show a few examples:

<img src="docs/_images/module_1.png" width="200"/>
<img src="docs/_images/module_2.png" width="200"/>
<img src="docs/_images/module_8.png" width="200"/>
<img src="docs/_images/module_9.png" width="200"/>

The argument ```save_plots``` indicates a directory where the plots are saved in PNG format. Leaving the argument empty (default) will only show the plots, without saving them.

### Saving and loading a Psix object

We can save out Psix object for future uses. This will allow us to skip all the preprocessing, scoring and clustering steps. We can save out object by running the following command:
    

```python
psix_object.save_psix_object(psix_dir = 'psix_output')
```

This will create a directory named ```psix_output``` (if it doesn't exist already), where it will store the Psix object. By default, Psix will not overwrite an existing Psix oject with the same name. If you want to overwrite an existing Psix object, you should use the ```overwrite=True``` argument when running ```save_psix_object```.

Next time we need to run Psix, we can load the results of the previous run (including the <img src="https://render.githubusercontent.com/render/math?math=\hat{\Psi}"> and mRNA tables, the Psix scores, and modules) by creating a new Psix object as follows:

```python
psix_object = psix.Psix(psix_object = 'psix_output')
```

### Creating a Psix object from UMI data (Beta)

Unique molecular identifier (UMI) based sequencing methods, such as 10X, provide an accurate estimate of how many mRNA molecules are captured per gene. For this reason, when running Psix on such data, we do not need to approximate the number of mRNA molecules by normalizing TPM counts. We specify to Psix that we are working with UMI data on the ```junctions2psi``` function:

```python
import psix

psix_object = psix.Psix()
psix_object.junctions2psi(
        sj_dir='/path/to/SJ_files/directory/',
        intron_file='/path/to/cassette_exon_annotation.tab',
        save_files_in='psix_output/',
        tenX = True
    )
```

The ```TenX = True``` argument will let Psix know that it is working with UMI data. Notice that the ```tpm_file``` argument is not included; passing anything when ```tenX=True``` will be ignored. Not specifying a ```tpm_file``` when ```tenX=False``` (by default) will result in an error.

After running ```junctions2psi```, the rest of the analysis is the same as when working with smart-seq2 data, including saving the processed tables, and the Psix object.


## Psix turbo

The most time consuming step of Psix is calculating the Psix score of each cell. When estimating the score of large datasets, specially when estimating the p-values, this can make Psix take a few hours to run. To work around these time limitations, we implemented a turbo functionality for Psix. Psix Turbo uses predefined look-up tables to efficiently assign a Psix score to each cell at a minor cost in accuracy.

To use Psix turbo, we need to pre-compute look-up tables first as follows:

```python
import psix
psix.make_turbo(out_dir = 'psix_turbo/', 
                granularity = 0.01, 
                max_mrna = 20, 
                capture_efficiency = 0.1, 
                min_probability = 0.01
               )
```

This will create a directory with the look-up tables at ```psix_turbo/```. You can download a pre-computed directory of look-up tables [here](https://github.com/lareaulab/psix/tree/master/psix/psix_turbo). 

Once we have out look-up table index, we can tun Psix turbo simply by specifying the location of this directory when running ```run_psix```:

```python
psix_object.run_psix(atent='/path/to/low_dimensional_space.tab', 
                     n_random_exons=2000, 
                     n_neighbors=100, 
                     turbo='psix_turbo/')
```

This will cut the runtime several orders of magnitude shorter at a small cost in accuracy.
