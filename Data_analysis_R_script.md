# Data analysis in R
## Study: Metabarcoding analysis of fungi associated with roots of three plant species: _Diphasiastrum complanatum_, _Pinus sylvestris_ and _Vaccinium myrtillus_

### Load packages
```{r, message= FALSE, warning = FALSE}
library("phyloseq")
library("vegan")
library("DESeq2")
library("ggplot2")
library("dendextend")
library("tidyr")
library("viridis")
library("reshape")
library("metacoder")
library("dplyr")
library('readr')
library('stringr')
library('agricolae')
library('ape')
library('ggrepel')
library("RColorBrewer")
library('fungaltraits')
library('rstatix')
library('microeco')
library('ggalluvial')
library('metacoder')
library('iNEXT')

```

### Reading  data

```

count_tab <-read.table("ASV_counts_new.txt", header=T, row.names=1, check.names=F)
sample_info <- read.table("info.txt", header=T, row.names=1, check.names=F)

tax_tab<- read.delim("Taxonomy.txt")
#tax_tab$ASV[duplicated(tax_tab$ASV)] # check for duplicated ASVs

count_tab <- count_tab[rownames(count_tab) %in% tax_tab$ASV, ]

head(count_tab)
head(tax_tab)
head(sample_info)

#to count the number of species:
#unique_Species<- unique(tax_data$Species)
#print(unique_Species)
```

### Treatment of NTC

```
# sum of NTC counts for each ASV
NTC <-rowSums(count_tab[, 1:7]) 

# sum of sample counts for each ASV
sample_counts <- rowSums(count_tab[, 8:91])

# normalize sample counts by dividing the samples' total by 12 – as there are 12 as many samples (84) as there are NTCs (7) (84/7=12)
norm_sample_ASV_counts <- sample_counts/12

# which ASVs are deemed likely contaminants based on the threshold noted above:
blank_ASVs <- names(NTC[NTC * 10  > norm_sample_ASV_counts]) # ASVs which have more counts than 1/10 of the sample counts 
length(blank_ASVs) # this approach identified  6 out of ~2115 ASVs that are likely to have originated from contamination

# looking at the percentage of reads retained for each sample after removing these presumed contaminant ASVs 
colSums(count_tab[!rownames(count_tab) %in% blank_ASVs, ]) / colSums(count_tab) * 100

filt_count_tab <- count_tab[!rownames(count_tab) %in% blank_ASVs, -c(1:7)] # removing blank_ASVs and the blank samples from further analysis
filt_count_tab <- filt_count_tab[rowSums(filt_count_tab == 0) != ncol(filt_count_tab), ] # delete ASVs with 0 counts -> 2095 ASVs left

```

### Normalizing counts for sampling depth (variance stabilizing transformation in DEseq2 package)
```{r, warning = FALSE}

#reorder the row and column names
sample_info<-sample_info[order(rownames(sample_info)), ]
filt_count_tab<- filt_count_tab[,order(names(filt_count_tab))]

#add 1 to every count, so it is possible to calculate the geometric mean later
filt_count_tab1 <- filt_count_tab + 1 

# create a DESeq2 object
deseq_counts <- DESeqDataSetFromMatrix(filt_count_tab1, colData = sample_info, design = ~Species) 
deseq_counts_vst <- varianceStabilizingTransformation(deseq_counts, fitType='local')

#save as a matrix
vst_trans_count_tab <- assay(deseq_counts_vst)

#transform lowest values (absent counts) to 0
vst_trans_count_tab <- sweep(vst_trans_count_tab, 2, apply(vst_trans_count_tab, 2, min), "-") 

```

```{r, warning =FALSE}

# add colors specific to species to the sample info table
sample_info$color[sample_info$Species == "Pinus"] <- "dodgerblue3"
sample_info$color[sample_info$Species == "Diphasiastrum"] <- "darkorange1"
sample_info$color[sample_info$Species == "Vaccinium"] <- "purple4"
sample_info$color[sample_info$Species == "Soil"] <- "hotpink2"

# Create a binary dataset in case if you want to compare normalised dataset with thepresence/absense dataset - in my case, there was not so much difference.
#binary <- as.matrix((filt_count_tab>0)+0)
#binary_df <- data.frame(binary)

# I continue the analysis with only normalaised data
```


### PCoA - all ASVs

