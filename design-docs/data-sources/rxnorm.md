# RxNorm Drug Database

## Overview

**RxNorm** is a normalized naming system for clinical drugs produced by the National Library of Medicine. It provides standardized names for clinical drugs and links to many drug vocabularies.

- **URL**: https://rxnav.nlm.nih.gov/
- **API**: https://rxnav.nlm.nih.gov/REST.html
- **Free**: Yes, no API key required
- **Update Frequency**: Daily

---

## What RxNorm Provides

### Core Data
- **Generic Names**: International Nonproprietary Names (INN)
- **Brand Names**: Proprietary drug names
- **Ingredients**: Active and inactive ingredients
- **Dosage Forms**: Tablet, capsule, injection, etc.
- **Strengths**: Amount of active ingredient
- **Drug Interactions**: Known drug interactions
- **Relationships**: Brand-to-generic, ingredient-to-drug mappings

### Unique Identifiers
- **RxCUI**: RxNorm Concept Unique Identifier
- **RXNATOM**: RxNorm Atom (specific form)
- **RXNSTRENGTH**: Strength concept

---

## API Endpoints

### 1. Drug Information

```
GET https://rxnav.nlm.nih.gov/REST/drugs.json?name={drug_name}
```

Search for drugs by name (brand or generic).

```
GET https://rxnav.nlm.nih.gov/REST/rxcui.json?name={drug_name}&search=2
```

Get RxCUI for a drug name.

### 2. Drug Relationships

```
GET https://rxnav.nlm.nih.gov/REST/rxcui/{rxcui}/allProperties.json?prop=properties
```

Get all properties for a drug.

```
GET https://rxnav.nlm.nih.gov/REST/related.json?rxCUI={rxcui}&prop=all
```

Get related drugs (brand, generic, ingredients).

### 3. Drug Interactions

```
GET https://rxnav.nlm.nih.gov/REST/interactions.json?rxCUI={rxcui}
```

Get drug interactions for a given RxCUI.

### 4. Spelling Suggestions

```
GET https://rxnav.nlm.nih.gov/REST/spellingsuggestions.json?name={drug_name}
```

Get spelling suggestions for drug names.

---

## Python Implementation

```python
import requests
from typing import List, Dict, Optional
from dataclasses import dataclass
from enum import Enum

class InteractionSeverity(Enum):
    HIGH = "High"
    MODERATE = "Moderate"
    LOW = "Low"
    N/A = "N/A"

@dataclass
class DrugInfo:
    rxcui: str
    name: str
    tty: str  # Term type (SCD, SBD, IN, BN, etc.)
    synonym: Optional[str] = None
    strength: Optional[str] = None
    dosage_form: Optional[str] = None

@dataclass
class DrugInteraction:
    rxcui1: str
    drug1: str
    rxcui2: str
    drug2: str
    description: str
    severity: str

class RxNormClient:
    """Client for RxNorm API."""
    
    BASE_URL = "https://rxnav.nlm.nih.gov/REST"
    
    def search_drugs(self, name: str) -> List[DrugInfo]:
        """Search for drugs by name."""
        url = f"{self.BASE_URL}/drugs.json"
        params = {"name": name}
        
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        
        data = response.json()
        drugs = []
        
        for group in data.get("drugGroup", {}).get("conceptGroup", []):
            for concept in group.get("conceptProperties", []):
                drugs.append(DrugInfo(
                    rxcui=concept.get("rxcui", ""),
                    name=concept.get("name", ""),
                    tty=concept.get("tty", ""),
                    synonym=concept.get("synonym"),
                    strength=concept.get("strength"),
                    dosage_form=concept.get("doseForm", {}).get("name")
                ))
        
        return drugs
    
    def get_rxnorm_id(self, name: str) -> Optional[str]:
        """Get RxCUI for a drug name."""
        url = f"{self.BASE_URL}/rxcui.json"
        params = {"name": name, "search": "2"}  # Search for exact match
        
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        
        data = response.json()
        id_group = data.get("idGroup", {})
        
        if id_group.get("rxnormId"):
            return id_group["rxnormId"][0]
        
        # Try alternative search
        drugs = self.search_drugs(name)
        return drugs[0].rxcui if drugs else None
    
    def get_related_drugs(self, rxcui: str) -> Dict:
        """Get related drugs (brand, generic, ingredients)."""
        url = f"{self.BASE_URL}/related.json"
        params = {"rxCUI": rxcui, "prop": "all"}
        
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        
        return response.json()
    
    def get_drug_interactions(
        self,
        rxcui: str
    ) -> List[DrugInteraction]:
        """Get drug interactions for a given RxCUI."""
        url = f"{self.BASE_URL}/interactions.json"
        params = {"rxCUI": rxcui}
        
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        
        data = response.json()
        interactions = []
        
        for group in data.get("interactionsGroup", {}).get("interactions", []):
            for pair in group.get("interactionPair", []):
                interaction_concepts = pair.get("interactionConcept", [])
                
                if len(interaction_concepts) >= 2:
                    concept1 = interaction_concepts[0].get("conceptProperties", {})
                    concept2 = interaction_concepts[1].get("conceptProperties", {})
                    
                    interactions.append(DrugInteraction(
                        rxcui1=rxcui,
                        drug1=concept1.get("name", ""),
                        rxcui2=concept2.get("rxcui", ""),
                        drug2=concept2.get("name", ""),
                        description=pair.get("description", ""),
                        severity=pair.get("severity", "N/A")
                    ))
        
        return interactions
    
    def get_all_properties(self, rxcui: str) -> Dict:
        """Get all properties for a drug."""
        url = f"{self.BASE_URL}/rxcui/{rxcui}/allProperties.json"
        params = {"prop": "properties"}
        
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        
        return response.json()


# Usage Examples
client = RxNormClient()

# Search for a drug
metformin = client.search_drugs("Metformin")
print(f"Found {len(metformin)} results for Metformin")
for drug in metformin[:3]:
    print(f"  - {drug.name} (RxCUI: {drug.rxcui}, TTY: {drug.tty})")

# Get RxCUI
rxcui = client.get_rxnorm_id("Metformin")
print(f"Metformin RxCUI: {rxcui}")

# Check drug interactions
if rxcui:
    interactions = client.get_drug_interactions(rxcui)
    print(f"Found {len(interactions)} interactions")
    for interaction in interactions[:3]:
        print(f"  - {interaction.drug2}: {interaction.description}")
```

