# Consumer Health Data Sources

## Overview

Consumer health data sources provide patient-friendly medical information. These are essential for:
- Patient-facing responses
- Explaining medical terms in plain language
- Providing trustworthy health information

---

## 1. MedlinePlus

### Overview
- **URL**: https://medlineplus.gov/
- **Content**: Consumer health information from NIH
- **Languages**: English, Spanish
- **Free**: Yes

### Content Types
- Disease and condition articles
- Drug and supplement information
- Nutrition information
- Mental health topics
- Exercise and fitness
- Medical tests encyclopedia

### Direct Link Formats
```
# Health topic
https://medlineplus.gov/diabetes.html

# Drug information
https://medlineplus.gov/druginfo/meds/a682235.html

# Medical tests
https://medlineplus.gov/lab-tests/blood-glucose-test/
```

### Python Implementation

```python
import requests
from bs4 import BeautifulSoup
from typing import Dict, List, Optional

class MedlinePlusClient:
    """Client for MedlinePlus consumer health information."""
    
    BASE_URL = "https://medlineplus.gov"
    
    def get_topic(self, topic: str) -> Dict:
        """Get MedlinePlus topic page."""
        url = f"{self.BASE_URL}/{topic.lower().replace(' ', '')}.html"
        
        response = requests.get(url, timeout=30)
        response.raise_for_status()
        
        soup = BeautifulSoup(response.text, 'html.parser')
        
        # Extract main content
        main_content = soup.find('main', {'id': 'main-content'})
        
        if not main_content:
            return {"error": "Topic not found", "url": url}
        
        # Extract sections
        sections = []
        for section in main_content.find_all('section'):
            heading = section.find('h2')
            if heading:
                content = section.get_text(separator=' ', strip=True)
                sections.append({
                    "heading": heading.get_text(strip=True),
                    "content": content
                })
        
        return {
            "topic": topic,
            "url": url,
            "sections": sections
        }
    
    def get_drug_info(self, drug_name: str) -> Dict:
        """Get drug information from MedlinePlus."""
        # Search for drug (simplified)
        url = f"{self.BASE_URL}/druginfo/meds/{drug_name.lower().replace(' ', '')}.html"
        
        try:
            response = requests.get(url, timeout=30)
            if response.status_code == 404:
                return {"error": "Drug not found"}
            
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # Extract drug information
            main = soup.find('main', {'id': 'main-content'})
            
            if not main:
                return {"error": "Could not parse drug page"}
            
            content = {
                "name": drug_name,
                "url": url,
                "sections": []
            }
            
            for section in main.find_all('section'):
                heading = section.find('h2')
                if heading:
                    content["sections"].append({
                        "heading": heading.get_text(strip=True),
                        "content": section.get_text(separator=' ', strip=True)
                    })
            
            return content
            
        except Exception as e:
            return {"error": str(e)}
    
    def search_topics(self, query: str) -> List[Dict]:
        """Search for MedlinePlus topics."""
        # Use their API if available, or scrape search results
        url = f"{self.BASE_URL}/websearch"
        params = {"pq": query}
        
        response = requests.get(url, params=params, timeout=30)
        soup = BeautifulSoup(response.text, 'html.parser')
        
        results = []
        for result in soup.find_all('li', class_='results-list_item')[:10]:
            link = result.find('a')
            if link:
                results.append({
                    "title": link.get_text(strip=True),
                    "url": self.BASE_URL + link.get('href', '')
                })
        
        return results


# Usage
medline = MedlinePlusClient()

# Get diabetes topic
diabetes = medline.get_topic("diabetes")
print(f"Topic: {diabetes.get('topic')}")
print(f"Sections: {len(diabetes.get('sections', []))}")

# Get drug info
aspirin = medline.get_drug_info("aspirin")
if "error" not in aspirin:
    print(f"Drug: {aspirin.get('name')}")
    for section in aspirin.get("sections", [])[:3]:
        print(f"  {section['heading']}")
```

---

## 2. NIH MedlinePlus Connect

### Overview
- **Service**: Links EHRs to MedlinePlus
- **Use**: Get patient-friendly info from codes

```python
def get_medlineplus_from_code(code: str, code_type: str) -> Dict:
    """
    Get MedlinePlus links from medical codes.
    
    code_type: ICD9, ICD10, CPT, SNOMED
    """
    url = "https://apps.nlm.nih.gov/medlineplus/services/mpconnectservice.cfm"
    
    params = {
        "code": code,
        "context": code_type
    }
    
    response = requests.get(url, params=params, timeout=30)
    
    # Parse XML response
    # Return related links
    return {"code": code, "type": code_type, "response": response.text}
```

---

## 3. ClinicalTrials.gov API

### Overview
- **URL**: https://clinicaltrials.gov/
- **API**: https://clinicaltrials.gov/api/api_home
- **Content**: Clinical trial information
- **Free**: Yes