```{r, warning =FALSE}

# create phyloseq object with normalized data
vst_count_phy <- otu_table(vst_trans_count_tab, taxa_are_rows=T)
sample_info_tab_phy <- sample_data(sample_info)
vst_physeq <- phyloseq(vst_count_phy, sample_info_tab_phy)

#generating and visualizing the PCoA with phyloseq
vst_pcoa <- ordinate(vst_physeq, method="MDS", distance="euclidean")
eigen_vals <- vst_pcoa$values$Eigenvalues # allows us to scale the axes according to their magnitude of separating apart the samples

pca<-plot_ordination(vst_physeq, vst_pcoa, color="Species") + 
  labs(col="Species") + geom_point(size=1) + 
  geom_text_repel(aes(label=rownames(sample_info)), size =3, force = 5, max.overlaps = 50) + 
  coord_fixed(sqrt(eigen_vals[2]/eigen_vals[1])) + ggtitle("PCoA") + 
  scale_color_manual(values=unique(sample_info$color[order(sample_info$Species)])) + 
  theme(legend.position="none")
pca


```

### PCoA - only ASVs of Ectomycorrhizal (ECM) fungi and Endophytes

```{r, warning =FALSE}

# filter only ECM and Endophytes:

matching_rows <- tax_tab$ASV[tax_tab$Guild %in% c("Ectomycorrhizal", "Endophyte")]

# Subset the matrix based on the matching row names
filtered_vs_trans_count_tab <- vst_trans_count_tab[matching_rows, ]


# create phyloseq object with normalized data
vst_count_phy <- otu_table(filtered_vs_trans_count_tab, taxa_are_rows=T)
sample_info_tab_phy <- sample_data(sample_info)
vst_physeq <- phyloseq(vst_count_phy, sample_info_tab_phy)


#generating and visualizing the PCoA with phyloseq
vst_pcoa <- ordinate(vst_physeq, method="MDS", distance="euclidean")
eigen_vals <- vst_pcoa$values$Eigenvalues # allows us to scale the axes according to their magnitude of separating apart the samples

pca<-plot_ordination(vst_physeq, vst_pcoa, color="Species") + 
  labs(col="Species") + geom_point(size=1) + 
  geom_text_repel(aes(label=rownames(sample_info)), size =3, force = 5, max.overlaps = 50) + 
  coord_fixed(sqrt(eigen_vals[2]/eigen_vals[1])) + ggtitle("PCoA") + 
  scale_color_manual(values=unique(sample_info$color[order(sample_info$Species)])) + 
  theme(legend.position="none")
pca

```

# Taxonomic summaries

### Phylum

```{r, warning =FALSE}

#read info table with the samples without replicates
sample_info_no_rep <- read.table("Info_noR.txt", header=T, row.names=1, check.names=F)

tax_table<-tax_tab
row.names(tax_table) <- tax_table$ASV
tax_table <- tax_table[, -1]
tax_table<- tax_table[order(rownames(tax_table)),]

# create a data frame from normalized data
#to get rid of negative values, the data were transformed (the lowest value was subtracted from each value)

counts_norm_df<-as.data.frame(vst_trans_count_tab)
row_names <- rownames(counts_norm_df)
counts_norm_df <- lapply(counts_norm_df, function(col) col - min(counts_norm_df))
counts_norm_df <-as.data.frame(counts_norm_df)
rownames(counts_norm_df) <- row_names


# create microeco file
meco_fungi <- microtable$new(sample_table = sample_info_no_rep, otu_table = counts_norm_df, tax_table = tax_table)
 
# color palette
mycol <- c("#A6CEE3" ,"#1F78B4" ,"#B2DF8A" ,"#33A02C" ,"#FB9A99", "#E31A1C" ,"#FDBF6F" ,"#FF7F00" ,"#CAB2D6" ,"#6A3D9A" ,"#FFFF99" ,"#B15928" , "#6fdc8c", "green4", "#004144")

t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Phylum", ntaxa = 15)

t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)

```

### Order 

```{r}
t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Order", ntaxa = 15)

t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)

```


### Per forest - Order 

```{r}
t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Forest",xtext_size = 6, color_values = mycol)

# No big difference between parts of the forest
```

### Class 

```{r}
t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Class", ntaxa = 15)

t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)

```

### Family

```{r}
t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Family", ntaxa = 15)

t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)
```

### Genus

```{r}
t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Genus", ntaxa = 15)

t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)
```

### Species

```{r}
t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Species", ntaxa = 15)
t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)
```


## Heat tree

### Package "metacoder"
https://grunwaldlab.github.io/metacoder_documentation/workshop--04--manipulating.html

### All taxa, all samples

