---
title: "Clustering with BLAST Results"  
teaching: 30
exercises: 15
questions:
- "How can we use the blast results to form families?"
objectives:
- "Use a clustering algorithm to form families using the E-value."
keypoints:
- "The Bidirectional Best-Hit algorithm groups sequences together into families according to the E-value."
---

## Using E-values to cluster sequences into families

In the previous episode, we obtained the E-value between each pair of sequences of our mini dataset. Even though it is not strictly an identity measure between the sequences, 
the E-value allows us to know from a pool of sequences which one is the best match to a query sequence. We will now use this information to cluster the sequences from 
families using an algorithm written in Python, and we will see how it 
joins the sequences progressively until each of them is part of a family.

## Processing the BLAST results

For this section, we will use Python. Let's open the notebook and start by importing the libraries that we will need.
~~~
import os 
import pandas as pd
from matplotlib import cm
import numpy as np
~~~
{: .language-python}
First, we need to read the `mini-genomes.blast` file that we produced.
Let's import the BLAST results to Python using the column names: `qseqid`,`sseqid`, `evalue`.

~~~
os.getcwd()
blastE = pd.read_csv( '~/pan_workshop/results/blast/mini/output_blast/mini-genomes.blast', sep = '\t',names = ['qseqid','sseqid','evalue'])
blastE.head()
~~~
{: .language-python}

~~~
  qseqid	               sseqid	               evalue
0	2603V|GBPINHCM_01420	NEM316|AOGPFIKH_01528	4.110000e-67
1	2603V|GBPINHCM_01420	A909|MGIDGNCP_01408	4.110000e-67
2	2603V|GBPINHCM_01420	515|LHMFJANI_01310	4.110000e-67
3	2603V|GBPINHCM_01420	2603V|GBPINHCM_01420	4.110000e-67
4	2603V|GBPINHCM_01420	A909|MGIDGNCP_01082	1.600000e+00
~~~
{: .output}

Now we want to make two columns that have the name of the genomes of the queries, and the name of the genomes of the subjects. We will take this information from the query 
and subject IDs (the label that we added at the beginning of the episode).

First, let's obtain the genome of each query gene.
~~~
qseqid = pd.DataFrame(blastE,columns=['qseqid'])

newqseqid = qseqid["qseqid"].str.split("|", n = 1, expand = True)
newqseqid.columns= ["Genome1", "Gen"]
newqseqid["qseqid"]= qseqid
dfqseqid =newqseqid[['Genome1','qseqid']]

dfqseqid.head()
~~~
{: .language-python}

~~~
  Genome1	qseqid
0	2603V	2603V|GBPINHCM_01420
1	2603V	2603V|GBPINHCM_01420
2	2603V	2603V|GBPINHCM_01420
3	2603V	2603V|GBPINHCM_01420
4	2603V	2603V|GBPINHCM_01420
~~~
{: .output}

Now let's repeat the same for the `sseqid` column.
~~~
sseqid = pd.DataFrame(blastE,columns=['sseqid'])

newsseqid = sseqid["sseqid"].str.split("|", n = 1, expand = True)
newsseqid.columns= ["Genome2", "Gen"]
newsseqid["sseqid"]= sseqid 
dfsseqid = newsseqid[['Genome2','sseqid']]
~~~
{: .language-python}

Now that we have two dataframes with the new columns that we wanted, let's combine them with the `evalue` of the `blastE` dataframe into a new one called `df`.

~~~
evalue = pd.DataFrame(blastE, columns=['evalue'])
df = dfqseqid
df['Genome2']=dfsseqid['Genome2']
df['sseqid']=sseqid
df['evalue']=evalue
df.head()
~~~
{: .language-python}

~~~
  Genome1	qseqid	Genome2	sseqid	evalue
0	2603V	2603V|GBPINHCM_01420	NEM316	NEM316|AOGPFIKH_01528	4.110000e-67
1	2603V	2603V|GBPINHCM_01420	A909	A909|MGIDGNCP_01408	4.110000e-67
2	2603V	2603V|GBPINHCM_01420	515	515|LHMFJANI_01310	4.110000e-67
3	2603V	2603V|GBPINHCM_01420	2603V	2603V|GBPINHCM_01420	4.110000e-67
4	2603V	2603V|GBPINHCM_01420	A909	A909|MGIDGNCP_01082	1.600000e+00
~~~
{: .output}

