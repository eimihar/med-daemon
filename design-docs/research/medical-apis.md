# Medical Data Source APIs

## RxNorm (Drug Database)

### Overview
- **API**: https://rxnav.nlm.nih.gov/REST.html
- **Purpose**: Drug names, RxCUI, relationships, interactions
- **Free**: Yes, no API key required

### Key Endpoints

```python
import requests
from typing import List, Dict

class RxNormClient:
    BASE_URL = "https://rxnav.nlm.nih.gov/REST"
    
    def get_drug_info(self, name: str) -> List[Dict]:
        """Get drug info by name."""
        url = f"{self.BASE_URL}/drugs.json?name={name}"
        response = requests.get(url)
        return response.json().get("drugGroup", {}).get("conceptGroup", [])
    
    def get_rxnorm_id(self, name: str) -> str:
        """Get RxCUI for a drug name."""
        url = f"{self.BASE_URL}/rxcui.json?name={name}"
        response = requests.get(url)
        id_group = response.json().get("idGroup", {})
        return id_group.get("rxnormId", [None])[0]
    
    def get_drug_interactions(self, rxcui: str) -> List[Dict]:
        """Get drug interactions."""
        url = f"{self.BASE_URL}/interactions.json?rxCUI={rxcui}"
        response = requests.get(url)
        return response.json().get("interactionsGroup", {}).get("interactions", [])
    
    def get_related_drugs(self, rxcui: str) -> Dict:
        """Get related drugs (brand, generic, ingredients)."""
        url = f"{self.BASE_URL}/related.json?rxCUI={rxcui}&prop=all"
        response = requests.get(url)
        return response.json()

# Usage
rxnorm = RxNormClient()
rxcui = rxnorm.get_rxnorm_id("Metformin")
interactions = rxnorm.get_drug_interactions(rxcui)
```

### Drug Interaction Example

```python
def check_drug_interaction(drug1: str, drug2: str) -> Dict:
    """Check interactions between two drugs."""
    rxnorm = RxNormClient()
    
    rxcui1 = rxnorm.get_rxnorm_id(drug1)
    rxcui2 = rxnorm.get_rxnorm_id(drug2)
    
    if not rxcui1 or not rxcui2:
        return {"error": "Drug not found"}
    
    # Get interactions for first drug
    url = f"https://rxnav.nlm.nih.gov/REST/interactions.json?rxCUI={rxcui1}"
    response = requests.get(url)
    data = response.json()
    
    # Find interactions with second drug
    interactions = []
    for group in data.get("interactionsGroup", {}).get("interactions", []):
        for interaction in group.get("interactionPair", []):
            if rxcui2 in interaction.get("interactionConcept", []):
                interactions.append({
                    "description": interaction.get("description"),
                    "severity": interaction.get("severity"),
                    "documentation": interaction.get("documentation")
                })
    
    return {"drug1": drug1, "drug2": drug2, "interactions": interactions}
```

---

## FDA Drug Labels API

### Overview
- **API**: https://api.fda.gov/drug/label.json
- **Purpose**: Drug labels, warnings, indications
- **Free**: Yes

### Examples

```python
import requests
from typing import List, Dict

class FDADrugClient:
    BASE_URL = "https://api.fda.gov/drug/label.json"
    
    def search_by_brand(self, brand_name: str, limit: int = 10) -> List[Dict]:
        """Search drugs by brand name."""
        url = f"{self.BASE_URL}?search=brand_name:\"{brand_name}\"&limit={limit}"
        response = requests.get(url)
        return response.json().get("results", [])
    
    def search_by_active_ingredient(self, ingredient: str, limit: int = 10) -> List[Dict]:
        """Search by active ingredient."""
        url = f"{self.BASE_URL}?search=active_ingredient:\"{ingredient}\"&limit={limit}"
        response = requests.get(url)
        return response.json().get("results", [])
    
    def get_label_info(self, product_ndc: str) -> Dict:
        """Get full label for a product."""
        url = f"{self.BASE_URL}?search=product_ndc:\"{product_ndc}\"&limit=1"
        response = requests.get(url)
        results = response.json().get("results", [])
        return results[0] if results else {}

# Usage
fda = FDADrugClient()
labels = fda.search_by_brand("Metformin")
label = fda.get_label_info("0002-3224-90")

# Extract key information
if label:
    print(f"Indications: {label.get('indications_and_usage', ['N/A'])}")
    print(f"Warnings: {label.get('warnings', ['N/A'])}")
    print(f"Adverse Reactions: {label.get('adverse_reactions', ['N/A'])}")
```

