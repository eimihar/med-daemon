# FDA Drug Data Sources

## Overview

The FDA provides several APIs for drug-related information including labels, recalls, adverse events, and approvals.

---

## 1. openFDA Drug Labels API

### Endpoint
```
GET https://api.fda.gov/drug/label.json
```

### Search Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `search` | Query string | `brand_name:"Tylenol"` |
| `limit` | Results limit (max 1000) | `10` |
| `skip` | Number to skip | `0` |

### Searchable Fields
- `brand_name`
- `generic_name`
- `active_ingredient`
- `manufacturer_name`
- `product_ndc`
- `route`
- `dosage_form`

### Python Implementation

```python
import requests
from typing import List, Dict, Optional
from dataclasses import dataclass

@dataclass
class DrugLabel:
    brand_name: str
    generic_name: List[str]
    active_ingredients: List[str]
    manufacturer: str
    indications: str
    warnings: List[str]
    contraindications: str
    adverse_reactions: str
    drug_interactions: str
    dosage: str
    ndc: str

class FDADrugClient:
    """Client for FDA drug APIs."""
    
    BASE_URL = "https://api.fda.gov/drug"
    
    def search_labels(
        self,
        query: str,
        limit: int = 10
    ) -> List[Dict]:
        """Search drug labels."""
        url = f"{self.BASE_URL}/label.json"
        params = {
            "search": query,
            "limit": limit
        }
        
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        
        return response.json().get("results", [])
    
    def search_by_brand(self, brand: str, limit: int = 10) -> List[Dict]:
        """Search by brand name."""
        return self.search_labels(f'brand_name:"{brand}"', limit)
    
    def search_by_ingredient(self, ingredient: str, limit: int = 10) -> List[Dict]:
        """Search by active ingredient."""
        return self.search_labels(f'active_ingredient:"{ingredient}"', limit)
    
    def get_label(self, product_ndc: str) -> Optional[Dict]:
        """Get label for specific NDC."""
        results = self.search_labels(f'product_ndc:"{product_ndc}"', 1)
        return results[0] if results else None
    
    def get_warnings(self, drug_name: str) -> List[str]:
        """Get warnings for a drug."""
        labels = self.search_by_brand(drug_name, 5)
        warnings = []
        
        for label in labels:
            if label.get("warnings"):
                warnings.extend(label["warnings"])
        
        return warnings


# Usage
fda = FDADrugClient()

# Search for Metformin labels
labels = fda.search_by_brand("Metformin", limit=5)
print(f"Found {len(labels)} label(s)")

for label in labels[:2]:
    print(f"\nBrand: {label.get('brand_name', ['N/A'])}")
    print(f"Manufacturer: {label.get('manufacturer_name', ['N/A'])}")
    print(f"Indications: {label.get('indications_and_usage', ['N/A'])}")
```

---

## 2. FDA Drug Recalls API

### Endpoint
```
GET https://api.fda.gov/drug/enforcement.json
```

### Search Fields
- `product_description`
- `product_type`
- `reason_for_recall`
- `classification`
- `recalling_firm`
- `report_date`

### Recall Classifications
| Class | Meaning |
|-------|---------|
| Class I | Dangerous or defective product that could cause serious health problems |
| Class II | Product that might cause temporary health problems |
| Class III | Product unlikely to cause adverse health problems |

```python
def get_drug_recalls(
    drug_name: str = None,
    classification: str = None,
    limit: int = 20
) -> List[Dict]:
    """Get drug recall data."""
    url = "https://api.fda.gov/drug/enforcement.json"
    
    queries = []
    if drug_name:
        queries.append(f'product_description:"{drug_name}"')
    if classification:
        queries.append(f'classification:"{classification}"')
    
    search = " AND ".join(queries) if queries else "*"
    
    params = {"search": search, "limit": limit}
    response = requests.get(url, params=params, timeout=30)
    
    return response.json().get("results", [])


# Example: Get Class I recalls for pain relievers
recalls = get_drug_recalls(drug_name="Ibuprofen", classification="Class I")
```

---

## 3. FDA Adverse Events API

### Endpoint
```
GET https://api.fda.gov/drug/event.json
```

### Search Fields
- `patient.drug.openfda.brand_name`
- `patient.drug.openfda.generic_name`
- `patient.reaction.reaction_description`
- `outcome.description`