Now we want a list of the unique genes in our dataset.
~~~
qseqid_unique=pd.unique(df['qseqid'])
sseqid_unique=pd.unique(df['sseqid'])
genes = pd.unique(np.append(qseqid_unique, sseqid_unique))
~~~
{: .language-python}

We can check that we have 43 genes in total with `len(genes)`.

Now, we want to know which one is the biggest genome (the one with more genes) to make the comparisons.  

First, we compute the unique genomes.

~~~
genomes=pd.unique(df['Genome1'])
genomes=list(genomes)
genomes
~~~
{: .language-python}
~~~
['2603V', '515', 'A909', 'NEM316']
~~~
{: .output}

Now, we will create a dictionary that shows which genes are in each genome.

~~~
dic_gen_genomes={}
for a in genomes:
    temp=[]
    for i in range(len(genes)):
        if a in genes[i]:
            gen=genes[i]
            temp.append(gen)
    dic_gen_genomes[a]=temp
~~~
{: .language-python}

We can now use this dictionary to know how many genes each genome has and therefore identify the biggest genome.
~~~
genome_temp=[]
size_genome=[]
for i in dic_gen_genomes.keys():
    size=len(dic_gen_genomes[i])
    genome_temp.append(i)
    size_genome.append(size)

genomes_sizes = pd.DataFrame(genome_temp, columns=['Genome'])
genomes_sizes['Size']=size_genome

genome_sizes_df = genomes_sizes.sort_values('Size', ascending=False)
genome_sizes_df
~~~
{: .language-python}

~~~
Genome	Size
2	A909	12
0	2603V	11
1	515	10
3	NEM316	10
~~~
{: .output}

Now we can sort our genomes by their size.
~~~
genomes=genome_sizes_df['Genome'].tolist()
genomes
~~~
{: .language-python}

~~~
['A909', '2603V', '515', 'NEM316']
~~~
{: .output}

So the biggest genome is `A909` and we will start our clustering algorithm with it.

## Finding gene families with the BBH algorithm

To make a gene family, we first need to identify the most similar genes between genomes. 
The Bidirectional best-hit algorithm will allow us to find the pairs of genes that are the most similar 
(lowest e-value) to each other in each pair of genomes.  

<a href="{{ page.root }}/fig/bdbh.png">
   <img src="{{ page.root }}/fig/bdbh.png" alt=" Bidirectional best-hit algorithm" />
  </a>

For this, we will define a function to find in each genome the gene that is most similar to each
gene in our biggest genome A909. 

> ## Clustering algorithms
> Can the BBH algorithm make gene families that have more than one gene from the same genome?
> > ## Solution
> > No. Since BBH finds the best hit of a query in each of the other genomes it will only give one hit per genome. This will force to
> > have a different gene family for each duplicate of a gene (paralog).
> > This also means that you are getting the **best** hit, which is not necessarily a **"good"** hit. To define what a good hit is we would
> > need to use an algorithm that considers a similarity threshold.
> > 
> {: .solution}
{: .discussion}

~~~
def besthit(gen,genome,data):
    # gen: a fixed gen in the list of unique genes
    # genome: the genome in which we will look the best hit
    # df: the data frame with the evalues
    filter_cd=(data['qseqid']==gen) & (data['Genome2']==genome) & (data['Genome1']!=genome)
    if (len(data[filter_cd]) == 0 ):
        gen_besthit = "NA"
    else:
        gen_besthit = data.loc[filter_cd,'sseqid'].at[data.loc[filter_cd,'evalue'].idxmin()]
   
    return(gen_besthit)
~~~
{: .language-python}

Now we will define a second function, that uses the previous one, to obtain the bidirectional
best hits.  

~~~
def besthit_bbh(gengenome,listgenomes,genome,data):
    # gengenome: a list with all the genes of the biggest genome.
    # listgenomes: the list with all the genomes in order.
    # genome: the genome to which the genes in `gengenome` belongs.
    # data: the data frame with the evalues.
    
    dic_besthits = {}
    for a in gengenome:
        temp=[]
        for b in listgenomes:
            temp2=besthit(a,b,data)
            temp3=besthit(temp2,genome,data)
            if temp3 == a:
                temp.append(temp2)
            else:
                temp.append('NA')
        dic_besthits[a]=temp
        
    return(dic_besthits)
~~~
{: .language-python}

In one of the previous steps, we created a dictionary with all the genes present in each genome.
Since we know that the biggest genome is `A909`, we will obtain the genes belonging to `A909` and 
gather them in a list.  

~~~
genome_A909 = dic_gen_genomes['A909']
~~~
{: .language-python}