---

## MedlinePlus

### Overview
- **URL**: https://medlineplus.gov/webguide/
- **Purpose**: Consumer health information
- **Access**: Direct scraping or their API

### Topics Available
- Diseases and conditions
- Drugs and supplements
- Nutrition
- Mental health
- Exercise

```python
import requests
from bs4 import BeautifulSoup

class MedlinePlusClient:
    BASE_URL = "https://medlineplus.gov"
    
    def search_topic(self, query: str) -> List[Dict]:
        """Search MedlinePlus topics."""
        url = f"{self.BASE_URL}/api/Connect/search/{query}"
        response = requests.get(url)
        return response.json()
    
    def get_topic(self, topic_id: str) -> Dict:
        """Get topic content."""
        url = f"{self.BASE_URL}/api/Connect/topics/{topic_id"
        response = requests.get(url)
        return response.json()
    
    def get_drug_info(self, drug_name: str) -> str:
        """Get drug information page."""
        url = f"{self.BASE_URL}/drugs/{drug_name.lower().replace(' ', '')}.html"
        response = requests.get(url)
        soup = BeautifulSoup(response.text, 'html.parser')
        
        # Extract content
        content = soup.find('div', class_='topic-content')
        return content.get_text() if content else ""
```

---

## ClinicalTrials.gov API

### Overview
- **API**: https://clinicaltrials.gov/api/api_home
- **Purpose**: Clinical trial information
- **Free**: Yes

```python
class ClinicalTrialsClient:
    BASE_URL = "https://clinicaltrials.gov/api/v2"
    
    def search_studies(self, condition: str, interventions: List[str] = None) -> List[Dict]:
        """Search for clinical trials."""
        query = f"AREA[Condition]={condition}"
        if interventions:
            for i in interventions:
                query += f" AND AREA[InterventionName]={i}"
        
        url = f"{self.BASE_URL}/studies?query.cond={condition}&pageSize=20"
        response = requests.get(url)
        return response.json().get("studies", [])
    
    def get_study_details(self, nct_id: str) -> Dict:
        """Get detailed study information."""
        url = f"{self.BASE_URL}/studies/{nct_id}"
        response = requests.get(url)
        return response.json()

# Usage
trials = ClinicalTrialsClient()
studies = trials.search_studies("Type 2 Diabetes", ["Metformin"])
```

---

## Summary of Data Sources

| Source | Type | Use Case |
|--------|------|----------|
| **PubMed** | Literature | Research, citations |
| **RxNorm** | Drugs | Drug names, interactions |
| **FDA Labels** | Drugs | Warnings, indications |
| **MedlinePlus** | Consumer health | Patient-facing info |
| **ClinicalTrials.gov** | Trials | Clinical trial data |

---

## Integration Strategy

1. **PubMed**: Primary source for medical literature (RAG knowledge base)
2. **RxNorm**: Drug lookup tool for agent
3. **FDA**: Drug label reference
4. **MedlinePlus**: Consumer-friendly explanations

```python
# Agent tool definitions
@tool
def search_medical_literature(query: str) -> str:
    """Search PubMed for medical literature."""
    ...

@tool
def check_drug_interaction(drug1: str, drug2: str) -> Dict:
    """Check interactions between two drugs using RxNorm."""
    ...

@tool
def get_drug_label(drug_name: str) -> Dict:
    """Get FDA drug label information."""
    ...
```