### API Endpoints

```
# Search studies
GET https://clinicaltrials.gov/api/v2/studies?query.cond={condition}&pageSize=20

# Get study details
GET https://clinicaltrials.gov/api/v2/studies/{nct_id}

# Search with interventions
GET https://clinicaltrials.gov/api/v2/studies?query.cond={condition}&query.intervention={drug}
```

### Python Implementation

```python
import requests
from typing import List, Dict, Optional
from dataclasses import dataclass

@dataclass
class ClinicalTrial:
    nct_id: str
    title: str
    status: str
    conditions: List[str]
    interventions: List[str]
    sponsor: str
    phase: str
    summary: str

class ClinicalTrialsClient:
    """Client for ClinicalTrials.gov API."""
    
    BASE_URL = "https://clinicaltrials.gov/api/v2"
    
    def search_studies(
        self,
        condition: str = None,
        intervention: str = None,
        sponsor: str = None,
        status: str = None,
        phase: str = None,
        page_size: int = 20
    ) -> List[Dict]:
        """Search for clinical trials."""
        
        params = {"pageSize": page_size}
        
        # Build query
        if condition:
            params["query.cond"] = condition
        if intervention:
            params["query.intervention"] = intervention
        if sponsor:
            params["query.spons"] = sponsor
        
        if status:
            params["filter.overallStatus"] = status
        if phase:
            params["filter.phase"] = phase
        
        response = requests.get(
            f"{self.BASE_URL}/studies",
            params=params,
            timeout=30
        )
        
        data = response.json()
        return data.get("studies", [])
    
    def get_study(self, nct_id: str) -> Optional[Dict]:
        """Get detailed study information."""
        response = requests.get(
            f"{self.BASE_URL}/studies/{nct_id}",
            timeout=30
        )
        
        if response.status_code == 404:
            return None
        
        return response.json()
    
    def format_study_summary(self, study: Dict) -> str:
        """Format study as readable summary."""
        protocol = study.get("protocolSection", {})
        
        title = protocol.get("identificationModule", {}).get("briefTitle", "N/A")
        nct_id = protocol.get("identificationModule", {}).get("nctId", "N/A")
        status = protocol.get("statusModule", {}).get("overallStatus", "N/A")
        
        conditions = protocol.get("conditionsModule", {}).get("conditions", [])
        interventions = protocol.get("armsInterventionsModule", {}).get("interventions", [])
        
        summary = protocol.get("descriptionModule", {}).get("briefSummary", "N/A")
        
        return f"""
Clinical Trial: {title}
NCT ID: {nct_id}
Status: {status}
Conditions: {', '.join(conditions)}
Interventions: {', '.join([i.get("name") for i in interventions])}
Summary: {summary}
        """.strip()


# Usage
trials = ClinicalTrialsClient()

# Search for diabetes drug trials
studies = trials.search_studies(
    condition="Type 2 Diabetes",
    intervention="Metformin",
    status="RECRUITING",
    page_size=5
)

print(f"Found {len(studies)} trials")

for study in studies[:3]:
    print(trials.format_study_summary(study))
```

---

## 4. Mayo Clinic (Reference)

### Overview
- **URL**: https://www.mayoclinic.org/diseases-conditions
- **Content**: Disease and condition information
- **Note**: Use web scraping or official API if available

---

## 5. UpToDate (Reference)

- **URL**: https://www.uptodate.com/
- **Content**: Evidence-based clinical information
- **Note**: Requires subscription, not for direct integration

---

## Summary: Consumer Health Data Sources

| Source | Type | Best For | Access |
|--------|------|----------|--------|
| **MedlinePlus** | Consumer health | Patient education | Web scraping |
| **ClinicalTrials.gov** | Trials | Trial information | Official API |
| **Mayo Clinic** | Reference | Disease info | Web scraping |

---

## Integration Strategy

1. **Patient queries**: Use MedlinePlus for plain-language explanations
2. **Clinical context**: Use PubMed for research-quality info
3. **Treatment options**: Use ClinicalTrials.gov for trials
4. **Drug info**: Combine RxNorm + FDA + MedlinePlus

```python
# Example: Combine sources for patient-friendly drug info
def get_patient_drug_info(drug_name: str) -> Dict:
    """Get comprehensive patient-friendly drug information."""
    
    # Get RxNorm for basic info
    rxcui = rxnorm_client.get_rxnorm_id(drug_name)
    
    # Get FDA label
    fda_labels = fda_client.search_by_brand(drug_name)
    
    # Get MedlinePlus
    medline = medline_client.get_drug_info(drug_name)
    
    # Combine into patient-friendly response
    return {
        "drug_name": drug_name,
        "rxnorm_id": rxcui,
        "fda_info": extract_key_fda_info(fda_labels),
        "patient_info": medline
    }
```