Now, we will apply the function `besthit_bbh` to the previous list, `genomes`, and  the genome `A909` that is `genomes[0]`.

~~~
g_A909_bbh=besthit_bbh(genome_A909,genomes,genomes[0],df)
~~~
{: .language-python}

In `g_A909_bbh` we have a dictionary that has one gene family for each gene in A909. Let's convert it to a dataframe and have a better look at it.

~~~
family_A909=pd.DataFrame(g_A909_bbh).transpose()
family_A909.columns = ['g_A909','g_2603V','g_515','g_NEM316']
family_A909.g_A909 = family_A909.index
family_A909
~~~
{: .language-python}

~~~
	                g_A909	                g_2603V	                g_515	                g_NEM316
A909|MGIDGNCP_01408	A909|MGIDGNCP_01408	2603V|GBPINHCM_01420	515|LHMFJANI_01310	NEM316|AOGPFIKH_01528
A909|MGIDGNCP_00096	A909|MGIDGNCP_00096	2603V|GBPINHCM_00097	515|LHMFJANI_00097	NEM316|AOGPFIKH_00098
A909|MGIDGNCP_01343	A909|MGIDGNCP_01343	NA                  	NA	                NEM316|AOGPFIKH_01415
A909|MGIDGNCP_01221	A909|MGIDGNCP_01221	NA	                515|LHMFJANI_01130	NA
A909|MGIDGNCP_01268	A909|MGIDGNCP_01268	2603V|GBPINHCM_01231	515|LHMFJANI_01178	NEM316|AOGPFIKH_01341
A909|MGIDGNCP_00580	A909|MGIDGNCP_00580	2603V|GBPINHCM_00554	515|LHMFJANI_00548	NEM316|AOGPFIKH_00621
A909|MGIDGNCP_00352	A909|MGIDGNCP_00352	2603V|GBPINHCM_00348	515|LHMFJANI_00342	NEM316|AOGPFIKH_00350
A909|MGIDGNCP_00064	A909|MGIDGNCP_00064	2603V|GBPINHCM_00065	515|LHMFJANI_00064	NEM316|AOGPFIKH_00065
A909|MGIDGNCP_00627	A909|MGIDGNCP_00627	NA	                NA	                NA
A909|MGIDGNCP_01082	A909|MGIDGNCP_01082	2603V|GBPINHCM_01042	NA	                NA
A909|MGIDGNCP_00877	A909|MGIDGNCP_00877	2603V|GBPINHCM_00815	515|LHMFJANI_00781	NEM316|AOGPFIKH_00855
A909|MGIDGNCP_00405	A909|MGIDGNCP_00405	2603V|GBPINHCM_00401	515|LHMFJANI_00394	NEM316|AOGPFIKH_00403
~~~
{: .output}

Here, we have all the families that contain one gene from the biggest genome. The following step is to repeat
this for the second-biggest genome. To do this, we need to remove from the list `genes` the genes that are already 
placed in the current families.

~~~
list_g=[]
for elemt in g_A909_bbh.keys():
    list_g.append(elemt)
    for g_hit in g_A909_bbh[elemt]:
        list_g.append(g_hit)
~~~
{: .language-python}

~~~
genes2=genes
genes2=genes2.tolist()
genesremove=pd.unique(list_g).tolist()
genesremove.remove('NA')
for b_hits in genesremove:
    genes2.remove(b_hits)

genes2
~~~
{: .language-python}

~~~
['2603V|GBPINHCM_00748', '2603V|GBPINHCM_01226', '515|LHMFJANI_01625', 'NEM316|AOGPFIKH_01842']
~~~
{: .output}

For this 4 genes we will repeat the algorithm. First, we create the list with the genes that belongs to the second biggest genome `2603V`.

~~~
genome_2603V=[]
for i in range(len(genes2)):
    if "2603V" in genes2[i]:
        gen = genes2[i]
        genome_2603V.append(gen)
        
genome_2603V
~~~
{: .language-python}
~~~
['2603V|GBPINHCM_00748', '2603V|GBPINHCM_01226']
~~~
{: .output}

We apply the function `besthit_bbh` to this list.

~~~
g_2603V_bbh=besthit_bbh(genome_2603V,genomes,genomes[1],df)
~~~
{: .language-python}

We convert the dictionary into a dataframe.

~~~
family_2603V=pd.DataFrame(g_2603V_bbh).transpose()
family_2603V.columns = ['g_A909','g_2603V','g_515','g_NEM316']
family_2603V.g_2603V = family_2603V.index
family_2603V.head()
~~~
{: .language-python}

