"Moving Pictures" tutorial
==========================

.. note:: This guide assumes you have installed QIIME 2 using one of the procedures in the :doc:`install documents <../install/index>`.

In this tutorial you'll use QIIME 2 to perform an analysis of human microbiome samples from two individuals at four body sites at five timepoints, the first of which immediately followed antibiotic usage. A study based on these samples was originally published in `Caporaso et al. (2011)`_. The data used in this tutorial were sequenced on an Illumina HiSeq using the `Earth Microbiome Project`_ hypervariable region 4 (V4) 16S rRNA sequencing protocol.

.. qiime1-users::
   These are the same data that are used in the QIIME 1 `Illumina Overview Tutorial`_.

Before beginning this tutorial, create a new directory and change to that directory.

.. command-block::
   :no-exec:

   mkdir qiime2-moving-pictures-tutorial
   cd qiime2-moving-pictures-tutorial

Sample metadata
---------------

Before starting the analysis, explore the sample metadata to familiarize yourself with the samples used in this study. The `sample metadata`_ is available as a Google Sheet. You can download this file as tab-separated text by selecting ``File`` > ``Download as`` > ``Tab-separated values``. Alternatively, the following command will download the sample metadata as tab-separated text and save it in the file ``sample-metadata.tsv``. This ``sample-metadata.tsv`` file is used throughout the rest of the tutorial.

.. download::
   :url: https://data.qiime2.org/2017.2/tutorials/moving-pictures/sample_metadata.tsv
   :saveas: sample-metadata.tsv

.. tip:: `Keemei`_ is a Google Sheets add-on for validating sample metadata. Validation of sample metadata is important before beginning any analysis. Try installing Keemei following the instructions on its website, and then validate the sample metadata spreadsheet linked above. The spreadsheet also includes a sheet with some invalid data to try out with Keemei.

Obtaining and importing data
----------------------------

Download the sequence reads that we'll use in this analysis. In this tutorial we'll work with a small subset of the complete sequence data so that the commands will run quickly.

.. command-block::

   mkdir emp-single-end-sequences

.. download::
   :url: https://data.qiime2.org/2017.2/tutorials/moving-pictures/emp-single-end-sequences/barcodes.fastq.gz
   :saveas: emp-single-end-sequences/barcodes.fastq.gz

.. download::
   :url: https://data.qiime2.org/2017.2/tutorials/moving-pictures/emp-single-end-sequences/sequences.fastq.gz
   :saveas: emp-single-end-sequences/sequences.fastq.gz

All data that is used as input to QIIME 2 is in form of QIIME 2 artifacts, which contain information about the type of data and the source of the data. So, the first thing we need to do is import these sequence data files into a QIIME 2 artifact.

The semantic type of this QIIME 2 artifact is ``EMPSingleEndSequences``. ``EMPSingleEndSequences`` QIIME 2 artifacts contain sequences that are multiplexed, meaning that the sequences have not yet been assigned to samples (hence the inclusion of both ``sequences.fastq.gz`` and ``barcodes.fastq.gz`` files, where the ``barcodes.fastq.gz`` contains the barcode read associated with each sequence in ``sequences.fastq.gz``.) To learn about how to import sequence data in other formats, see the :doc:`importing data tutorial <importing>`.

.. command-block::

   qiime tools import \
     --type EMPSingleEndSequences \
     --input-path emp-single-end-sequences \
     --output-path emp-single-end-sequences.qza

.. tip::
   Links are included to view and download precomputed QIIME 2 artifacts and visualizations created by commands in the documentation. For example, the command above created a single ``emp-single-end-sequences.qza`` file, and a corresponding precomputed file is linked above. You can view precomputed QIIME 2 artifacts and visualizations without needing to install additional software (e.g. QIIME 2).