```{r, warning =FALSE}

tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

# Read sample info table without technical replicates
sample_data <-read.table("info_noR.txt", sep = "\t", header=T, check.names=F)

# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df, tax_data, by = "ASV" ) 

#converting to taxmap format
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy",
                      class_sep = ";")

#remove unclassified taxa 
obj <- filter_taxa(obj, taxon_names != "-")

set.seed(50)
heattree<-heat_tree(obj, node_label = gsub(pattern = "\\[|\\]", replacement = "", taxon_names),
            node_size = n_obs,
            node_size_range = c(0.005, 0.04), 
            node_color = n_obs,
            node_color_range = c("#C9E4CA", "#87BBA2", "#55828B", "#3B6064", "#364958"),
            #edge_label = n_obs,
            #edge_label_size_range = c(0.01, 0.02), 
            node_label_size_range = c(0.01, 0.02), 
            node_color_axis_label = "ASV count",
            layout = "davidson-harel", initial_layout = "reingold-tilford")

plot(heattree)
```

## Heat tree  (all ASVs, 252 taxa)

### Comparison of sample types

```{r, warning =FALSE}

tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column


# Read sample info table without technical replicates
sample_data <-read.table("info_noR.txt", sep = "\t", header=T, check.names=F)

# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df, tax_data, by = "ASV" ) 
Tax_otu_data  <- subset(Tax_otu_data , select = -grep("_", names(Tax_otu_data ))) #delete technical replicates


#converting to taxmap format
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy", 
                      class_sep = ";") # 

#remove unclassified taxa:
obj <- filter_taxa(obj, taxon_names != "-")


#For the heat tree comparison of abundances
obj$data$tax_abund <- calc_taxon_abund(obj, "tax_data", cols = sample_data$Sample)
obj$data$tax_occ <- calc_n_samples(obj, "tax_abund", groups = sample_data$Species, cols = sample_data$Sample)
obj$data$diff_table <- compare_groups(obj, data= "tax_abund", 
                                      cols = sample_data$Sample, # What columns of sample data to use
                                      groups = sample_data$Species) # What category each sample is assigned to

#We then need to correct for multiple comparisons and set non-significant differences to zero:
obj <- mutate_obs(obj, "diff_table",
                  wilcox_p_value = p.adjust(wilcox_p_value, method = "fdr")) #false discovery rate
obj$data$diff_table$log2_median_ratio[obj$data$diff_table$wilcox_p_value > 0.05] <- 0


# To print heat tree for group comparison:
set.seed(50)
heattree<-heat_tree_matrix(obj, key_size = 0.65,
                 data = "diff_table",
                 node_size = n_obs, # the number of ASVs per taxon
                 node_label = taxon_names,
                 node_color = log2_median_ratio, 
                 node_color_range = diverging_palette(), 
                 node_color_trans = "linear", 
                 node_color_interval = c(-2, 2), # The range of `log2_median_ratio` 
                 edge_color_interval = c(-2, 2), # The range of `log2_median_ratio`
                 node_size_axis_label = "Number of ASVs",
                 node_color_axis_label = "Log2 ratio median proportions",
                 layout = "davidson-harel", # The primary layout algorithm
                 initial_layout = "reingold-tilford") # The layout algorithm that initializes node locations
plot(heattree)
#The grey tree on the lower left functions as a key for the unlabeled trees. 
#Each of the smaller trees represent a comparison between Species in the columns and rows. 
#A taxon colored brown is more abundant in the species in the column and a taxon colored green is more abundant in species of the row. 
```

## Heat tree (only abundant taxa -> easier to see the taxa)
### Comparison of sample types