```python
def get_adverse_events(
    drug_name: str,
    limit: int = 100
) -> List[Dict]:
    """Get adverse event reports for a drug."""
    url = "https://api.fda.gov/drug/event.json"
    
    # Search by brand or generic name
    search = f'patient.drug.openfda.brand_name:"{drug_name}" OR patient.drug.openfda.generic_name:"{drug_name}"'
    
    params = {"search": search, "limit": limit}
    response = requests.get(url, params=params, timeout=30)
    
    return response.json().get("results", [])


def summarize_adverse_reactions(drug_name: str) -> Dict:
    """Summarize adverse reactions for a drug."""
    events = get_adverse_events(drug_name, limit=500)
    
    reactions = {}
    outcomes = {}
    
    for event in events:
        # Extract reactions
        for reaction in event.get("patient", {}).get("reaction", []):
            desc = reaction.get("reaction_description", "Unknown")
            reactions[desc] = reactions.get(desc, 0) + 1
        
        # Extract outcomes
        for outcome in event.get("patient", {}).get("outcome", []):
            desc = outcome.get("description", "Unknown")
            outcomes[desc] = outcomes.get(desc, 0) + 1
    
    return {
        "drug": drug_name,
        "total_reports": len(events),
        "top_reactions": sorted(reactions.items(), key=lambda x: -x[1])[:10],
        "outcomes": outcomes
    }
```

---

## 4. NDC Directory API

### Endpoint
```
GET https://api.fda.gov/drug/ndc.json
```

### Search Fields
- `product_ndc`
- `brand_name`
- `manufacturer_name`
- `active_ingredient`
- `dosage_form`

```python
def get_ndc_info(product_ndc: str) -> Dict:
    """Get NDC product information."""
    url = "https://api.fda.gov/drug/ndc.json"
    params = {"search": f'product_ndc:"{product_ndc}"', "limit": 1}
    
    response = requests.get(url, params=params, timeout=30)
    results = response.json().get("results", [])
    
    return results[0] if results else {}
```

---

## 5. Combined Drug Information Tool

```python
class ComprehensiveDrugLookup:
    """Combined drug information lookup."""
    
    def __init__(self):
        self.fda = FDADrugClient()
    
    def get_drug_info(self, drug_name: str) -> Dict:
        """Get comprehensive drug information."""
        result = {
            "search_term": drug_name,
            "labels": [],
            "recalls": [],
            "interactions_placeholder": "Use RxNorm API for interactions"
        }
        
        # Get FDA labels
        try:
            labels = self.fda.search_by_brand(drug_name, limit=3)
            result["labels"] = [
                {
                    "brand": l.get("brand_name", ["N/A"])[0],
                    "generic": l.get("generic_name", []),
                    "manufacturer": l.get("manufacturer_name", ["N/A"])[0],
                    "indications": l.get("indications_and_usage", ["N/A"]),
                    "warnings": l.get("warnings", [])[:3],
                    "contraindications": l.get("contraindications", ["N/A"])
                }
                for l in labels
            ]
        except Exception as e:
            result["labels_error"] = str(e)
        
        # Get recalls
        try:
            recalls = get_drug_recalls(drug_name, limit=5)
            result["recalls"] = [
                {
                    "date": r.get("report_date"),
                    "description": r.get("product_description", "")[:200],
                    "classification": r.get("classification"),
                    "reason": r.get("reason_for_recall", "")[:200]
                }
                for r in recalls
            ]
        except Exception as e:
            result["recalls_error"] = str(e)
        
        return result


# Usage
lookup = ComprehensiveDrugLookup()
info = lookup.get_drug_info("Aspirin")

print(f"Drug: {info['search_term']}")
print(f"\nLabels found: {len(info['labels'])}")
for label in info['labels']:
    print(f"  - {label['brand']} by {label['manufacturer']}")
    print(f"    Indications: {label['indications'][:100]}...")

print(f"\nRecalls: {len(info['recalls'])}")
for recall in info['recalls']:
    print(f"  - {recall['date']}: {recall['classification']}")
```

---

## Rate Limits

| API | Limit |
|-----|-------|
| openFDA | 1 request/second, 240 requests/minute |
| Recommended | Cache responses, batch requests |

---

## References

- openFDA API: https://open.fda.gov/apis/
- Drug Label API: https://api.fda.gov/drug/label.json
- Drug Recalls: https://api.fda.gov/drug/enforcement.json
- Adverse Events: https://api.fda.gov/drug/event.json