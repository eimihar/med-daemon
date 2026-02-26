# Malaysian Medical Data Sources

## Overview

This document covers medical data sources specific to Malaysia, including regulatory bodies, clinical guidelines, drug information, and health databases relevant to the Malaysian healthcare system.

---

## 1. National Pharmaceutical Regulatory Agency (NPRA)

### Overview
- **URL**: https://www.npra.gov.my
- **English Name**: National Pharmaceutical Regulatory Agency
- **Malay Name**: Agensi Regulatori Farmasi Negara
- **Status**: Government agency under Ministry of Health Malaysia
- **API**: No public API available

### What NPRA Provides
- **Drug Registration**: Registered pharmaceutical products in Malaysia
- **Product Search**: Database of registered medicines
- **Consumer Drug Information**: Leaflets, package inserts
- **Adverse Drug Reactions**: Reporting system

### Key Resources

#### Product Search Database
```
URL: https://www.npra.gov.my/eng/public-database/search-product.html
Content: Searchable database of registered drugs
Access: Free, web-based search
```

#### QUEST (Quest for ADR Reporting)
```
URL: https://www.npra.gov.my/eng/publications/adr-reporting.html
Content: Adverse drug reaction reporting system
Access: Healthcare professional registration required
```

### Implementation Approach

```python
import requests
from bs4 import BeautifulSoup
from typing import List, Dict, Optional
import re

class NPRAClient:
    """Client for NPRA drug database (via web scraping)."""
    
    BASE_URL = "https://www.npra.gov.my"
    SEARCH_URL = "https://www.npra.gov.my/eng/public-database/search-product.html"
    
    def search_drug(self, name: str) -> List[Dict]:
        """Search for registered drugs in NPRA database."""
        
        # Note: NPRA doesn't have a public API, web scraping required
        # This is a conceptual implementation
        
        headers = {
            "User-Agent": "Mozilla/5.0 (compatible; MedDaemon/1.0)"
        }
        
        # The actual implementation would need to handle form submission
        # and parse the response HTML
        
        return []
    
    def get_drug_details(self, product_id: str) -> Dict:
        """Get detailed information about a specific drug."""
        # Would require scraping the product detail page
        return {}


# Alternative: Maintain a local copy of NPRA registration data
# This would need periodic updates from NPRA website

class MalaysianDrugDatabase:
    """Local database of NPRA-registered drugs."""
    
    def __init__(self, db_path: str):
        self.db_path = db_path
        self.drugs = self._load_database()
    
    def _load_database(self) -> List[Dict]:
        """Load drug database from local storage."""
        import json
        try:
            with open(self.db_path, 'r', encoding='utf-8') as f:
                return json.load(f)
        except FileNotFoundError:
            return []
    
    def search(self, query: str) -> List[Dict]:
        """Search drugs by name, generic name, or manufacturer."""
        query_lower = query.lower()
        results = []
        
        for drug in self.drugs:
            if (query_lower in drug.get('product_name', '').lower() or
                query_lower in drug.get('generic_name', '').lower() or
                query_lower in drug.get('manufacturer', '').lower()):
                results.append(drug)
        
        return results
    
    def get_by_registration_number(self, reg_no: str) -> Optional[Dict]:
        """Get drug by NPRA registration number."""
        for drug in self.drugs:
            if drug.get('registration_number') == reg_no:
                return drug
        return None
```

---

## 2. Ministry of Health Malaysia (KKM)

### Overview
- **URL**: https://www.moh.gov.my
- **Malay Name**: Kementerian Kesihatan Malaysia
- **Content**: Health policies, clinical guidelines, disease surveillance

### Key Resources

#### Clinical Practice Guidelines
```
URL: https://www.moh.gov.my/index.php/pages/view/clinical-practice-guidelines
Content: Ministry-approved clinical guidelines
Format: PDF downloads
```

#### National Health Surveillance
```
URL: https://www.moh.gov.my/index.php/pages/view/press-release
Content: Disease outbreaks, health alerts
```

#### Malaysian Health Technology Assessment (MaHTAS)
```
URL: https://www.moh.gov.my/moh/calls/hta/
Content: Health technology assessments
Format: PDF reports
```

---

## 3. Malaysian Medical Council (MMC)

### Overview
- **URL**: https://www.mmc.gov.my
- **Function**: Medical practitioner registration and regulation
- **Content**: Registered doctors database, ethical guidelines

### Key Resources

#### Registered Doctors Search
```
URL: https://www.mmc.gov.my/cpl/
Content: Search for registered medical practitioners
Access: Public search available
```

#### Medical Ethics Guidelines
```
URL: https://www.mmc.gov.my/cpl/medical-ethics/
Content: Code of professional conduct
Format: PDF documents
```

---

## 4. Clinical Research Malaysia (CRM)

### Overview
- **URL**: https://www.clinicalresearch.my
- **Function**: Clinical trial coordination and support
- **Content**: Clinical trials in Malaysia

### Key Resources

#### Clinical Trials Registry
```
URL: https://www.clinicaltrials.gov (filter for Malaysia)
Content: International clinical trials conducted in Malaysia
API: ClinicalTrials.gov API available
```