```{r, warning =FALSE}

tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

sample_data <-read.table("info_noR.txt", sep = "\t", header=T, check.names=F)

# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df, tax_data, by = "ASV" ) 
Tax_otu_data  <- subset(Tax_otu_data , select = -grep("_", names(Tax_otu_data ))) #delete technical replicates


#converting to taxmap format
library(metacoder)
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy", # The column in the input table
                      class_sep = ";") # What each taxon is separated by

#remove  _ (unclassified taxa) and rare taxa:
obj <- filter_taxa(obj, taxon_names != "-")
obj <- filter_taxa(obj, n_obs > 9)


#For the heat tree comparison of abundances
obj$data$tax_abund <- calc_taxon_abund(obj, "tax_data", cols = sample_data$Sample)
obj$data$tax_occ <- calc_n_samples(obj, "tax_abund", groups = sample_data$Species, cols = sample_data$Sample)
obj$data$diff_table <- compare_groups(obj, data= "tax_abund", 
                                      cols = sample_data$Sample, # What columns of sample data to use
                                      groups = sample_data$Species) # What category each sample is assigned to

#We then need to correct for multiple comparisons and set non-significant differences to zero:
obj <- mutate_obs(obj, "diff_table",
                  wilcox_p_value = p.adjust(wilcox_p_value, method = "fdr")) #false discovery rate
obj$data$diff_table$log2_median_ratio[obj$data$diff_table$wilcox_p_value > 0.05] <- 0


# To print heat tree for group comparison:
set.seed(50)
heattree<-heat_tree_matrix(obj, key_size = 0.65,
                 data = "diff_table",
                 node_size = n_obs, # the number of ASVs per taxon
                 node_label = taxon_names,
                 node_color = log2_median_ratio, 
                 node_color_range = diverging_palette(), 
                 node_color_trans = "linear", 
                 node_color_interval = c(-2, 2), # The range of `log2_median_ratio` 
                 edge_color_interval = c(-2, 2), # The range of `log2_median_ratio`
                 node_size_axis_label = "Number of ASVs",
                 node_color_axis_label = "Log2 ratio median proportions",
                 layout = "davidson-harel", # The primary layout algorithm
                 initial_layout = "reingold-tilford") # The layout algorithm that initializes node locations

plot(heattree)

```

## iNEXT package - diversity indices

```{r}

# Hill number = effective number of species.  Hill numbers determines the measures’ sensitivity to species relative abundances.  Hill numbers include the three most widely used species diversity measures as special cases: species richness (q = 0), Shannon diversity (q = 1, as the effective number of common species in the assemblage) and Simpson diversity (q = 2, effective number of dominant species in the assemblage)


# I used normalized counts after NTCs treatments and without technical replicates
count_tab <- counts_norm_df[, unlist(lapply(counts_norm_df, is.numeric))] 
row.names(count_tab) <- count_tab$ASV.ID
count_tab <- subset(count_tab, select = -grep("_", names(count_tab))) #delete technical replicates
count_tab <- count_tab > 0

# get ASV incidence data for all plots separately
inc_P <- count_tab[,grepl("P", colnames(count_tab))] 
inc_D <- count_tab[,grepl("D", colnames(count_tab))]
inc_V <- count_tab[,grepl("V", colnames(count_tab))]
inc_S <- count_tab[,grepl("S", colnames(count_tab))]

# create a dataframe with incidence data summed by rows within plots
inc_byspecies <- cbind(rowSums(inc_P), rowSums(inc_D), rowSums(inc_V),
                    rowSums(inc_S))

colnames(inc_byspecies) <- c("Pinus", "Diphasiastrum", "Vaccinium", "Soil")

inc_byspecies <- rbind(c(length(colnames(inc_P)),
                      length(colnames(inc_D)),
                      length(colnames(inc_V)),
                      length(colnames(inc_S))), inc_byspecies)

inc_byspecies<-as.data.frame(inc_byspecies)


# Get the diversity estimates
out <- iNEXT(inc_byspecies, q=c(0,1,2), datatype ="incidence_freq", se = TRUE)

# Plot the diversity estimates

# Sample-size-based rarefaction and extrapolation (R/E) curve (type=1). This curve plots diversity estimates with 95% confidence intervals (if se=TRUE) as a function of sample size up to double the reference sample size, by default, or a user‐specified endpoint:

ggiNEXT(out, type=1, facet.var="Order.q", se = TRUE, color.var="Assemblage")


```



### Sample completeness curve

```{r}
# Sample completeness curve (type=2) with confidence intervals (if se=TRUE): see Figs. 1b and 2b in Hsieh et al. (2016). This curve plots the sample coverage with respect to sample size for the same range described in (1).

ggiNEXT(out, type=2, facet.var="None", color.var="Assemblage", se = TRUE)

```

```{r}
#Coverage-based R/E curve (type=3): see Figs. 1c and 2c in Hsieh et al. (2016). This curve plots the diversity estimates with confidence intervals (if se=TRUE) as a function of sample coverage up to the maximum coverage obtained from the maximum size described in (1)

ggiNEXT(out, type=3, facet.var="Order.q", color.var="Assemblage", se =TRUE)
```

### Diversity indices (only Symbiotrophs)

