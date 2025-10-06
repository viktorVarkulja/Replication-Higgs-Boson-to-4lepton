# Level 2 — Reproducing the reference 4-lepton mass plot (mass4l_combine)

Level 2 focuses on **reproducing the reference 4-lepton invariant mass plot** (*mass4l_combine*) from the official CMS Higgs → ZZ → 4ℓ analysis.  
The goal is to use **publicly available ROOT files** and **C++ macro code** to generate a histogram that matches the published CMS result from 2012.  
This level demonstrates **data transparency** and **independent reproducibility** of the CMS Open Data analysis.

---

## Environment setup

The replication was performed using the **CMS Open Data Virtual Machine (VM)**, available from the [CERN Open Data portal](https://opendata.cern.ch/docs/cms-virtual-machine-2011).  
This environment includes:
- **ROOT** libraries  
- **CMSSW** (CMS Software Framework)  
- Preconfigured access to Open Data files  

It ensures full compatibility with the original CMS analysis conditions.

---

## Step 1 — Setting up the environment

All commands in this section must be executed inside the CMS Shell within the CMS Open Data Virtual Machine.
Running them in a standard terminal will result in missing environment variables and ROOT/CMSSW dependencies not being found.

In the CMS VM open the CMS Shell :

```bash
# Create and enter CMSSW working directory 
cmsrel CMSSW_5_3_32
cd CMSSW_5_3_32/src
cmsenv

# Create a folder for input ROOT files
mkdir rootfiles && cd rootfiles
```

---

## Step 2 — Downloading the input files

Use `wget` to download the provided ROOT file list and all related datasets:

```bash
wget https://opendata.web.cern.ch/record/5501/files/rootfilelist.txt
wget -i rootfilelist.txt
```

> ⚠️ Note:  
> The file list may require updating to use **HTTPS** instead of **HTTP** links for secure download.

---

## Step 3 — Downloading and running the macro

The analysis uses the C++ macro **`M4Lnormdatall.cc`**, available from the CMS Open Data repository:

```bash
wget https://opendata.web.cern.ch/record/5500/files/M4Lnormdatall.cc
root -l M4Lnormdatall.cc
```

This macro:
- Loads all required ROOT files (data + Monte Carlo simulations)
- Defines integrated luminosities
- Applies **scaling factors** and **cross sections** for each process
- Generates normalized histograms for:
  - Backgrounds (ZZ, TTBar, DY)
  - Signal (H → ZZ → 4ℓ)
  - Experimental data (DoubleMu, DoubleE)

---

## What the C++ macro does

Once executed, the `M4Lnormdatall.cc` macro performs all the key operations required to reproduce the CMS 4-lepton mass plot.

### 1. Data scaling and normalization

Simulated datasets are normalized using the standard luminosity–cross section relation:

\[
\omega = \frac{L \times \sigma \times SF}{N_{\text{events}}}
\]

Where:
- **L** — integrated luminosity (fb⁻¹)  
- **σ** — theoretical cross section (pb)  
- **SF** — scale factor (if applicable)  
- **Nₑvents** — total number of generated events  

This scaling ensures that simulated samples match the expected event yields from real experimental conditions.  
Experimental datasets (`DoubleMu`, `DoubleE`) are loaded **without scaling**, since they already represent measured counts.

### 2. Visualization and final output

After processing all input files, the macro creates a stacked histogram using ROOT’s `THStack` class.  
Each process is represented by a distinct visual style:

- **Backgrounds:** filled histograms (ZZ = blue, DY = green, TTBar = gray)  
- **Signal (H → ZZ → 4ℓ):** red line  
- **Experimental data:** black markers with error bars  

The final diagram includes:
- Labels for center-of-mass energy (`√s = 7, 8 TeV`)  
- Total integrated luminosities  
- A CMS Open Data header and legend for clarity  

<p align="center">
  <img src="./mass4l_combine.png" alt="Reproduced 4-lepton invariant mass plot" width="600"/>
</p>

*Reproduced 4-lepton invariant mass distribution (H → ZZ → 4ℓ) showing the Higgs peak around 125 GeV.*


---

## Outcome and interpretation

The resulting histogram **accurately matches** the published CMS result.  
The **distinct peak near 125 GeV** corresponds to the Higgs boson signal, while background processes are clearly separated and well-modeled.  

This confirms that it is possible to **replicate the official CMS results** using only open data and standard ROOT/CMSSW tools — demonstrating transparency, accessibility, and scientific reproducibility.

---

### Next step
Continue to [Level 3 →](../level_3/README.md), where simulated and experimental data are processed directly through the `HiggsDemoAnalyzer` module.