---

## 5. Malaysian Health Technology Assessment Section (MaHTAS)

### Overview
- **URL**: https://www.moh.gov.my/moh/calls/hta/
- **Function**: Health technology assessment reports
- **Content**: Clinical guidelines based on health technology assessments

### What MaHTAS Provides
- **Technology Assessments**: Evaluations of medical technologies
- **Clinical Practice Guidelines**: Evidence-based recommendations
- **Horizon Scanning**: Emerging health technologies

### Implementation

```python
class MaHTASClient:
    """Client for MaHTAS health technology assessments."""
    
    BASE_URL = "https://www.moh.gov.my/moh/calls/hta/"
    
    def list_assessments(self) -> List[Dict]:
        """List available health technology assessments."""
        # Would require web scraping
        return []
    
    def get_assessment(self, assessment_id: str) -> Dict:
        """Get specific assessment report."""
        # Would require downloading PDF and parsing
        return {}
```

---

## 6. Malaysian Specialty Society Guidelines

### Malaysian Society of Anaesthesiologists (MSA)
- **URL**: https://www.msah.net.my
- **Content**: Anaesthesia guidelines, sedation protocols

### National Heart Association of Malaysia (NHAM)
- **URL**: https://www.nham.com.my
- **Content**: Cardiology guidelines, acute coronary syndrome protocols

### Malaysian Diabetes Association (MDA)
- **URL**: https://www.diabetes.org.my
- **Content**: Diabetes management guidelines

### Malaysian Society of Gastroenterology & Hepatology
- **URL**: https://msgh.org.my
- **Content**: GI/ Hepatology guidelines

### Malaysian Paediatric Association (MPA)
- **URL**: www.mpaweb.org.my
- **Content**: Paediatric guidelines

---

## 7. Hospital Guidelines (Major Malaysian Hospitals)

### Hospital Universiti Malaya (HUM)
- **URL**: https://www.ummc.edu.my
- **Guidelines**: Various clinical protocols

### Hospital Kuala Lumpur
- **URL**: https://www.hkl.gov.my
- **Guidelines**: Institutional clinical guidelines

### Hospital Ampang
- **Guidelines**: Available via hospital intranet

### Hospital Selayang
- **Guidelines**: Available via hospital intranet

---

## 8. Malaysian Drug Database (Pharmacy-Specific)

### Pharmacy Board Malaysia
- **URL**: https://www.pharmacy.gov.my
- **Content**: Pharmacist registration, pharmacy regulations

### Malaysian Community Pharmacy Guild
- **URL**: https://www.mcpg.org.my
- **Content**: Community pharmacy information

### Implementation: Malaysian Drug Formulary

```python
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum

class DrugCategory(Enum):
    PRESCRIPTION = "Prescription Only Medicine (POM)"
    OTC = "Over-The-Counter (OTC)"
    POISON = "Group C Poison"
    NATURAL = "Natural Medicine"

@dataclass
class MalaysianDrug:
    """Represents a drug registered in Malaysia."""
    registration_number: str  # NPRA registration number
    product_name: str
    generic_name: str
    manufacturer: str
    dosage_form: str
    strength: str
    category: DrugCategory
    package_size: str
    indication: str
    malaysia_specific: bool  # True if uniquely Malaysian formulation

class MalaysianDrugFormulary:
    """Local formulary of Malaysian-registered drugs."""
    
    def __init__(self, data_path: str):
        self.data_path = data_path
        self.drugs: List[MalaysianDrug] = []
        self._load()
    
    def _load(self):
        """Load drug formulary from local storage."""
        import json
        with open(self.data_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
            for item in data:
                self.drugs.append(MalaysianDrug(**item))
    
    def search_by_generic(self, generic: str) -> List[MalaysianDrug]:
        """Search by generic/INN name."""
        generic_lower = generic.lower()
        return [d for d in self.drugs 
                if generic_lower in d.generic_name.lower()]
    
    def search_by_brand(self, brand: str) -> List[MalaysianDrug]:
        """Search by brand/product name."""
        brand_lower = brand.lower()
        return [d for d in self.drugs 
                if brand_lower in d.product_name.lower()]
    
    def get_by_registration(self, reg_no: str) -> Optional[MalaysianDrug]:
        """Get drug by NPRA registration number."""
        for drug in self.drugs:
            if drug.registration_number == reg_no:
                return drug
        return None
    
    def get_by_manufacturer(self, manufacturer: str) -> List[MalaysianDrug]:
        """Get all drugs from a specific manufacturer."""
        manuf_lower = manufacturer.lower()
        return [d for d in self.drugs 
                if manuf_lower in d.manufacturer.lower()]
```

---

## 9. Integration with Malaysian Context

### Cross-Reference: NPRA + RxNorm