```{r}

#Hill number = effective number of species.  Hill numbers determines the measures’ sensitivity to species relative abundances.  Hill numbers include the three most widely used species diversity measures as special cases: species richness (q = 0), Shannon diversity (q = 1, as the effective number of common species in the assemblage) and Simpson diversity (q = 2, effective number of dominant species in the assemblage)

# filter only ECM and Endophytes:

matching_rows <- tax_tab$ASV[tax_tab$Guild %in% c("Ectomycorrhizal", "Endophyte")]

# Subset the matrix based on the matching row names
filtered_vs_trans_count_tab <- vst_trans_count_tab[matching_rows, ]

Symbio_df <- as.data.frame(filtered_vs_trans_count_tab) 


#I used normalized counts of reads after removing of ASVs found in NTCs and technical replicates
count_tab <- Symbio_df[, unlist(lapply(Symbio_df, is.numeric))] 
row.names(count_tab) <- count_tab$ASV.ID
count_tab <- subset(count_tab, select = -grep("_", names(count_tab))) #delete technical replicates
count_tab <- count_tab > 0


inc_P <- count_tab[,grepl("P", colnames(count_tab))] # get ASV incidence data for all plots separately
inc_D <- count_tab[,grepl("D", colnames(count_tab))]
inc_V <- count_tab[,grepl("V", colnames(count_tab))]
inc_S <- count_tab[,grepl("S", colnames(count_tab))]

# create a dataframe with incidence data summed by rows within plots
inc_byspecies <- cbind(rowSums(inc_P), rowSums(inc_D), rowSums(inc_V),
                    rowSums(inc_S))

colnames(inc_byspecies) <- c("Pinus", "Diphasiastrum", "Vaccinium", "Soil")

inc_byspecies <- rbind(c(length(colnames(inc_P)),
                      length(colnames(inc_D)),
                      length(colnames(inc_V)),
                      length(colnames(inc_S))), inc_byspecies)

inc_byspecies<-as.data.frame(inc_byspecies)


# Get the diversity estimates
out <- iNEXT(inc_byspecies, q=c(0,1,2), datatype ="incidence_freq", se = TRUE)

# Plot the diversity estimates

# Sample-size-based rarefaction and extrapolation (R/E) curve (type=1). This curve plots diversity estimates with 95% confidence intervals (if se=TRUE) as a function of sample size up to double the reference sample size, by default, or a user‐specified endpoint:

ggiNEXT(out, type=1, facet.var="Order.q", se = TRUE, color.var="Assemblage")


```

## Trophic Mode (FunGuild database)

```{r}
#read info table with the samples without replicates
sample_info_no_rep <- read.table("Info_noR.txt", header=T, row.names=1, check.names=F)

tax_table<-tax_tab
row.names(tax_table) <- tax_table$ASV
tax_table <- tax_table[, -1]
tax_table<- tax_table[order(rownames(tax_table)),]

# create a data frame from normalized dataset
counts_norm_df<-as.data.frame(vst_trans_count_tab)
row_names <- rownames(counts_norm_df)
counts_norm_df <-as.data.frame(counts_norm_df)
rownames(counts_norm_df) <- row_names


# create microeco file
meco_fungi <- microtable$new(sample_table = sample_info_no_rep, otu_table = counts_norm_df, tax_table = tax_table)
 
t1t <- trans_abund$new(dataset = meco_fungi, taxrank = "Trophic_Mode", ntaxa = 15)

t1t$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)
```

### Trophic Mode boxplots

```{r}
scolor<-c("darkorange1","dodgerblue3","hotpink2", "purple4")
t1t$plot_box(group = "Species", xtext_angle = 30, color_values = scolor) + ylab("Relative abundance (%)")
```

## Statistical comparison of Saprotroph abundancy

```{r}
abund_t<-t1t$data_abund

shapiro.test(abund_t[abund_t$Taxonomy == "Saprotroph",]$Abundance) # data normally distributed
bartlett.test(Abundance ~ Species, abund_t[abund_t$Taxonomy == "Saprotroph",]) # variances are equal

TukeySapro <- TukeyHSD(aov(Abundance ~ Species, abund_t[abund_t$Taxonomy == "Saprotroph",]))
TukeySapro

```

## Statistical comparison of Symbiotroph abundancy

