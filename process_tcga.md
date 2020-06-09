---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.2'
      jupytext_version: 1.4.2
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

```python
import pandas as pd
import numpy as np

from collections import Counter

from tqdm import tqdm_notebook as tqdm

from multiprocessing import Pool
```

# Sample information

```python
tcga_sample_info = pd.read_csv("raw/TCGA/TCGA_phenotype_denseDataOnlyDownload.tsv.gz",
                               sep="\t",
                               index_col=0
                              )

tcga_map = {
    'acute myeloid leukemia': "LAML",
    'adrenocortical cancer': "ACC",
    'bladder urothelial carcinoma': "BLCA",
    'brain lower grade glioma': "LGG",
    'breast invasive carcinoma': "BRCA",
    'cervical & endocervical cancer': "CESC",
    'cholangiocarcinoma': "CHOL",
    'colon adenocarcinoma': "COAD",
    'diffuse large B-cell lymphoma': "DLBC",
    'esophageal carcinoma': "ESCA",
    'glioblastoma multiforme': "GBM",
    'head & neck squamous cell carcinoma': "HNSC",
    'kidney chromophobe': "KICH",
    'kidney clear cell carcinoma': "KIRC",
    'kidney papillary cell carcinoma': "KIRP",
    'liver hepatocellular carcinoma': "LIHC",
    'lung adenocarcinoma': "LUAD",
    'lung squamous cell carcinoma': "LUSC",
    'mesothelioma': "MESO",
    'ovarian serous cystadenocarcinoma': "OV",
    'pancreatic adenocarcinoma': "PAAD",
    'pheochromocytoma & paraganglioma': "PCPG",
    'prostate adenocarcinoma': "PRAD",
    'rectum adenocarcinoma': "READ",
    'sarcoma': "SARC",
    'skin cutaneous melanoma': "SKCM",
    'stomach adenocarcinoma': "STAD",
    'testicular germ cell tumor': "TGCT",
    'thymoma': "THYM",
    'thyroid carcinoma': "THCA",
    'uterine carcinosarcoma': "UCS",
    'uterine corpus endometrioid carcinoma': "UCEC",
    'uveal melanoma': "UVM"
}

tcga_sample_info["abbreviated_disease"] = tcga_sample_info["_primary_disease"].apply(
    lambda x: tcga_map[x])


tcga_sample_info.to_hdf("processed/TCGA/tcga_sample_info.hdf",key="tcga_sample_info",mode="w")
```

# Mutations

```python
tcga_muts = pd.read_csv("raw/TCGA/mc3.v0.2.8.PUBLIC.xena.gz",
                        sep="\t",
                        index_col=0
                        )

tcga_muts.to_hdf("processed/tcga/tcga_muts.hdf", key="tcga_muts", mode="w")
```

# Copy number


## Continuous

```python
tcga_cn = pd.read_csv("raw/TCGA/Gistic2_CopyNumber_Gistic2_all_data_by_genes.gz",
                      sep="\t",
                      index_col=0
                      )

tcga_cn = tcga_cn.T
tcga_cn = tcga_cn.astype(np.float32)

tcga_cn.to_hdf(
    "processed/TCGA/tcga_cn.hdf", key="tcga_cn", mode="w")
```

## Thresholded

```python
tcga_cn_thresholded = pd.read_csv("raw/TCGA/Gistic2_CopyNumber_Gistic2_all_thresholded.by_genes.gz",
                                  sep="\t",
                                  index_col=0)

tcga_cn_thresholded = tcga_cn_thresholded.T
tcga_cn_thresholded = tcga_cn_thresholded.astype(np.float16)
tcga_cn_thresholded.to_hdf(
    "processed/TCGA/tcga_cn_thresholded.hdf", key="tcga_cn_thresholded", mode="w")
```

# Batch effects normalized RNAseq

