# IPnoisy: a method to estimate noisy transcription factor binding sites from replicated experiments 
IPnoisy is a method capable to detect noisy DNA binding regions of several transcription factors in a given cell type. It takes in input ChIP-seq peaks and outputs the noisy ones that tend to be not reproducible. It can be useful to use this method in case a wet lab has obtained peaks from ChIP-seq expriments and wants to know their reliability before downstream interpretative analysis and/or experimental follow up. The method can be extended also to other sequencing techniques that use replicated experiments such as for example ATAC-seq.


# Motivation
ChIP-seq is a standard technology in wet laboratories because it allows to map the genomic regions in which a protein or transcription factor is binding the DNA. Regulatory genomics labs have extensively used this technique to describe how, where and which gene is under the control of a transcription factor. However, the extent of the reproducibility of the binding sites can be confounded by several factors, such as the genomic location in which the transcription factor binds, DNA structures present at the moment of the immunoprecipitation, quality of the antibody as also the experimental conditions in cell culture (2,3,7). For this we have developed IPnoisy, a method that can report noisy transcription factors binding sites from ChIP-seq data in a given cell type of interest. Here the workflow:  

![workflow](https://user-images.githubusercontent.com/6462162/53089925-349c8d00-350e-11e9-87c6-a669741b9968.png)

###### Figure 1) In the workflow: A) ENCODE experiments are selected according to standard parameters (see points from 1 to 6 in the experimental design paragraph), B) peaks are mapped to genomic segments of a defined window size and a sliding window is used to compute a reproducibility score. C) Regions with a specific score are tested for significance and D) PCA is performed to check if the removal of the noisy regions can improve the explanation of the variability of  the samples. 

# Experimental Design: define suitable set of experiments from ENCODE project
We selected ENCODE experiments for four different proteins according to the following standard criteria:  
1) from the same cell line  
2) from the same lab (Snyder)  
3) quality check filter passed (green color)  
4) processed with the same bioinformatics ENCODE pipeline  
5) statistical test performed as Irreproducibility Discovery Rate (IDR)  
6) peaks significant at 5% FDR  

Here we select experiments of 4 proteins and each one with 3 replicates from cell line K562 in the Human Genome annotation hg19, example call:  
`python get_list.py K562 3 4 hg19`  

Here we select experiments of 4 proteins and each one with 3 replicates from cell line HepG2 in the Human Genome annotation hg19 example call:  
 `python get_list.py HepG2 3 4 hg19`  

# Extraction of the experiments, download the files and assign the peaks to genomic segments
The output of the python script is a table with the information of the experiments. We create the folder for each protein and download the files associated in the table. After we create for each protein the matrix with assigned peaks for every genomic segment. Segments can be of 200 or 400 bp length depending on the user choice. 

Segmentation of the genome and assignment of the peaks to genomics bins is performed with this bash script:  

`bash ./Map.Peaks.to.bins.sh`  

# Identification of reproducible and not reproducible regions 
After the identification of suitable experiments and assigned the significant peaks to the genomic bins, we extracted reproducible and not reproducibile regions. We formalized the assignment of reproducible and not reproducible segments according to the following pseudo code rules:

```
Let N be the number of replicates for a given protein;
   let S be the segments for a given genome;
      let P be the number of peaks detected in a genomic segment;
          for every segment in s;
              if P = 0, then assign an NA
                 if in between two NA P is < N, then reproducibility score at each segment is 0
                    else,
              reproducibility score at each segment is 1
      return several lists of reproducible and not reproducible regions where a region is defined as consecutive segments
```

For our study n represents the number of replicates for each protein under investigation in a given cell type, s the segments of the genome considering a window size of 200 base pairs, p is the number of peaks in every genomic segment. Consecutive segments with a signal reaching as a max value n are considered as reproducible regions and assigned with a value of 1. Opposite, consecutive segments with a signal reaching a max value lower than n are considered as not reproducible regions and assigned with a value of 0. The output is a table with a list of regions that are reproducible and not reproducible that will be further aggregated for all the protein under study. Schematic represenation can be observed in the Fig. 2 below.