~~~
                        g_A909	g_2603V	               g_515	  g_NEM316
2603V|GBPINHCM_00748	NA	2603V|GBPINHCM_00748	NA	   NA
2603V|GBPINHCM_01226	NA	2603V|GBPINHCM_01226	NA	   NA
~~~
{: .output}

Again, let's eliminate the genes that are already placed in families to repeat the algorithm.

~~~
for a in genome_2603V:
    genes2.remove(a)

genes2
~~~
{: .language-python}

~~~
['515|LHMFJANI_01625', 'NEM316|AOGPFIKH_01842']
~~~
{: .output}

~~~
genome_515=[]
for i in range(len(genes2)):
    if "515" in genes2[i]:
        gen = genes2[i]
        genome_515.append(gen)
        
genome_515
~~~
{: .language-python}
~~~
['515|LHMFJANI_01625']
~~~
{: .output}
~~~
g_515_bbh=besthit_bbh(genome_515,genomes,genomes[2],df)
~~~
{: .language-python}

~~~
family_515=pd.DataFrame(g_515_bbh).transpose()
family_515.columns = ['g_A909','g_2603V','g_515','g_NEM316']
family_515.g_515 = family_515.index
family_515
~~~
{: .language-python}

~~~
                    g_A909  g_2603V g_515               g_NEM316
515|LHMFJANI_01625  NA      NA      515|LHMFJANI_01625  NEM316|AOGPFIKH_01842
~~~
{: .output}

Since in this last step we used all the genes, we have finished our algorithm.  

Now we will only create a final dataframe to integrate all of the obtained families.

~~~
families_bbh=pd.concat([family_A909,family_2603V,family_515])
families_bbh.to_csv('families_bbh.csv')
families_bbh
~~~
{: .language-python}

~~~
	                 g_A909	           g_2603V	             g_515	             g_NEM316
A909|MGIDGNCP_01408	A909|MGIDGNCP_01408	2603V|GBPINHCM_01420	515|LHMFJANI_01310	NEM316|AOGPFIKH_01528
A909|MGIDGNCP_00096	A909|MGIDGNCP_00096	2603V|GBPINHCM_00097	515|LHMFJANI_00097	NEM316|AOGPFIKH_00098
A909|MGIDGNCP_01343	A909|MGIDGNCP_01343	NA	                NA	                  NEM316|AOGPFIKH_01415
A909|MGIDGNCP_01221	A909|MGIDGNCP_01221	NA	                515|LHMFJANI_01130	  NA
A909|MGIDGNCP_01268	A909|MGIDGNCP_01268	2603V|GBPINHCM_01231	515|LHMFJANI_01178	NEM316|AOGPFIKH_01341
A909|MGIDGNCP_00580	A909|MGIDGNCP_00580	2603V|GBPINHCM_00554	515|LHMFJANI_00548	NEM316|AOGPFIKH_00621
A909|MGIDGNCP_00352	A909|MGIDGNCP_00352	2603V|GBPINHCM_00348	515|LHMFJANI_00342	NEM316|AOGPFIKH_00350
A909|MGIDGNCP_00064	A909|MGIDGNCP_00064	2603V|GBPINHCM_00065	515|LHMFJANI_00064	NEM316|AOGPFIKH_00065
A909|MGIDGNCP_00627	A909|MGIDGNCP_00627	NA	                NA	                    NA
A909|MGIDGNCP_01082	A909|MGIDGNCP_01082	2603V|GBPINHCM_01042	NA	                NA
A909|MGIDGNCP_00877	A909|MGIDGNCP_00877	2603V|GBPINHCM_00815	515|LHMFJANI_00781	NEM316|AOGPFIKH_00855
A909|MGIDGNCP_00405	A909|MGIDGNCP_00405	2603V|GBPINHCM_00401	515|LHMFJANI_00394	NEM316|AOGPFIKH_00403
2603V|GBPINHCM_00748	NA	             2603V|GBPINHCM_00748	  NA	                NA
2603V|GBPINHCM_01226	NA	             2603V|GBPINHCM_01226	  NA	                NA
515|LHMFJANI_01625	NA	                NA	                 515|LHMFJANI_01625	  NEM316|AOGPFIKH_01842
~~~
{: .output}
Here we have our complete pangenome! In the first column, we have the gene family names, and then one column per genome 
with the genes that belong to each family.  

Finally, we will export to a `csv` file.