```{r}
abund_t<-t1t$data_abund

shapiro.test(abund_t[abund_t$Taxonomy == "Symbiotroph",]$Abundance) # data not normally distributed
bartlett.test(Abundance ~ Species, abund_t[abund_t$Taxonomy == "Symbiotroph",]) # variances are not equal

kruskal_effsize(abund_t[abund_t$Taxonomy == "Symbiotroph",], Abundance ~ Species )
res.kruskal <-kruskal_test(abund_t[abund_t$Taxonomy == "Symbiotroph",], Abundance ~ Species)
res.kruskal 

pwc<-dunn_test(abund_t[abund_t$Taxonomy == "Symbiotroph",], Abundance ~ Species, p.adjust.method = "bonferroni") 
pwc

```

### Guild (FunGuild database)

```{r}
#read info table with the samples without replicates
sample_info_no_rep <- read.table("Info_noR.txt", header=T, row.names=1, check.names=F)

tax_table<-tax_tab
row.names(tax_table) <- tax_table$ASV
tax_table <- tax_table[, -1]
tax_table<- tax_table[order(rownames(tax_table)),]

# create a data frame from normalized data
counts_norm_df<-as.data.frame(vst_trans_count_tab)
row_names <- rownames(counts_norm_df)
counts_norm_df <-as.data.frame(counts_norm_df)
rownames(counts_norm_df) <- row_names


# create microeco file
meco_fungi <- microtable$new(sample_table = sample_info_no_rep, otu_table = counts_norm_df, tax_table = tax_table)
 
t1 <- trans_abund$new(dataset = meco_fungi, taxrank = "Guild", ntaxa = 15)

t1$plot_bar(bar_full = FALSE, use_alluvium = TRUE,xtext_keep = TRUE, xtext_angle = 90,facet = "Species",xtext_size = 6, color_values = mycol)

```

### Guilds boxplots 

```{r}
t1$plot_box(group = "Species", xtext_angle = 30, color_values = scolor) + ylab("Relative abundance (%)") 
```


### Statistical comparison of Ectomycorrhizal ASVs abundancy

```{r}

abund<-t1$data_abund

shapiro.test(abund[abund$Taxonomy == "Ectomycorrhizal",]$Abundance) # data not normally distributed
bartlett.test(Abundance ~ Species, abund[abund$Taxonomy == "Ectomycorrhizal",]) # variances are not equal

kruskal_effsize(abund[abund$Taxonomy == "Ectomycorrhizal",], Abundance ~ Species )
res.kruskal <-kruskal_test(abund[abund$Taxonomy == "Ectomycorrhizal",], Abundance ~ Species)
res.kruskal 
pwc<-dunn_test(abund[abund$Taxonomy == "Ectomycorrhizal",], Abundance ~ Species, p.adjust.method = "bonferroni") 
pwc

```


## Statistical comparison of Saprotroph abundancy (-> no diff)
```{r}

shapiro.test(abund[abund$Taxonomy == "Undefined Saprotroph",]$Abundance) # data normally distributed
bartlett.test(Abundance ~ Species, abund[abund$Taxonomy == "Undefined Saprotroph",]) # variances are equal

TukeySapro <- TukeyHSD(aov(Abundance ~ Species, abund[abund$Taxonomy == "Undefined Saprotroph",], correction =))
TukeySapro

```

# Statistical comparison of Endophyte abundancy (-> no diff)
```{r}

shapiro.test(abund[abund$Taxonomy == "Endophyte",]$Abundance) # data normally distributed
bartlett.test(Abundance ~ Species, abund[abund$Taxonomy == "Endophyte",]) # variances are equal

TukeyEndoph <- TukeyHSD(aov(Abundance ~ Species, abund[abund$Taxonomy == "Endophyte",], correction ="bonferroni"))
TukeyEndoph 

```






## Comparison of the samples with different Pinus tree age and tree height


### Preparation of datasets
```{r}
#dataset only for Pinus:
tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

# Select only Pinus samples ans ASV column:
columns_to_keep <- c(TRUE, !grepl("[_]", names(df)[-1]))
df_all <- df[, columns_to_keep, drop = FALSE] 


sample_info_no_R <- sample_info_no_rep
sample_data <- sample_info_no_R
sample_data <- sample_info_no_R[sample_info_no_R$Species == "Pinus",]

# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df, tax_data, by = "ASV" ) 

All_table<-Tax_otu_data

#write.table(All_table, file = "ASV_count_DIPH.txt", append = FALSE, quote = FALSE, sep = " ",
           # eol = "\n", na = "NA", dec = ".", row.names = FALSE,
           # col.names = TRUE, qmethod = c("escape", "double"),
           # fileEncoding = "")

```