![fig 2a](https://user-images.githubusercontent.com/6462162/53350621-bb42d700-391f-11e9-89bc-fd092064ad3f.png)

###### Figure 2-A) Steps to identify reproducible and not reproducible regions considering the boarder of each segment and then the tale of each peak for NCOR1 protein. The genome is scanned using a sliding window apporach. Regions that are in between segments with sum vector of 0 are defined as reproducible if the maximum value is three and not reproducible if the maximum value is lower than three.  

For this we have developed three main functions in R that create the vector with the number of signals at each genomic segment (createSumMatrix), create the Id for each segment with the signal (createId) and finally extract the regions with a signal and compute reproducibile and not reproducible regions (getSignalContainingRegions):

Function 1) `createSumMatrix`  

Function 2) `createId`

Function 3) `getSignalContaingRegions`


# Reproducibility Score Matrix (RSM)
Reproducible and not reproducible regions for all the proteins used in the experiments are aggregated in a reproducibility score matrix (Fig.3). 

Function 4) `ReproducibilityScoreMatrix(df1,df2,df3,df4)`  
where df1, df2, df3 and df4 are the matrix with the regions reproducible and not reproducible for each protein

![rsm](https://user-images.githubusercontent.com/6462162/53360397-b9840e00-3935-11e9-87cf-f577b7b8cb6d.png)
###### Figure 2-B) Reproducibility Score Matrix where rows show segments and columns show their conversion score for each protein (1 for segments in regions that are reproducible and 0 for segments in regions that are not reproducible) and a final reproducibility score (RS) defined as the average value of the row (or NA if more than 1 conversion score equals NA)

Afterwards, regions with more than one NA value were discarded and regions with a reproducibility score of 0, that we named noisy, are estimated computing a z-score and respective p.value after 1000 sampling of the reproducibility score matrix. Sampling is performed with the "sample" function in R.
 

# Noisy regions estimation in K562, GM12878, HepG2 and MCF-7 cell lines

A statistical test was computed based on to the computation of a z-score and p.value using 1000 randomizations:  

Function 5) `simulated.pval(n.simulations,cutoff,real.value)`  


![forgith](https://user-images.githubusercontent.com/6462162/46032674-9510d500-c0fc-11e8-8ddc-ea3971f1e075.png)

###### Figure 4) A null distribution is computed for each cell line by sampling the reproducibility score matrix of Fig3. Z-score and P.value is computed for each score. In the picture, represented are the statistical test for DNA regions with reproducibility score 0 (that we renamed Noisy).

# Noisy regions prediction in mESC according to several DNA features
We used the R package "randomforest" to check whether specific genomic regions were predictive of the noisy behaviour for the proteins under investigation. We used a pannel of published datasets and mapped the noisy regions to them. A null model was created with the package gkmSVM and the performance of the algorithm was checked with the package pROC.

The script can be run:  

`Rstudio Random.Forest.r`


![roc for manuscript](https://user-images.githubusercontent.com/6462162/53503254-ff64e180-3aaf-11e9-8982-105edb3166cc.png)
![model random](https://user-images.githubusercontent.com/6462162/53503321-1e637380-3ab0-11e9-81fe-0f05beb48838.png)
###### Figure 5) Random forest algorithm predicts noisy regions in mESCs according to sevral features


# PCA in K562 cell lines with and without the noisy regions
We created a python script using pandas in order to perform the PCA and check whether the removal of noisy peaks improves the separation of the replicates in the PCA.  

The script can be run:  
`python pca.py ./matrix.tsv`  

![pca](https://user-images.githubusercontent.com/6462162/53353427-f267b700-3924-11e9-94d0-669962139ab5.png)
###### Figure 6) PCA shows an improvment in the separation of the groups and respecitve replicates upon removal of the noisy regions 


![intra k562](https://user-images.githubusercontent.com/6462162/53503435-566ab680-3ab0-11e9-91fd-d223dfcaa127.png)
![dotplot](https://user-images.githubusercontent.com/6462162/53503476-68e4f000-3ab0-11e9-880c-bb21e75caf62.png)
###### Figure 7) Euclidean distance of pairwise comparisons between replicates of the same protein as a box plot and as a dot plot 

# References
1) Blackledge, N. P., Zhou, J. C., Tolstorukov, M. Y., Farcas, A. M., Park, P. J., & Klose, R. J. (2010). CpG Islands Recruit a Histone H3 Lysine 36 Demethylase. Molecular Cell, 38(2), 179–190. https://doi.org/10.1016/j.molcel.2010.04.009  

2) HOT or not: Examining the basis of high-occupancy target regions. (n.d.). https://doi.org/https://doi.org/10.1101/107680  
3) Jain, D., Baldi, S., Zabel, A., Straub, T., & Becker, P. B. (2015). Active promoters give rise to false positive “Phantom Peaks” in ChIP-seq experiments. Nucleic Acids Research, 43(14), 6959–6968. https://doi.org/10.1093/nar/gkv637  

4) Kagey, Michae .H Newman, J. J., Bilodeau, S., Zhan, Y., David, A., Berkum, N. L. Van, Ebmeier, C. C., … Dekker, J. (2010). Mediator and Cohesin Connect Gene Expression and Chromatin Architecture. Nature, 467(7314), 430–435. https://doi.org/10.1038/nature09380.Mediator  

5) Landt, S. G., Marinov, G. K., Kundaje, A., Kheradpour, P., Pauli, F., Batzoglou, S., … Snyder, M. (2012). ChIP-seq guidelines and practices of the ENCODE and modENCODE consortia. Genome Research. https://doi.org/10.1101/gr.136184.111  

6) Li, Q., Brown, J. B., Huang, H., & Bickel, P. J. (2011). Measuring reproducibility of high-throughput experiments. Annals of Applied Statistics, 5(3), 1752–1779. https://doi.org/10.1214/11-AOAS466  