.. qiime1-users::
   In QIIME 1, we generally suggested performing demultiplexing through QIIME (e.g., with ``split_libraries.py`` or ``split_libraries_fastq.py``) as this step also performed quality control of sequences. We now separate the demultiplexing and quality control steps, so you can begin QIIME 2 with either multiplexed sequences (as we're doing here) or demultiplexed sequences.

Demultiplexing sequences
------------------------

To demultiplex sequences we need to know which barcode sequence is associated with each sample. This information is contained in the `sample metadata`_ file. You can run the following commands to demultiplex the sequences (the ``demux emp-single`` command refers to the fact that these sequences are barcoded according to the `Earth Microbiome Project`_ protocol, and are single-end reads) and then generate a summary of the demultiplexing results. The ``demux.qza`` QIIME 2 artifact will contain the demultiplexed sequences.

.. command-block::

    qiime demux emp-single \
      --i-seqs emp-single-end-sequences.qza \
      --m-barcodes-file sample-metadata.tsv \
      --m-barcodes-category BarcodeSequence \
      --o-per-sample-sequences demux.qza
    qiime demux summarize \
      --i-data demux.qza \
      --o-visualization demux.qzv

Sequence quality control
------------------------

We'll next perform quality control on the demultiplexed sequences using `DADA2`_. DADA2 is a pipeline for detecting and correcting (where possible) Illumina amplicon sequence data. As implemented in the ``q2-dada2`` plugin, this quality control process will additionally filter any phiX reads (commonly present in marker gene Illumina sequence data) that are identified in the sequencing data, and will filter chimeric sequences. The result of this step will be a ``FeatureTable[Frequency]`` QIIME 2 artifact, which contains counts (frequencies) of each unique sequence in each sample in the dataset, and a ``FeatureData[Sequence]`` QIIME 2 artifact, which maps feature identifiers in the ``FeatureTable`` to the sequences they represent.

.. qiime1-users::
   The ``FeatureTable[Frequency]`` QIIME 2 artifact is the equivalent of the QIIME 1 OTU or BIOM table, and the ``FeatureData[Sequence]`` QIIME 2 artifact is the equivalent of the QIIME 1 *representative sequences* file. Because the "OTUs" resulting from DADA2 are creating by grouping unique sequences, these are the equivalent of 100% OTUs from QIIME 1. In DADA2, these 100% OTUs are referred to as *denoised sequence variants*. In QIIME 2, these OTUs are higher resolution than the QIIME 1 default of 97% OTUs, and they're higher quality due to the DADA2 denoising process. This should therefore result in more accurate estimates of diversity and taxonomic composition of samples than was achieved with QIIME 1.

The ``dada2 denoise-single`` method requires two parameters that are used in quality filtering: ``--p-trim-left m``, which trims off the first ``m`` bases of each sequence, and ``--p-trunc-len n`` which truncates each sequence at position ``n``. This allows the user to remove low quality regions of the sequences. To determine what values to pass for these two parameters, you should first run the ``dada2 plot-qualities`` visualizer, which will generate plots of the quality scores by position for a randomly selected set of samples. In the following command, we'll generate a quality plot using 10 randomly selected samples (specified by passing ``--p-n 10``).

.. command-block::

   qiime dada2 plot-qualities \
     --i-demultiplexed-seqs demux.qza \
     --p-n 10 \
     --o-visualization demux-qual-plots.qzv


.. note::
   All QIIME 2 visualizers (i.e., commands that take a ``--o-visualization`` parameter) will generate a ``.qzv`` file. You can view these files with ``qiime tools view``. We provide the command to view this first visualization, but for the remainder of this tutorial we'll tell you to *view the resulting visualization* after running a visualizer, which means that you should run ``qiime tools view`` on the .qzv file that was generated.

   .. command-block::
      :no-exec:

      qiime tools view demux-qual-plots.qzv

   Alternatively, you can view QIIME 2 artifacts and visualizations at `view.qiime2.org <https://view.qiime2.org>`__ by uploading files or providing URLs. There are also precomputed results linked above that can be viewed or downloaded.

.. question::
   Based on the plots you see in ``demux-qual-plots.qzv``, what values would you choose for ``--p-trunc-len`` and ``--p-trim-left`` in this case?

In these plots, the quality of the initial bases seems to be high, so we won't trim any bases from the beginning of the sequences. The quality seems to drop off around position 100, so we'll truncate our sequences at 100 bases. This next command may take up to 10 minutes to run, and is the slowest step in this tutorial.

.. command-block::

   qiime dada2 denoise-single \
     --i-demultiplexed-seqs demux.qza \
     --p-trim-left 0 \
     --p-trunc-len 100 \
     --o-representative-sequences rep-seqs.qza \
     --o-table table.qza

After the ``dada2 denoise-single`` step completes, you'll want to explore the resulting data. You can do this using the following two commands, which will create visual summaries of the data. The ``feature-table summarize`` command will give you information on how many sequences are associated with each sample and with each feature, histograms of those distributions, and some related summary statistics. The ``feature-table tabulate-seqs`` command will provide a mapping of feature IDs to sequences, and provide links to easily BLAST each sequence against the NCBI nt database. The latter visualization will be very useful later in the tutorial, when you want to learn more about specific features that are important in the data set.

.. command-block::

   qiime feature-table summarize \
     --i-table table.qza \
     --o-visualization table.qzv \
     --m-sample-metadata-file sample-metadata.tsv
   qiime feature-table tabulate-seqs \
     --i-data rep-seqs.qza \
     --o-visualization rep-seqs.qzv

Generate a tree for phylogenetic diversity analyses
---------------------------------------------------

QIIME supports several phylogenetic diversity metrics, including Faith's Phylogenetic Diversity and weighted and unweighted UniFrac. In addition to counts of features per sample (i.e., the data in the ``FeatureTable[Frequency]`` QIIME 2 artifact), these metrics require a rooted phylogenetic tree relating the features to one another. This information will be stored in a ``Phylogeny[Rooted]`` QIIME 2 artifact. The following steps will generate this QIIME 2 artifact.

First, we perform a multiple sequence alignment of the sequences in our ``FeatureData[Sequence]`` to create a ``FeatureData[AlignedSequence]`` QIIME 2 artifact. Here we do this with the ``mafft`` program.

.. command-block::

   qiime alignment mafft \
     --i-sequences rep-seqs.qza \
     --o-alignment aligned-rep-seqs.qza

Next, we mask (or filter) the alignment to remove positions that are highly variable. These positions are generally considered to add noise to a resulting phylogenetic tree.

.. command-block::

   qiime alignment mask \
     --i-alignment aligned-rep-seqs.qza \
     --o-masked-alignment masked-aligned-rep-seqs.qza

Next, we'll apply FastTree to generate a phylogenetic tree from the masked alignment.

.. command-block::

   qiime phylogeny fasttree \
     --i-alignment masked-aligned-rep-seqs.qza \
     --o-tree unrooted-tree.qza

The FastTree program creates an unrooted tree, so in the final step in this section we apply midpoint rooting to place the root of the tree at the midpoint of the longest tip-to-tip distance in the unrooted tree.

.. command-block::

   qiime phylogeny midpoint-root \
     --i-tree unrooted-tree.qza \
     --o-rooted-tree rooted-tree.qza

Alpha and beta diversity analysis
---------------------------------

QIIME 2's diversity analyses are available through the ``q2-diversity`` plugin, which supports computing alpha and beta diversity metrics, applying related statistical tests, and generating interactive visualizations. We'll first apply the ``core-metrics`` method, which rarefies a ``FeatureTable[Frequency]`` to a user-specified depth, and then computes a series of alpha and beta diversity metrics. The metrics computed by default are:

* Alpha diversity

  * Shannon's diversity index (a quantitative measure of community richness)
  * Observed OTUs (a qualitative measure of community richness)
  * Faith's Phylogenetic Diversity (a qualitiative measure of community richness that incorporates phylogenetic relationships between the features)
  * Evenness (or Pielou's Evenness; a measure of community evenness)

* Beta diversity

  * Jaccard distance (a qualitative measure of community dissimilarity)
  * Bray-Curtis distance (a quantitative measure of community dissimilarity)
  * unweighted UniFrac distance (a qualitative measure of community dissimilarity that incorporates phylogenetic relationships between the features)
  * weighted UniFrac distance (a quantitative measure of community dissimilarity that incorporates phylogenetic relationships between the features)

The only parameter that needs to be provided to this script is ``--p-sampling-depth``, which is the even sampling (i.e. rarefaction) depth. Because most diversity metrics are sensitive to different sampling depths across different samples, this script will randomly subsample the counts from each sample to the value provided for this parameter. For example, if you provide ``--p-sampling-depth 500``, this step will subsample the counts in each sample without replacement so that each sample in the resulting table has a total count of 500. If the total count for any sample(s) are smaller than this value, those samples will be dropped from the diversity analysis. Choosing this value is tricky. We recommend making your choice by reviewing the information presented in the ``table.qzv`` file that was created above and choosing a value that is as high as possible (so you retain more sequences per sample) while excluding as few samples as possible.

.. question::
   View the ``table.qzv`` QIIME 2 artifact, and in particular the *Interactive Sample Detail* tab in that visualization. What value would you choose to pass for ``--p-sampling-depth``? How many samples will be excluded from your analysis based on this choice? Approximately how many total sequences will you be analyzing in the ``core-metrics`` command?

.. command-block::

   qiime diversity core-metrics \
     --i-phylogeny rooted-tree.qza \
     --i-table table.qza \
     --p-sampling-depth 1441 \
     --output-dir cm1441

Here we set the ``--p-sampling-depth`` parameter to 1441. This value was chosen here because it's nearly the same number of sequences as the next few samples, and because it is the lowest value it will allow us to retain all of our samples. In many Illumina runs however you'll observe a few samples that have much lower sequence counts. You will typically want to exclude those from the analysis by choosing a larger value.

After computing diversity metrics, we can begin to explore the microbial composition of the samples in the context of the sample metadata. This information is present in the `sample metadata`_ file that was downloaded earlier.

We'll first test for associations between discrete metadata categories and alpha diversity data. We'll do that here for the Faith Phylogenetic Diversity (a measure of community richness) and evenness metrics.

.. command-block::

   qiime diversity alpha-group-significance \
     --i-alpha-diversity cm1441/faith_pd_vector.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization cm1441/faith-pd-group-significance.qzv

   qiime diversity alpha-group-significance \
     --i-alpha-diversity cm1441/evenness_vector.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization cm1441/evenness-group-significance.qzv

.. question::
   What discrete sample metadata categories are most strongly associated with the differences in microbial community **richness**? Are these differences statistically significant?

.. question::
   What discrete sample metadata categories are most strongly associated with the differences in microbial community **evenness**? Are these differences statistically significant?

Next, we'll test for associations between alpha diversity metrics and continuous sample metadata (such as pH or elevation). We can do this running the following two commands, which will support analysis of Faith's Phylogenetic Diversity metric and evenness in the context of our continuous metadata. Run these commands and view the resulting QIIME 2 artifacts.

.. command-block::

   qiime diversity alpha-correlation \
     --i-alpha-diversity cm1441/faith_pd_vector.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization cm1441/faith-pd-correlation.qzv

   qiime diversity alpha-correlation \
     --i-alpha-diversity cm1441/evenness_vector.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization cm1441/evenness-correlation.qzv

.. question::
   What do you conclude about the associations between continuous sample metadata and the richness and evenness of these samples?

Next we'll analyze sample composition in the context of discrete metadata using PERMANOVA (first described in `Anderson (2001)`_) using the ``beta-group-significance`` command. The following commands will test whether distances between samples within a group, such as samples from the same body site (e.g., skin or gut), are more similar to each other then they are to samples from a different group. This command can be slow to run since it is based on permutation tests, so unlike the previous commands we'll run this on specific categories of metadata that we're interested in exploring, rather than all metadata categories that it's applicable to. Here we'll apply this to our unweighted UniFrac distances, using two sample metadata categories, as follows.

.. command-block::

   qiime diversity beta-group-significance \
     --i-distance-matrix cm1441/unweighted_unifrac_distance_matrix.qza \
     --m-metadata-file sample-metadata.tsv \
     --m-metadata-category BodySite \
     --o-visualization cm1441/unweighted-unifrac-body-site-significance.qzv

   qiime diversity beta-group-significance \
     --i-distance-matrix cm1441/unweighted_unifrac_distance_matrix.qza \
     --m-metadata-file sample-metadata.tsv \
     --m-metadata-category Subject \
     --o-visualization cm1441/unweighted-unifrac-subject-group-significance.qzv

.. question::
   Are the associations between subjects and differences in microbial composition statistically significant? How about body sites? What body sites appear to be most different from each other?

Finally, we'll explore associations between the microbial composition of the samples and continuous sample metadata using ``bioenv`` (originally described in `Clarke and Ainsworth (1993)`_). This approach tests for associations of pairwise distances between sample microbial composition (a measure of beta diversity) and sample metadata (for example, the matrix of Bray-Curtis distances between samples and the matrix of absolute differences in pH between samples). A powerful feature of this method is that it explores combinations of sample metadata to see which groups of metadata differences are most strongly associated with the observed microbial differences between samples. You can apply ``bioenv`` to the unweighted UniFrac distances and Bray-Curtis distances between the samples, respectively, as follows. After running these commands, open the resulting visualizations.

.. command-block::

   qiime diversity bioenv \
     --i-distance-matrix cm1441/unweighted_unifrac_distance_matrix.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization cm1441/unweighted-unifrac-bioenv.qzv

   qiime diversity bioenv \
     --i-distance-matrix cm1441/bray_curtis_distance_matrix.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization cm1441/bray-curtis-bioenv.qzv

.. question::
   What sample metadata or combinations of sample metadata are most strongly associated with the differences in microbial composition of the samples? How strong are these correlations?

Finally, ordination is a popular approach for exploring microbial community composition in the context of sample metadata. We can use the `Emperor`_ tool to explore principal coordinates (PCoA) plots in the context of sample metadata. PCoA is run as part of the ``core-metrics`` command, so we can generate these plots for unweighted UniFrac and Bray-Curtis as follows. The ``--p-custom-axis`` parameter that we pass here is very useful for exploring temporal data. The resulting plot will contain axes for principal coordinate 1 (labelled ``0``), principal coordinate 2 (labelled ``1``), and days since the experiment start. This is useful for exploring how the samples change over time.

.. command-block::

   qiime emperor plot \
     --i-pcoa cm1441/unweighted_unifrac_pcoa_results.qza \
     --m-metadata-file sample-metadata.tsv \
     --p-custom-axis DaysSinceExperimentStart \
     --o-visualization cm1441/unweighted-unifrac-emperor.qzv

   qiime emperor plot \
     --i-pcoa cm1441/bray_curtis_pcoa_results.qza \
     --m-metadata-file sample-metadata.tsv \
     --p-custom-axis DaysSinceExperimentStart \
     --o-visualization cm1441/bray-curtis-emperor.qzv

.. question::
    Do the Emperor plots support the other beta diversity analyses we've performed here? (Hint: Experiment with coloring points by different metadata.)

.. question::
    What differences do you observe between the unweighted UniFrac and Bray-Curtis PCoA plots?

Taxonomic analysis
------------------

In the next sections we'll begin to explore the taxonomic composition of the samples, and again relate that to sample metadata. The first step in this process is to assign taxonomy to the sequences in our ``FeatureData[Sequence]`` QIIME 2 artifact. We'll do that using a pre-trained Naive Bayes classifier and the ``q2-feature-classifier`` plugin. This classifier was trained on the Greengenes 13_8 99% OTUs, where the sequences have been trimmed to only include 250 bases from the region of the 16S that was sequenced in this analysis (the V4 region, bound by the 515F/806R primer pair). We'll apply this classifier to our sequences, and we can generate a visualization of the resulting mapping from sequence to taxonomy.

.. note:: Taxonomic classifiers perform best when they are trained based on your specific sample preparation and sequencing parameters, including the primers that were used for amplification and the length of your sequence reads. Therefore in general you should follow the instructions in :doc:`Training feature classifiers with q2-feature-classifier <../tutorials/feature-classifier>` to train your own taxonomic classifiers. We provide some common classifiers on our :doc:`data resources page <../data-resources>`, including Silva-based 16S classifiers, though in the future we may stop providing these in favor of having users train their own classifiers which will be most relevant to their sequence data.


.. download::
   :url: https://data.qiime2.org/2017.2/common/gg-13-8-99-515-806-nb-classifier.qza
   :saveas: gg-13-8-99-515-806-nb-classifier.qza

.. command-block::

   qiime feature-classifier classify \
     --i-classifier gg-13-8-99-515-806-nb-classifier.qza \
     --i-reads rep-seqs.qza \
     --o-classification taxonomy.qza

   qiime taxa tabulate \
     --i-data taxonomy.qza \
     --o-visualization taxonomy.qzv

.. question::
    Recall that our ``rep-seqs.qzv`` visualization allows you to easily BLAST the sequence associated with each feature against the NCBI nt database. Using that visualization and the ``taxonomy.qzv`` visualization created here, compare the taxonomic assignments with the taxonomy of the best BLAST hit for a few features. How similar are the assignments? If they're dissimilar, at what *taxonomic level* do they begin to differ (e.g., species, genus, family, ...)?

Next, we can view the taxonomic composition of our samples with interactive bar plots. Generate those plots with the following command and then open the visualization.

.. command-block::

   qiime taxa barplot \
     --i-table table.qza \
     --i-taxonomy taxonomy.qza \
     --m-metadata-file sample-metadata.tsv \
     --o-visualization taxa-bar-plots.qzv

.. question::
    Visualize the samples at *Level 2* (which corresponds to the phylum level in this analysis), and then sort the samples by BodySite, then by Subject, and then by DaysSinceExperimentStart. What are the dominant phyla in each in BodySite? Do you observe any consistent change across the two subjects between DaysSinceExperimentStart ``0`` and the later timepoints?

Differential abundance analysis
-------------------------------

Finally, we can quantify the process of identifying taxa that are differentially abundance (or present in different abundances) across sample groups. We do that using ANCOM (`Mandal et al. (2015)`_), which is implemented in the ``q2-composition`` plugin. ANCOM operates on a ``FeatureTable[Composition]`` QIIME 2 artifact, which is based on relative frequencies of features on a per-sample basis, but cannot tolerate frequencies of zero. We work around this by adding a pseudocount of 1 to every count in our ``FeatureTable[Frequency]`` table. We can run this on the ``BodySite`` category to determine what features differ in abundance across body sites. This step may take about 5 minutes to complete.

.. command-block::

   qiime composition add-pseudocount \
     --i-table table.qza \
     --o-composition-table comp-table.qza

   qiime composition ancom \
     --i-table comp-table.qza \
     --m-metadata-file sample-metadata.tsv \
     --m-metadata-category BodySite \
     --o-visualization ancom-BodySite.qzv

.. question::
    What features differ in abundance across BodySite? What groups are they most and least abundant in? What are the taxonomies of some of these features? (To answer that last question you'll need to refer to a visualization that we generated earlier in this tutorial.)

We're also often interested in performing a differential abundance test at a specific taxonomic level. To do this, we can collapse the features in our ``FeatureTable[Frequency]`` at the taxonomic level of interest, and then re-run the above steps.

.. command-block::

   qiime taxa collapse \
     --i-table table.qza \
     --i-taxonomy taxonomy.qza \
     --p-level 2 \
     --o-collapsed-table table-l2.qza

   qiime composition add-pseudocount \
     --i-table table-l2.qza \
     --o-composition-table comp-table-l2.qza

   qiime composition ancom \
     --i-table comp-table-l2.qza \
     --m-metadata-file sample-metadata.tsv \
     --m-metadata-category BodySite \
     --o-visualization l2-ancom-BodySite.qzv

.. question::
    What phyla differ in abundance across BodySite? How does this align with what you observed in the ``taxa-bar-plots.qzv`` visualization that was generated above?

.. _sample metadata: https://data.qiime2.org/2017.2/tutorials/moving-pictures/sample_metadata
.. _Keemei: http://keemei.qiime.org
.. _DADA2: https://www.ncbi.nlm.nih.gov/pubmed/27214047
.. _Illumina Overview Tutorial: http://nbviewer.jupyter.org/github/biocore/qiime/blob/1.9.1/examples/ipynb/illumina_overview_tutorial.ipynb
.. _Caporaso et al. (2011): https://www.ncbi.nlm.nih.gov/pubmed/21624126
.. _Earth Microbiome Project: http://earthmicrobiome.org
.. _Clarke and Ainsworth (1993): http://www.int-res.com/articles/meps/92/m092p205.pdf
.. _PERMANOVA: http://onlinelibrary.wiley.com/doi/10.1111/j.1442-9993.2001.01070.pp.x/full
.. _Anderson (2001): http://onlinelibrary.wiley.com/doi/10.1111/j.1442-9993.2001.01070.pp.x/full
.. _Emperor: http://emperor.microbio.me
.. _Bergmann et al. (2011): https://www.ncbi.nlm.nih.gov/pubmed/22267877
.. _Mandal et al. (2015): https://www.ncbi.nlm.nih.gov/pubmed/26028277