~~~
families_bbh.to_csv('~/pan_workshop/results/blast/mini/families_minis.csv')
~~~
{: .language-python}

## Explore functional annotation of gene families

Now that we have our genes grouped together in gene families and since we have the functional annotation of each gene, we can check if the
obtained families coincide with the functional annotations. For this, we will go back to our Terminal to use Bash.

The unique functional annotation that our mini genomes have are the following.

~~~
$ cat mini-genomes.faa | grep '>' | cut -d' ' -f2- | sort | uniq
~~~
{: .language-bash}

~~~
30S ribosomal protein S16
50S ribosomal protein L16
bifunctional DNA primase/polymerase
Glutamate 5-kinase 1
Glycine betaine transporter OpuD
glycosyltransferase
Glycosyltransferase GlyG
peptidase U32 family protein
Periplasmic murein peptide-binding protein
PII-type proteinase
Putative N-acetylmannosamine-6-phosphate 2-epimerase
Replication protein RepB
Ribosome hibernation promotion factor
UDP-N-acetylglucosamine--N-acetylmuramyl-(pentapeptide) pyrophosphoryl-undecaprenol N-acetylglucosamine transferase
Vitamin B12 import ATP-binding protein BtuD
~~~
{: .output}

Let's use these functional annotation names to obtain the gene names in the `mini-genomes.faa` file and the gene family names
from the `families_minis.csv` table. With all the information together let's create a new table that describes our pangenome.

~~~
$ echo Function$'\t'Gene$'\t'Family > mini_pangenome.tsv
$ cat mini-genomes.faa | grep '>' | cut -d' ' -f2- | sort | uniq | while read function
do 
grep "$function" mini-genomes.faa | cut -d' ' -f1 | cut -d'>' -f2 | while read line
do 
family=$(grep $line families_minis.csv| cut -d',' -f1)
echo $function$'\t'$line$'\t'$family
done
done >> mini_pangenome.tsv
$ head mini_pangenome.tsv
~~~
{: .language-bash}

~~~
Function                  Gene                  Family
30S ribosomal protein S16 2603V|GBPINHCM_01420  A909|MGIDGNCP_01408
30S ribosomal protein S16 515|LHMFJANI_01310    A909|MGIDGNCP_01408
30S ribosomal protein S16 A909|MGIDGNCP_01408   A909|MGIDGNCP_01408
30S ribosomal protein S16 NEM316|AOGPFIKH_01528 A909|MGIDGNCP_01408
50S ribosomal protein L16 2603V|GBPINHCM_00097  A909|MGIDGNCP_00096
50S ribosomal protein L16 515|LHMFJANI_00097    A909|MGIDGNCP_00096
50S ribosomal protein L16 A909|MGIDGNCP_00096   A909|MGIDGNCP_00096
50S ribosomal protein L16 NEM316|AOGPFIKH_00098 A909|MGIDGNCP_00096
~~~
{: .output}

> ## Exercise 1: Partitioning the pangenome
> Since we have a very small pangenome we can know the partitions of our pangenome just by looking at a small table.
> Look at the `mini_pangenom.tsv` table and decide which families correspond to the **Core**, **Shell** and **Cloud** genomes.
> 
> Note: You might want to download the file to your computer and open it in a spreadsheet program to read it easily.
> > ## Solution
> > 
> > |Functional annotation of family | No. Genomes | Partition |
> > |---|---|---|
> > |30S ribosomal protein S16|4|Core|
> > |50S ribosomal protein L16|4|Core|
> > |Glutamate 5-kinase 1|4|Core|
> > |glycosyltransferase|4|Core|
> > |peptidase U32 family protein|4|Core|
> > |Putative N-acetylmannosamine-6-phosphate 2-epimerase|4|Core|
> > |Ribosome hibernation promotion factor|4|Core|
> > |UDP-N-acetylglucosamine--N-acetylmuramyl-(pentapeptide) pyrophosphoryl-undecaprenol N-acetylglucosamine transferase|4|Core|
> > |Glycine betaine transporter OpuD|2|Shell|
> > |Periplasmic murein peptide-binding protein|2|Shell|
> > |Replication protein RepB|2|Shell|
> > |Vitamin B12 import ATP-binding protein BtuD|2|Shell|
> > |bifunctional DNA primase/polymerase|1|Cloud|
> > |Glycosyltransferase GlyG|1|Cloud|
> > |PII-type proteinase|1|Cloud|
> > 
> {: .solution}
{: .challenge}