### _Pinus_ trees age distribution 
```{r}

sample_info_no_R <- read.table("Info_noR.txt", header=T,  check.names=F)
Age<-sort(sample_info_no_R$Age)

plot(Age)

sample_info_no_R$Age_group <- ifelse(sample_info_no_R$Age < 100, "young", 
                                     ifelse(sample_info_no_R$Age < 110, "average", "old"))
```

### Heattrees to compare taxa abundancy between different tree age groups (All samples types)
```{r warning=FALSE}

tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

sample_data <- sample_info_no_R

# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df, tax_data, by = "ASV" ) 
Tax_otu_data  <- subset(Tax_otu_data , select = -grep("_", names(Tax_otu_data ))) #delete technical replicates


#converting to taxmap format
library(metacoder)
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy", # The column in the input table
                      class_sep = ";") # What each taxon is separated by

#remove unclassified taxa:
obj <- filter_taxa(obj, taxon_names != "-")


#For the heat tree comparison of abundances
obj$data$tax_abund <- calc_taxon_abund(obj, "tax_data", cols = sample_data$Sample)
obj$data$tax_occ <- calc_n_samples(obj, "tax_abund", groups = sample_data$Age_group, cols = sample_data$Sample)
obj$data$diff_table <- compare_groups(obj, data= "tax_abund", 
                                      cols = sample_data$Sample, # What columns of sample data to use
                                      groups = sample_data$Age_group) # What category each sample is assigned to

#We then need to correct for multiple comparisons and set non-significant differences to zero:
obj <- mutate_obs(obj, "diff_table",
                  wilcox_p_value = p.adjust(wilcox_p_value, method = "fdr")) #false discovery rate
obj$data$diff_table$log2_median_ratio[obj$data$diff_table$wilcox_p_value > 0.05] <- 0


# To print heat tree for group comparison:
set.seed(50)
heattree<-heat_tree_matrix(obj, key_size = 0.65,
                 data = "diff_table",
                 node_size = n_obs, # the number of ASVs per taxon
                 node_label = taxon_names,
                 node_color = log2_median_ratio, 
                 node_color_range = diverging_palette(), 
                 node_color_trans = "linear", 
                 node_color_interval = c(-2, 2), # The range of `log2_median_ratio` 
                 edge_color_interval = c(-2, 2), # The range of `log2_median_ratio`
                 node_size_axis_label = "Number of ASVs",
                 node_color_axis_label = "Log2 ratio median proportions",
                 layout = "davidson-harel", # The primary layout algorithm
                 initial_layout = "reingold-tilford") # The layout algorithm that initializes node locations

range(obj$data$diff_table$wilcox_p_value, finite = TRUE) #check if there are significant differences -> no difference at all
```

### Comparison of old and young trees (only Pinus samples)
```{r warning=FALSE}
sample_info_no_R$Age_group <- ifelse(sample_info_no_R$Age < mean(sample_info_no_R$Age), "young", "old" )
sample_info_no_R$Age_group <- ifelse(sample_info_no_R$Age < 90, "young", "old" )

tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

# Select only Pinus samples:
columns_to_keep <- c(TRUE, !grepl("[DVS]", names(df)[-1]))  # Including the first column
df_P <- df[, columns_to_keep, drop = FALSE]  # Subset dataframe with selected columns


sample_data <- sample_info_no_R
sample_data <- sample_info_no_R[sample_info_no_R$Species == "Pinus",]


# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df_P, tax_data, by = "ASV" ) 
Tax_otu_data  <- subset(Tax_otu_data , select = -grep("_", names(Tax_otu_data ))) #delete technical replicates

#converting to taxmap format
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy", # The column in the input table
                      class_sep = ";") # What each taxon is separated by

#remove unclassified taxa:
obj <- filter_taxa(obj, taxon_names != "-")


#For the heat tree comparison of abundances
obj$data$tax_abund <- calc_taxon_abund(obj, "tax_data", cols = sample_data$Sample)
obj$data$tax_occ <- calc_n_samples(obj, "tax_abund", groups = sample_data$Age_group, cols = sample_data$Sample)
obj$data$diff_table <- compare_groups(obj, data= "tax_abund", 
                                      cols = sample_data$Sample, # What columns of sample data to use
                                      groups = sample_data$Age_group) # What category each sample is assigned to

#We then need to correct for multiple comparisons and set non-significant differences to zero:
obj <- mutate_obs(obj, "diff_table",
                  wilcox_p_value = p.adjust(wilcox_p_value, method = "fdr")) #false discovery rate



```


