# Replication of the Higgs boson decay H → ZZ → 4ℓ (CERN Open Data)

This repository accompanies my master's thesis **“Replikacija analize podataka za Higsov bozon: raspad u četiri leptona u ROOT Framework-u”**.  
It presents a multi-level replication of the CMS Higgs → ZZ → 4 leptons (H → ZZ → 4ℓ) analysis using **publicly available data** from the [CERN Open Data portal](http://opendata.cern.ch).

---

## About the analysis

The Higgs boson was discovered in 2012 by the ATLAS and CMS collaborations at the LHC.  
One of the key decay channels is **H → ZZ → 4ℓ** (where ℓ = e, μ).  

This project reproduces the main elements of that analysis using open data:

- **Level 1–3**: Simplified replications based on the [CMS Open Data Higgs to four leptons example using 2011-2012 data](https://opendata.cern.ch/record/5500)  
- **Level 4**: Also present in the example repository, but my contribution extends it by **parallelizing the processing** across multiple **CernVM instances** on **Google Cloud Platform (GCP)**, enabling more efficient large-scale analysis.

The replication is inspired by the published CMS analysis:  
> CMS Collaboration, *Observation of a new boson at a mass of 125 GeV with the CMS experiment at the LHC*,  
> Phys. Lett. B716 (2012) 30–61, [arXiv:1207.7235](https://arxiv.org/abs/1207.7235).

---

## Repository structure

- `docs/` – Thesis PDF and slides (in Serbian)
- `level_1/` – Instructions for Level 1  
- `level_2/` – Instructions for Level 2  
- `level_3/` – Instructions for Level 3  
- `level_4/` – Instructions for Level 4  

---

## Data sources

All data are obtained from the [CERN Open Data portal](http://opendata.cern.ch).  
Raw data are **not** stored in this repository. Scripts and dataset lists are provided to fetch the required files.

---

## Credits

- Levels 1–3 and the base structure of Level 4 are adapted from the official CMS Open Data example:  
  [Higgs-to-four-lepton analysis example using 2011-2012 data](https://opendata.cern.ch/record/5500).  
- My contribution is the **parallelization of Level 4** by distributing jobs across multiple VMs on Google Cloud. 