---

## Check Drug Interactions Between Two Drugs

```python
def check_interaction(drug1: str, drug2: str) -> Dict:
    """Check interactions between two drugs."""
    client = RxNormClient()
    
    # Get RxCUIs
    rxcui1 = client.get_rxnorm_id(drug1)
    rxcui2 = client.get_rxnorm_id(drug2)
    
    if not rxcui1 or not rxcui2:
        return {
            "success": False,
            "error": f"Could not find one or both drugs: {drug1}, {drug2}"
        }
    
    # Get interactions for first drug
    interactions = client.get_drug_interactions(rxcui1)
    
    # Filter for interactions with second drug
    relevant = [
        {
            "drug1": i.drug1,
            "drug2": i.drug2,
            "description": i.description,
            "severity": i.severity
        }
        for i in interactions
        if drug2.lower() in i.drug2.lower()
    ]
    
    return {
        "success": True,
        "drug1": drug1,
        "drug2": drug2,
        "rxcui1": rxcui1,
        "rxcui2": rxcui2,
        "interactions": relevant,
        "has_interaction": len(relevant) > 0
    }


# Example usage
result = check_interaction("Warfarin", "Aspirin")
print(f"Interaction check: {result}")
```

---

## Term Types (TTY)

| TTY | Description | Example |
|-----|-------------|---------|
| SCD | Semantic Clinical Drug | Metformin 500mg Tablet |
| SBD | Semantic Branded Drug | Glucophage 500mg Tablet |
| IN | Ingredient | Metformin |
| BN | Branded Name | Glucophage |
| PIN | Precise Ingredient | Metformin Hydrochloride |
| MIN | Multiple Ingredients | Metformin/Glyburide |
| SCDF | Clinical Drug Form | Metformin Oral Tablet |

---

## Integration with Agent Tools

```python
from langchain.tools import tool

@tool
def lookup_drug(drug_name: str) -> str:
    """Look up drug information by name."""
    client = RxNormClient()
    drugs = client.search_drugs(drug_name)
    
    if not drugs:
        return f"No drugs found for: {drug_name}"
    
    results = []
    for drug in drugs[:5]:
        results.append(
            f"Name: {drug.name}\n"
            f"  RxCUI: {drug.rxcui}\n"
            f"  Type: {drug.tty}\n"
            f"  Strength: {drug.strength or 'N/A'}"
        )
    
    return "\n\n".join(results)

@tool
def check_drug_interactions(drug1: str, drug2: str) -> str:
    """Check interactions between two drugs."""
    result = check_interaction(drug1, drug2)
    
    if not result["success"]:
        return result["error"]
    
    if not result["has_interaction"]:
        return f"No known interactions found between {drug1} and {drug2}"
    
    output = [f"Interactions between {drug1} and {drug2}:\n"]
    for interaction in result["interactions"]:
        output.append(
            f"- {interaction['severity']}: {interaction['description']}"
        )
    
    return "\n".join(output)
```

---

## Rate Limits

- **No official rate limit** for RxNorm API
- **Recommended**: 1 request/second to be respectful
- **Caching**: Safe to cache drug information (changes infrequently)

---

## References

- RxNorm API: https://rxnav.nlm.nih.gov/REST.html
- RxNorm Documentation: https://www.nlm.nih.gov/healthit/snomedct/rxnorm.html
- RxNorm Browser: https://rxnav.nlm.nih.gov/