7) Park, D., Lee, Y., Bhupindersingh, G., & Iyer, V. R. (2013). Widespread misinterpretable ChIP-seq bias in yeast. PLoS ONE, 8(12), 1–16. https://doi.org/10.1371/journal.pone.0083506  

8) Quinlan, A. R., & Hall, I. M. (2010). BEDTools: A flexible suite of utilities for comparing genomic features. Bioinformatics, 26(6), 841–842. https://doi.org/10.1093/bioinformatics/btq033  

9) Rahl, P. B., Lin, C. Y., Seila, A. C., Flynn, R. A., McCuine, S., Burge, C. B., … Young, R. A. (2010). C-Myc regulates transcriptional pause release. Cell, 141(3), 432–445. https://doi.org/10.1016/j.cell.2010.03.030  

10) Ramachandran, P., Palidwor, G. A., & Perkins, T. J. (2015). BIDCHIPS: Bias decomposition and removal from ChIP-seq data clarifies true binding signal and its functional correlates Medicine. Epigenetics and Chromatin, 8(1). https://doi.org/10.1186/s13072-015-0028-2  

11) Shen, L., Wu, H., Diep, D., Yamaguchi, S., D’Alessio, A. C., Fung, H. L., … Zhang, Y. (2013). Genome-wide analysis reveals TET- and TDG-dependent 5-methylcytosine oxidation dynamics. Cell. https://doi.org/10.1016/j.cell.2013.04.002  

12) Stadler, M. B., Murr, R., Burger, L., Ivanek, R., Lienert, F., Schöler, A., ... & Tiwari, V. K. (2011). (n.d.). DNA-binding factors shape the mouse methylome at distal regulatory regions.  

13) Teytelman, L., Thurtle, D. M., Rine, J., & van Oudenaarden, A. (2013). Highly expressed loci are vulnerable to misleading ChIP localization of multiple unrelated proteins. Proceedings of the National Academy of Sciences, 110(46), 18602–18607. https://doi.org/10.1073/pnas.1316064110  

14) Whyte, W. A., Bilodeau, S., Orlando, D. A., Hoke, H. A., Frampton, G. M., Foster, C. T., … Young, R. A. (2012). Enhancer decommissioning by LSD1 during embryonic stem cell differentiation. Nature, 482(7384), 221–225. https://doi.org/10.1038/nature10805  

15) Wu, H., D’Alessio, A. C., Ito, S., Xia, K., Wang, Z., Cui, K., … Zhang, Y. (2011). Dual functions of Tet1 in transcriptional regulation in mouse embryonic stem cells. Nature, 473(7347), 389–394. https://doi.org/10.1038/nature09934  

16) J, Park, P. (2009). ChIPseq{:}-- advantages and challenges of a maturing technology. Nature Reviews Genetics, 10(10), 669. Retrieved from https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3191340/  

17) Neri, F., Incarnato, D., Krepelova, A., Rapelli, S., Anselmi, F., Parlato, C., … Oliviero, S. (2015). Single-Base resolution analysis of 5-formyl and 5-carboxyl cytosine reveals promoter DNA Methylation Dynamics. Cell Reports, 10(5), 674–683. https://doi.org/10.1016/j.celrep.2015.01.008  
