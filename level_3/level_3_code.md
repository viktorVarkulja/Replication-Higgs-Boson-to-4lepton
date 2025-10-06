# Level 3 — Code walkthrough (Python configs & C++ analyzers)

This document explains the **Python configuration files** and **C++ source files** used in **Level 3**.  
It focuses on how raw CMS Open Data are read, filtered, and written to new ROOT files, and how those files are visualized.

> [!IMPORTANT] 
> Run all commands and CMSSW jobs **inside the CMS Shell** (`cmsenv`) within the **CMS Open Data VM**.  
> A regular terminal will miss the environment and dependencies.

---

## Python configuration files (CMSSW)

Level 3 uses two CMSSW Python configs to drive the C++ analyzer:

- `demoanalyzer_cfg_level3data.py` (referred to as **`data.py`**) — reads **experimental data** (DoubleMuParked, 2012).  
- `demoanalyzer_cfg_level3MC.py` (referred to as **`MC.py`**) — reads **Monte Carlo** signal (H → ZZ → 4ℓ, 8 TeV).

These files configure a `cms.Process("Demo")`, logging, sources, *optional lumi filtering*, and the output service. Both launch the same EDAnalyzer: **`HiggsDemoAnalyzer`**.

### `data.py` — experimental data + JSON lumi mask

Key lines (simplified):

```python
# define JSON file for 2012 data
goodJSON = '/home/cms-opendata/CMSSW_5_3_32/src/Demo/DemoAnalyzer/datasets/Cert_190456-208686_8TeV_22Jan2013ReReco_Collisions12_JSON.txt'
myLumis = LumiList.LumiList(filename=goodJSON).getCMSSWString().split(',')

# to speed up, pick single example file with 1 nice 2mu2e Higgs candidate (9058 events)
process.source = cms.Source(
    "PoolSource",
    fileNames=cms.untracked.vstring(
        'root://eospublic.cern.ch//eos/opendata/cms/Run2012C/DoubleMuParked/AOD/22Jan2013-v1/10000/F2878994-766C-E211-8693-E0CB4EA0A939.root'
    )
)

# Apply lumi mask (typical usage in config)
# process.source.lumisToProcess = cms.untracked.VLuminosityBlockRange(*myLumis)
```

**What this does**  
- Selects a curated **JSON certification file** (a *lumi mask*) so only data taken with fully operational detector subsystems are processed.  
- Reads one AOD file with **9058 events**, containing **one Higgs candidate** (2μ2e) — ideal for a fast tutorial run.

### `MC.py` — Monte Carlo signal

Key lines (simplified):

```python
# to speed up, read only first file with 7499 events
process.source = cms.Source(
    "PoolSource",
    fileNames=cms.untracked.vstring(
        'root://eospublic.cern.ch//eos/opendata/cms/MonteCarlo2012/Summer12_DR53X/SMHiggsToZZTo4L_M-125_8TeV-powheg15-JHUgenV3-pythia6/AODSIM/PU_S10_START53_V19-v1/10000/029D759D-6CD9-E211-B3E2-1CC1DE041FD8.root'
    )
)

# No JSON filtering for MC
```

**What this does**  
- Reads a **signal MC** file with **~7.5k events**.  
- No JSON lumi mask is needed for MC (there are no detector downtime periods).

### Shared structure (both configs)

- **Process and logging**
  - `cms.Process("Demo")` and `MessageLogger` for concise output.  
  - Set `maxEvents` (often `-1` for “process all”).

- **EDAnalyzer instantiation**
  ```python
  process.demo = cms.EDAnalyzer('HiggsDemoAnalyzer')
  ```
  Launches the C++ analyzer that loops events, selects leptons, builds Z and 4ℓ candidates, and fills histograms.

- **Output ROOT file**
  - Via `TFileService` (declared in the config) so histograms/ntuples are written automatically by the analyzer.

- **Resulting files**
  - `DoubleMuParked2012C_10000_Higgs.root` — real data with one highlighted Higgs candidate.  
  - `Higgs4L1file.root` — reduced-statistics signal MC for quick checks.

> [!TIP] 
> The configs show **single-file** sources for speed. For larger studies, switch to **index lists** (`.txt`) with many files.

---

## C++ analyzer: `HiggsDemoAnalyzer.cc`

**Role:** Central CMSSW EDAnalyzer that processes raw events (data or MC), performs physics selections, and books/fills histograms. At the end, a ROOT file is produced with distributions for control and signal regions.

### Class skeleton

```cpp
class HiggsDemoAnalyzer : public edm::EDAnalyzer {
public:
  explicit HiggsDemoAnalyzer(const edm::ParameterSet&);
  ~HiggsDemoAnalyzer();
private:
  void beginJob() override;
  void analyze(const edm::Event&, const edm::EventSetup&) override;
  void endJob() override;
  bool providesGoodLumisection(const edm::Event& iEvent);

  // Histograms (examples)
  TH1D *h_globalmu_size, *h_recomu_size, *h_e_size;
  // ... many more (kinematics, control, mass distributions)
};
```

