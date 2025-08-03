# Tutorial: Developing Calorimeter Digitization in FCCSW

This guide explains how to:

1. Understand the simulation/digitization flow
2. Write your own digitization algorithm
3. Plug it into the FCCSW/Key4HEP stack
4. Validate your output

## Background: What is Digitization?

In high-energy physics software, _digitization_ is the step that turns _simulated energy deposits (SimHits)_ from Geant4 into _realistic readout signals_ (DigiHits), as they would appear from actual detector electronics.

## Step 1: Understand the FCCSW Data Flow

### Typical pipeline for a calorimeter:

```
Event Generation
    ↓
Geant4 Simulation (SimCalorimeterHits)
    ↓
Digitization (CalorimeterHits)
    ↓
Reconstruction (Clusters, PF Objects)
    ↓
Analysis
```
Data formats are defined using _EDM4hep_, and Gaudi algorithms orchestrate the pipeline.

## Step 2: Create a New Digitizer Algorithm

You will now write a _C++ digitizer_ that:

* Reads SimCalorimeterHitCollection
* Applies a calibration and threshold
* Outputs CalorimeterHitCollection

### File: `src/MyCalorimeterDigitizer.cpp`

```c++
#include "GaudiAlg/GaudiAlgorithm.h"
#include "DataObjects/SimCalorimeterHitCollection.h"
#include "DataObjects/CalorimeterHitCollection.h"

class MyCalorimeterDigitizer : public GaudiAlgorithm {
public:
  MyCalorimeterDigitizer(const std::string& name, ISvcLocator* svcLoc)
    : GaudiAlgorithm(name, svcLoc) {}

  StatusCode initialize() override {
    info() << "Initializing MyCalorimeterDigitizer" << endmsg;
    return StatusCode::SUCCESS;
  }

  StatusCode execute() override {
    const auto* simHits = get<edm4hep::SimCalorimeterHitCollection>(m_input);
    auto* digiHits = new edm4hep::CalorimeterHitCollection();

    for (const auto& sim : *simHits) {
      double energy = sim.getEnergy() * m_calibration;
      if (energy < m_threshold) continue;

      edm4hep::CalorimeterHit hit;
      hit.setCellID(sim.getCellID());
      hit.setEnergy(energy);
      hit.setTime(sim.getTime());  // could apply time smearing
      digiHits->push_back(hit);
    }

    put(digiHits, m_output);
    return StatusCode::SUCCESS;
  }

private:
  Gaudi::Property<std::string> m_input{this, "input", "ECalBarrelHits"};
  Gaudi::Property<std::string> m_output{this, "output", "ECalBarrelDigiHits"};
  Gaudi::Property<double> m_threshold{this, "threshold", 0.1};        // GeV
  Gaudi::Property<double> m_calibration{this, "calibration", 1.0};    // Scale factor
};

DECLARE_COMPONENT(MyCalorimeterDigitizer)
```

## Step 3: Register the Algorithm

`CMakeLists.txt`

Add this to your digitizer module:

```cmake
gaudi_add_module(MyDigitizers
  SOURCES src/MyCalorimeterDigitizer.cpp
  LINK GaudiAlgLib EDM4hep::edm4hep
)
```

Then build:

```bash
mkdir build && cd build
cmake .. && make install
```

## Step 4: Configure Your Digitizer in Python

`python/digi_config.py`

```python
from Gaudi.Configuration import *
from Configurables import MyCalorimeterDigitizer, PodioOutput

digitizer = MyCalorimeterDigitizer("MyCaloDigi")
digitizer.input = "ECalBarrelHits"
digitizer.output = "ECalBarrelDigiHits"
digitizer.threshold = 0.05
digitizer.calibration = 1.2

out = PodioOutput("out")
out.filename = "digitized.root"
out.outputCommands = ["keep *"]

ApplicationMgr(
    EvtSel="NONE",
    EvtMax=10,
    TopAlg=[digitizer, out],
    OutputLevel=INFO
)
```

Then run:

```python
gaudirun.py python/digi_config.py
```

## Step 5: Validate Output

### With ROOT:

```bash
root digitized.root
```

```cpp
Events->Scan("ECalBarrelDigiHits.energy")
```

### With Python:

```python
import uproot
import awkward as ak

file = uproot.open("digitized.root")
tree = file["events"]
energies = tree["ECalBarrelDigiHits.energy"].array()
print(ak.to_list(energies))
```

## Step 6: Extend It

You can expand your digitizer by:

* Adding Gaussian noise
* Simulating ADC quantization
* Applying time smearing
* Merging SimHits per cell

## Optional: Integrate into Full FCCSW Chain

If you want to use this in an FCCSW pipeline:

* Replace `SimCalorimeterHitDigi` in FCCSW with your own
* Make sure `readoutName` matches the one used in the geometry XML

## References

* [FCCSW GitHub](https://github.com/HEP-FCC/FCCSW)
* [Key4HEP Docs](https://key4hep.github.io/key4hep-doc/)
* [EDM4hep](https://edm4hep.web.cern.ch/)
* [Gaudi](https://gaudi.web.cern.ch/gaudi/)

