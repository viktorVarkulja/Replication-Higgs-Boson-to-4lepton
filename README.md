# Replication of the Higgs boson decay H → ZZ → 4ℓ (CERN Open Data)

This repository accompanies my master's thesis **“Replikacija analize podataka za Higsov bozon: raspad u četiri leptona u ROOT Framework-u”**.  
It presents a multi-level replication of the CMS Higgs → ZZ → 4 leptons (H → ZZ → 4ℓ) analysis using **publicly available data** from the [CERN Open Data portal](http://opendata.cern.ch).

---

## About the analysis

The Higgs boson was discovered in 2012 by the ATLAS and CMS collaborations at the LHC.  
One of the key decay channels is **H → ZZ → 4ℓ** (where ℓ = e, μ).  

This project reproduces the main elements of that analysis using open data:

- **Level 1–3**: Simplified replications based on the [CMS Open Data Higgs to four leptons example repository](https://github.com/cms-opendata-analyses/HiggsToFourLeptons2011)  
- **Level 4**: My own contribution — scaling the analysis to larger datasets using **CernVM** and distributed processing on **Google Cloud Platform (GCP)**

The replication is inspired by the published CMS analysis:  
> CMS Collaboration, *Observation of a new boson at a mass of 125 GeV with the CMS experiment at the LHC*,  
> Phys. Lett. B716 (2012) 30–61, [arXiv:1207.7235](https://arxiv.org/abs/1207.7235).

---

## Repository structure

- `docs/` – Thesis PDF, slides, and figures  
- `src/` – C++ macros, CMSSW analyzers, and Python config files  
- `datasets/` – JSON lumi masks, dataset lists, indexfiles  
- `scripts/` – Utilities for fetching data, running analyses, merging ROOT files  
- `cloud/` – Contextualization and helper scripts for CernVM + GCP  
- `results/` – Small ROOT outputs, plots, reports  

> Each level (1–4) will eventually have its own README with specific instructions.

---

## Data sources

All data are obtained from the [CERN Open Data portal](http://opendata.cern.ch).  
Raw data are **not** stored in this repository. Scripts and dataset lists are provided to fetch the required files.

---

## Credits

- Levels 1–3 are adapted from the official CMS Open Data analysis example:  
  [cms-opendata-analyses/HiggsToFourLeptons2011](https://github.com/cms-opendata-analyses/HiggsToFourLeptons2011)  
- My work focuses on **extending the analysis to Level 4**, introducing scalable distributed processing with CernVM and Google Cloud.  

---

## License

- **Code**: MIT  
- **Text and figures** (thesis, slides, plots): CC BY 4.0