```python
tcga_genex = pd.read_csv("raw/TCGA/EB++AdjustPANCAN_IlluminaHiSeq_RNASeqV2.geneExp.xena.gz",sep="\t",index_col=0).T
tcga_genex = tcga_genex.astype(np.float32)
tcga_genex.columns = [tcga_genex.columns[i] + "_" + str(i) for i in range(len(tcga_genex.columns))]

tcga_genex.to_hdf("processed/TCGA/TCGA_genex_norm.h5",key="tcga_genex",mode="w")
```

# Splicing (Kahles et al.)


## Load events

```python
from functools import reduce

def concat_cols(df, cols, delim):
    cols_str = [df[x].astype(str) for x in cols]

    return reduce(lambda a, b: a + delim + b, cols_str)
```

```python
def process_splicing(load_path, save_path):

    chunk_iterator = pd.read_csv(load_path, sep="\t", chunksize=1000)

    chunk_count = 0

    merged = []
    
    pbar = tqdm()

    for chunk in chunk_iterator:

        chunk["exon_id"] = concat_cols(chunk,
                                       [
                                           "gene_name",
                                           "event_type",
                                           "event_chr",
                                           "event_coordinates",
                                           "alt_region_coordinates"
                                       ],
                                       "_"
                                       )

        chunk = chunk.drop(["event_id",
                            "event_type",
                            "event_chr",
                            "event_coordinates",
                            "alt_region_coordinates",
                            "gene_name"
                            ], axis=1)

        chunk = chunk.set_index("exon_id")
        chunk = chunk.dropna(axis=0, thresh=len(chunk.columns)/10)

        chunk = chunk[chunk.std(axis=1) > 0.025]
        chunk = chunk.astype(np.float16)

        merged.append(chunk)

        chunk_count += 1
        
        pbar.update(1)
        
    merged = pd.concat(merged,axis=0)
    merged = merged.astype(np.float16)
    merged = merged.T
    
    # remove prefix identifiers from names
    merged.index = merged.index.map(lambda x: x.split(".")[0])
    
    merged.to_hdf(save_path,key="tcga_splicing",mode="w")
```

```python
path_pairs = [
    ["raw/TCGA/merge_graphs_alt_3prime_C2.confirmed.txt.gz","processed/TCGA/splicing_a3ss.h5"],
    ["raw/TCGA/merge_graphs_alt_5prime_C2.confirmed.txt.gz","processed/TCGA/splicing_a5ss.h5"],
    ["raw/TCGA/merge_graphs_intron_retention_C2.confirmed.txt.gz","processed/TCGA/splicing_ri.h5"],
    ["raw/TCGA/merge_graphs_exon_skip_C2.confirmed.txt.gz","processed/TCGA/splicing_se.h5"]
]
```

```python
with Pool(processes=4) as pool:
    pool.starmap(process_splicing, path_pairs)
```

```python
tcga_a3ss = pd.read_hdf("processed/TCGA/splicing_a3ss.h5",key="tcga_splicing")
tcga_a5ss = pd.read_hdf("processed/TCGA/splicing_a5ss.h5",key="tcga_splicing")
tcga_ri = pd.read_hdf("processed/TCGA/splicing_ri.h5",key="tcga_splicing")
tcga_se = pd.read_hdf("processed/TCGA/splicing_se.h5",key="tcga_splicing")
```

## Merge events

```python
t2g = pd.read_csv("../MDM4-splicing/data/intermediate/sleuth_diff/ensembl_t2g.csv")
t2g["format_gene_id"] = t2g["hgnc_gene"].fillna("") + "_" + t2g["ens_gene"]

format_gene_map = dict(zip(t2g["ens_gene"],t2g["format_gene_id"]))
```

```python
tcga_splicing = pd.concat([
    tcga_a3ss,
    tcga_a5ss,
    tcga_ri,
    tcga_se
],axis=1,join="outer")

tcga_splicing.columns = [format_gene_map.get(x.split(".")[0],"unnamed")+"_"+x for x in tcga_splicing.columns]
```

```python
tcga_splicing.to_hdf("processed/TCGA/merged.h5",key="tcga_splicing",mode="w")
```