```{r warning=FALSE}
obj$data$diff_table$log2_median_ratio[obj$data$diff_table$wilcox_p_value > 0.05] <- 0

range(obj$data$diff_table$wilcox_p_value, finite = TRUE) #check if there are significant differences

# -> No significant differences
```


## Compare taxa abundancy between groups with different tree height 

### Tree height distribuion
```{r}

Height<-sort(sample_info_no_R$Height)

plot(Height)

```

### Comparison of high and small trees (all species samples together)
```{r warning=FALSE}
sample_info_no_R$Height_group <- ifelse(sample_info_no_R$Height < mean(sample_info_no_R$Height), "small", "big" )


tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

sample_data <- sample_info_no_R

# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df, tax_data, by = "ASV" ) 
Tax_otu_data  <- subset(Tax_otu_data , select = -grep("_", names(Tax_otu_data ))) #delete technical replicates


#converting to taxmap format
library(metacoder)
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy", # The column in the input table
                      class_sep = ";") # What each taxon is separated by

#remove unclassified taxa:
obj <- filter_taxa(obj, taxon_names != "-")


#For the heat tree comparison of abundances
obj$data$tax_abund <- calc_taxon_abund(obj, "tax_data", cols = sample_data$Sample)
obj$data$tax_occ <- calc_n_samples(obj, "tax_abund", groups = sample_data$Height_group, cols = sample_data$Sample)
obj$data$diff_table <- compare_groups(obj, data= "tax_abund", 
                                      cols = sample_data$Sample, # What columns of sample data to use
                                      groups = sample_data$Height_group) # What category each sample is assigned to

```


```{r warning=FALSE}
#We then need to correct for multiple comparisons and set non-significant differences to zero:
obj <- mutate_obs(obj, "diff_table",
                  wilcox_p_value = p.adjust(wilcox_p_value, method = "fdr")) #false discovery rate
obj$data$diff_table$log2_median_ratio[obj$data$diff_table$wilcox_p_value > 0.05] <- 0

range(obj$data$diff_table$wilcox_p_value, finite = TRUE) #check if there are significant differences 

#-> No significant differences
```


### Comparison of samples with high and small trees (only Pinus samples)
```{r warning=FALSE}
sample_info_no_R$Height_group <- ifelse(sample_info_no_R$Height < mean(sample_info_no_R$Height), "small", "big" )


tax_data <-tax_tab
tax_data <- tax_data[tax_data$ASV %in% rownames(filt_count_tab), ] # delete the blank ASVs

counts_norm_df2<-counts_norm_df
rownames(counts_norm_df2) <- NULL
df <- cbind(ASV= row_names, counts_norm_df2) # Add row names as the first column

# Select only Pinus samples ans ASV column:
columns_to_keep <- c(TRUE, !grepl("[DVS_]", names(df)[-1]))
df_P <- df[, columns_to_keep, drop = FALSE] 

sample_data <- sample_info_no_R
sample_data <- sample_info_no_R[sample_info_no_R$Species == "Pinus",]


# Combine the tables sorting by ASV
Tax_otu_data <- left_join(df_P, tax_data, by = "ASV" ) 
Tax_otu_data  <- subset(Tax_otu_data , select = -grep("_", names(Tax_otu_data ))) #delete technical replicates

#converting to taxmap format
library(metacoder)
obj <- parse_tax_data(Tax_otu_data,
                      class_cols = "taxonomy", # The column in the input table
                      class_sep = ";") # What each taxon is separated by

#remove unclassified taxa:
obj <- filter_taxa(obj, taxon_names != "-")


#For the heat tree comparison of abundances
obj$data$tax_abund <- calc_taxon_abund(obj, "tax_data", cols = sample_data$Sample)
obj$data$tax_occ <- calc_n_samples(obj, "tax_abund", groups = sample_data$Height_group, cols = sample_data$Sample)
obj$data$diff_table <- compare_groups(obj, data= "tax_abund", 
                                      cols = sample_data$Sample, # What columns of sample data to use
                                      groups = sample_data$Height_group) # What category each sample is assigned to

```

```{r warning=FALSE}
#We then need to correct for multiple comparisons and set non-significant differences to zero:
obj <- mutate_obs(obj, "diff_table",
                  wilcox_p_value = p.adjust(wilcox_p_value, method = "fdr")) #false discovery rate
obj$data$diff_table$log2_median_ratio[obj$data$diff_table$wilcox_p_value > 0.05] <- 0

range(obj$data$diff_table$wilcox_p_value, finite = TRUE) #check if there are significant differences 

#-> No significant differences
```