```python
class MalaysianDrugLookup:
    """Integrated lookup for Malaysian drugs with RxNorm mapping."""
    
    def __init__(self, npra_formulary: MalaysianDrugFormulary, 
                 rxnorm_client):
        self.npra = npra_formulary
        self.rxnorm = rxnorm_client
    
    def lookup(self, query: str) -> Dict:
        """Comprehensive drug lookup for Malaysian context."""
        results = {
            "malaysian": self.npra.search_by_generic(query),
            "rxnorm": []
        }
        
        # Also search RxNorm for international context
        try:
            rxnorm_drugs = self.rxnorm.search_drugs(query)
            results["rxnorm"] = rxnorm_drugs
        except Exception:
            pass
        
        return results
    
    def get_full_profile(self, drug_name: str) -> Dict:
        """Get complete drug profile with Malaysian and international data."""
        
        # Search Malaysian formulary
        my_drugs = self.npra.search_by_generic(drug_name)
        
        # Search RxNorm
        try:
            rxcui = self.rxnorm.get_rxnorm_id(drug_name)
            rxnorm_interactions = []
            if rxcui:
                rxnorm_interactions = self.rxnorm.get_drug_interactions(rxcui)
        except Exception:
            rxcui = None
            rxnorm_interactions = []
        
        return {
            "malaysian_products": my_drugs,
            "international_rxnorm": {
                "rxcui": rxcui,
                "interactions": rxnorm_interactions
            }
        }
```

---

## 10. Malaysian-Specific Medical Guidelines

### Common Guidelines to Integrate

```python
MALAYSIAN_GUIDELINES = {
    "diabetes": {
        "organization": "Malaysian Diabetes Association",
        "title": "Clinical Practice Guidelines for Diabetes Mellitus",
        "url": "https://www.diabetes.org.my",
        "year": 2020
    },
    "hypertension": {
        "organization": "National Heart Association of Malaysia",
        "title": "Clinical Practice Guidelines for Hypertension",
        "url": "https://www.nham.com.my",
        "year": 2018
    },
    "asthma": {
        "organization": "Malaysian Thoracic Society",
        "title": "Asthma Management Guidelines",
        "url": "https://www.malaysianthoracic.org.my",
        "year": 2019
    },
    "dengue": {
        "organization": "Ministry of Health Malaysia",
        "title": "Dengue Clinical Management Guidelines",
        "url": "https://www.moh.gov.my",
        "year": 2022
    },
    "tuberculosis": {
        "organization": "Ministry of Health Malaysia",
        "title": "National TB Management Guidelines",
        "url": "https://www.moh.gov.my",
        "year": 2021
    },
    "covid19": {
        "organization": "Ministry of Health Malaysia",
        "title": "COVID-19 Management Guidelines",
        "url": "https://covid19.moh.gov.my",
        "year": 2023
    }
}
```

---

## 11. Data Sources Summary

| Source | URL | Content | API | Access |
|--------|-----|---------|-----|--------|
| **NPRA** | npra.gov.my | Drug registration | No | Web search |
| **Ministry of Health** | moh.gov.my | Guidelines, policies | No | Web/PDF |
| **MaHTAS** | moh.gov.my/hta | HTA reports | No | PDF |
| **MMC** | mmc.gov.my | Doctor registration | No | Web search |
| **ClinicalTrials.gov** | clinicaltrials.gov | Malaysia trials | Yes | API |
| **MDA** | diabetes.org.my | Diabetes guidelines | No | Web/PDF |
| **NHAM** | nham.com.my | Cardiology guidelines | No | Web/PDF |
| **MSA** | msah.net.my | Anaesthesia guidelines | No | Web/PDF |

---

## 12. Implementation Priority for Malaysian Sources

### Phase 1: NPRA Drug Database
1. Create local NPRA database from web scraping or data dumps
2. Implement search by brand/generic name
3. Map to RxNorm for international context

### Phase 2: Ministry of Health Guidelines
1. Download key clinical guidelines as PDFs
2. Parse and index for RAG system
3. Prioritize: Diabetes, Hypertension, Dengue, TB

### Phase 3: Specialty Society Guidelines
1. Collect specialty-specific guidelines
2. Index and integrate into knowledge base

### Phase 4: Real-time Updates
1. Monitor NPRA for new registrations
2. Track MOH for guideline updates
3. Periodic database refresh

---

## 13. Notes on Malaysian Healthcare Context

### Key Differences from US/UK Sources
1. **Drug Names**: Some medications have different brand names in Malaysia
2. **Formulations**: Some formulations are Malaysia-specific
3. **Guidelines**: Local adaptations of international guidelines
4. **Language**: Some documents in Bahasa Malaysia

### Important Considerations
- NPRA registration numbers are specific to Malaysia
- Some generic drugs have different approved names
- Dosage recommendations may differ from international guidelines
- Local disease patterns (e.g., dengue, malaria) require Malaysian-specific protocols

---

## References

- NPRA: https://www.npra.gov.my
- Ministry of Health Malaysia: https://www.moh.gov.my
- Malaysian Medical Council: https://www.mmc.gov.my
- Clinical Research Malaysia: https://www.clinicalresearch.my
- Malaysian Diabetes Association: https://www.diabetes.org.my
- National Heart Association Malaysia: https://www.nham.com.my
