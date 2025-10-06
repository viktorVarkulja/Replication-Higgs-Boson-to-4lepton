# Level 4 — End-to-end, parallelized analysis on CernVM + Google Cloud

Level 4 stitches everything together and **runs the full pipeline at scale**:
1) prepare dataset **index files**,  
2) spin up **CernVM** instances on **Google Cloud (no external IPs)**,  
3) **distribute** event ranges across VMs,  
4) **run** `cmsRun` in parallel,  
5) **merge** outputs and plot final **m4ℓ** distributions.

---

## 0) Prerequisites (local)

- Installed: `gcloud`, `gsutil`, and authenticated (`gcloud auth login`)
- Your Level 3 analyzer/plotting code and Level 4 configs:
  - `HiggsDemoAnalyzer.cc`
  - `demoanalyzer_cfg_level4data.py`, `demoanalyzer_cfg_level4MC.py`
  - JSON lumi masks for 2011 and 2012

---

## 1) Set up your GCP project

Replace placeholders in `<>` and set defaults:

```bash
gcloud config set project <PROJECT_ID>
gcloud config set compute/region <REGION>    # e.g. europe-central2
gcloud config set compute/zone <ZONE>        # e.g. europe-central2-a

# Enable APIs
gcloud services enable compute.googleapis.com iam.googleapis.com iap.googleapis.com oslogin.googleapis.com
```

Create a **service account** and grant minimal roles:

```bash
SA_NAME=compute-runner
SA_EMAIL="$SA_NAME@$(gcloud config get-value project).iam.gserviceaccount.com"

gcloud iam service-accounts create "$SA_NAME" --display-name="Compute Runner (CernVM)"

for ROLE in compute.instanceAdmin.v1 iap.tunnelResourceAccessor storage.objectViewer logging.logWriter monitoring.metricWriter; do
  gcloud projects add-iam-policy-binding $(gcloud config get-value project)     --member="serviceAccount:$SA_EMAIL" --role="roles/$ROLE"
done
```

---

## 2) Import CernVM image

```bash
BUCKET=<YOUR_BUCKET_NAME>
REGION=$(gcloud config get-value compute/region)
gsutil mb -l "$REGION" gs://$BUCKET/
wget https://cernvm.cern.ch/releases/production/cernvm4-micro-2021.05-1.tar.gz
gsutil cp cernvm4-micro-2021.05-1.tar.gz gs://$BUCKET/
gcloud compute images create cern-vm-image --source-uri=gs://$BUCKET/cernvm4-micro-2021.05-1.tar.gz
```

---

## 3) Configure private networking + Cloud NAT

```bash
REGION=$(gcloud config get-value compute/region)

gcloud compute routers create nat-router --network=default --region="$REGION"
gcloud compute routers nats create nat-config   --router=nat-router --region="$REGION"   --nat-all-subnet-ip-ranges --auto-allocate-nat-external-ips
```

SSH will use **IAP tunnels** (no public IPs).

---

## 4) Prepare startup context file

Use your `cms-opendata-startup.context` to configure:
- CVMFS + CMS environment (`SCRAM_ARCH=slc6_amd64_gcc472`)
- `cms-shell` wrapper for CMS environment
- SITECONF + xrootd/EOS mapping
- Download & build analyzer code + Level 4 configs

> Or use `cms-opendata-startup.context` from this repo

---

## 5) Generate dataset index files

```bash
cat > List_indexfile.txt << 'EOF'
/DoubleElectron/Run2011A-12Oct2013-v1/AOD
/DoubleMuParked/Run2012C-22Jan2013-v1/AOD
EOF

mkdir -p indexfiles

while IFS= read -r DATASET; do
  [[ -z "$DATASET" || "${DATASET:0:1}" != "/" ]] && continue
  SAFE=$(echo "$DATASET" | tr '/ ' '__')
  echo "Fetching: $DATASET"
  cernopendata-client get-file-locations --title "$DATASET" --protocol xrootd &> "indexfiles/${SAFE}.txt"
done < List_indexfile.txt
```

> Or use `./level_4/indexfiles` from this repo

Copy index files to each VM:

```bash
for VM in $(cat vm-names); do
  gcloud compute scp --tunnel-through-iap indexfiles/*     "$VM:~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/datasets/"     --zone="$ZONE"
done
```

---

## 6) Create VM fleet

Create a file `vm-names`:

```
doubleelectron-run2011a-1
doubleelectron-run2011a-2
doubleelectron-run2011a-3
```

> Or use `./level_4/vm-names` from this repo

Launch instances:

```bash
PROJECT=$(gcloud config get-value project)
ZONE=$(gcloud config get-value compute/zone)
SA_EMAIL=compute-runner@$PROJECT.iam.gserviceaccount.com
DISK_IMAGE=cern-vm-08-10
STARTUP_FILE=./cms-opendata-startup.context

while read -r VM_NAME; do
  [[ -z "$VM_NAME" ]] && continue
  gcloud compute instances create "$VM_NAME"     --project="$PROJECT"     --zone="$ZONE"     --machine-type=e2-medium     --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default,no-address     --service-account="$SA_EMAIL"     --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write     --create-disk=auto-delete=yes,boot=yes,image="$DISK_IMAGE",mode=rw,size=20,type=pd-standard     --labels=project=cernvm-opendata,vm-name="$VM_NAME"     --metadata-from-file user-data="$STARTUP_FILE"
done < vm-names
```

SSH via IAP:

```bash
gcloud compute ssh cms-opendata@doubleelectron-run2011a-4 --tunnel-through-iap --zone="$ZONE"
# password is *password* 
```

---

## 7) Configure skip / max events, data source and output file

```python
process.maxEvents = cms.untracked.PSet(input = cms.untracked.int32(9894640))

...
 
from FWCore.PythonUtilities import FileUtils
files2012data = FileUtils.loadListFromFile('/home/cms-opendata/CMSSW_5_3_32/src/Demo/DemoAnalyzer/datasets/_DoubleElectron_Run2011A-12Oct2013-v1_AOD.txt')

process.source = cms.Source(
    "PoolSource",
    fileNames = cms.untracked.vstring(*files2012data)
)

...

# Example for 4th VM:
process.source.skipEvents = cms.untracked.uint32(3 * 9894640)

...

process.TFileService = cms.Service("TFileService",
       fileName = cms.string('DoubleElectron_Run2011A-4.root'))
```

---

## 8) Run analysis

```bash
cd ~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/
cmsenv
# for real data datasets
nohup cmsRun demoanalyzer_cfg_level4data.py &> DoubleElectron_Run2011A-4.report &
# OR for MC datasets
nohup cmsRun demoanalyzer_cfg_level4MC.py &> Higgs4L_2012_MC.report &
```

Trim logs before transfer:

```bash
(head -n 200 *.report; tail -n 200 *.report) > DoubleElectron_Run2011A-4_summary.report
```

---

## 9) Merge outputs

Once all analysis VMs have finished running, choose **one main VM** to consolidate results.  
This can be **any of the analysis VMs** or a **new VM**, as long as it uses the **same configuration and environment** (CernVM + CMSSW setup).  
You’ll copy the output files from all worker VMs to this main instance and then merge them by corresponding datasets.

```bash
MAIN=vm-main
ZONE=europe-central2-a
for VM in $(cat vm-names); do
  gcloud compute scp --tunnel-through-iap --zone="$ZONE" \
    "cms-opendata@$VM:~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/*.root" \
    "cms-opendata@$MAIN:~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/rootfiles"

  gcloud compute scp --tunnel-through-iap --zone="$ZONE" \
    "cms-opendata@$VM:~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/*_summary.report" \
    "cms-opendata@$MAIN:~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/rootfiles"
done

gcloud compute ssh cms-opendata@$MAIN --tunnel-through-iap --zone="$ZONE"

cd ~/CMSSW_5_3_32/src/Demo/DemoAnalyzer/rootfiles
hadd -f ./results/DoubleElectron_Run2011A.root ./DoubleElectron_Run2011A-1.root ./DoubleElectron_Run2011A-2.root ...
cd ./results

#Configure M4Lnormdatall.cc input and output files
root -l M4Lnormdatall.cc
```

---

## 10) Clean up

```bash
gcloud compute instances delete $(cat vm-names) --zone="$ZONE"
gcloud compute routers nats delete nat-config --router=nat-router --region=$(gcloud config get-value compute/region)
gcloud compute routers delete nat-router --region=$(gcloud config get-value compute/region)
```

---

### Next

- See potential future projects in [Level 4](./README.md).
- See [Level 3 Code Explanation](../level_3/level_3_code_walkthrough.md) for details.