### Histogram booking (constructor)

```cpp
HiggsDemoAnalyzer::HiggsDemoAnalyzer(const edm::ParameterSet& iConfig) {
  edm::Service<TFileService> fs;

  // Example control plots
  h_globalmu_size = fs->make<TH1D>("NGMuons", "GMuon size", 10, 0., 10.);
  h_globalmu_size->GetXaxis()->SetTitle("Number of GMuons");
  h_globalmu_size->GetYaxis()->SetTitle("Number of Events");

  // Momentum of Global Muon
  TH1D* h_p_gmu = fs->make<TH1D>("GM_momentum", "GM momentum", 200, 0., 200.);
  h_p_gmu->GetXaxis()->SetTitle("Momentum (GeV/c)");
  h_p_gmu->GetYaxis()->SetTitle("Number of Events");

  // ... plus many physics histograms for 4μ, 4e, 2μ2e channels (mass, pT, η, etc.)
}
```

**What this does**  
- Declares and books **ROOT histograms** once at job start.  
- Sets binning, ranges, titles, and axes labels for physics/control variables.

### Event loop (`analyze`)

```cpp
void HiggsDemoAnalyzer::analyze(const edm::Event& iEvent, const edm::EventSetup& iSetup) {
  // Run, event, lumi
  nRun = iEvent.run();
  nEvt = (iEvent.id()).event();
  nLumi = iEvent.luminosityBlock();

  // Access collections
  edm::Handle<reco::MuonCollection> muons;
  iEvent.getByLabel("muons", muons);

  edm::Handle<reco::GsfElectronCollection> electrons;
  iEvent.getByLabel("gsfElectrons", electrons);

  // Initialize working variables (e.g., masses) to sentinel values
  eZ12 = eZ34 = -9999.;

  // Basic selection loop (example for muons)
  for (unsigned u = 0; u < muons->size(); ++u) {
    const reco::Muon &itMuon = (*muons)[u];
    if (/* PF, isolation, track quality, pT, |η| cuts */) {
      // keep candidate; fill control hists
    }
  }

  // Build Z candidates (2e, 2μ) and 4ℓ (2e2μ, 4e, 4μ) with charge & mass checks
  // Compute m(Z1), m(Z2), m(4ℓ), and fill physics histograms
}
```

**What this does**  
- Retrieves needed **reco collections** (muons, electrons, tracks, vertices).  
- Applies **quality and kinematic cuts** (isolation, impact parameters, pT, |η|, etc.).  
- Constructs **Z** and **4ℓ** candidates with charge balancing and invariant mass calculations (via `TLorentzVector`).  
- Fills **mass distributions** like `mass4mu`, `mass4e`, `mass2e2mu`, `massZ1`, `massZ2`, …

### Output

- Histograms and (optionally) ntuples are written via **`TFileService`** to the output ROOT file selected in the Python configs.  
- For the tutorial-sized inputs:  
  - **Data** yields **one highlighted Higgs candidate** near **125 GeV**.  
  - **MC** provides a **reference signal shape** with limited statistics.

---

## Plotting macro: `M4Lnormdatall_lvl3.cc`

**Role:** Load the new Level 3 ROOT outputs together with other backgrounds, apply normalization, and draw the **four-lepton invariant mass** stack plot.

- Uses the same logic as the Level 2 macro (`M4Lnormdatall.cc`) but **adds the Level 3 outputs** as inputs.  
- Highlights the data **Higgs candidate** (e.g., a **blue triangle**) on top of the stack.  
- Produces the updated figure used in Level 3 documentation.

> See your `level_3.md` for the execution snippet and resulting plot.

---

## Practical notes & pitfalls

- **CMS Shell required:** Always `cmsenv` before compiling/running.  
- **HTTPS sources:** If any file list uses `http`, switch to `https` for downloads.  
- **Scaling in Level 4:** Histogram normalization happens in the plotting macros; per-event weights are not applied inside `HiggsDemoAnalyzer` in this tutorial setup.  
- **From single file to many:** For larger stats, use **PoolSource** with a `vstring` of many files or read from a **text index**.

---

### Outputs used downstream

- `DoubleMuParked2012C_10000_Higgs.root` — real data (one candidate).  
- `Higgs4L1file.root` — MC signal.  
Both are consumed by `M4Lnormdatall_lvl3.cc` to make the Level 3 plot.

---

### Next step
Continue to [Level 4 →](../level_4/README.md), where the analysis is parallelized and distributed across multiple **CernVM** instances on **Google Cloud Platform (GCP)**.
