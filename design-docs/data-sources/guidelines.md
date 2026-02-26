# Medical Guidelines Data Sources

## Overview

Clinical practice guidelines are systematically developed statements to assist practitioner decisions. They provide evidence-based recommendations for patient care.

---

## 1. National Guideline Clearinghouse (NGC)

### Overview
- **URL**: https://www.guidelines.gov/
- **Status**: Was discontinued in 2018, now archived
- **Alternative**: See resources below

---

## 2. ECRI Guidelines Trust

### Overview
- **URL**: https://guidelines.ecri.org/
- **Content**: Clinical practice guidelines
- **Access**: Free registration required

---

## 3. Specific Guideline Sources

### A. American Diabetes Association (ADA)
- **URL**: https://care.diabetesjournals.org/
- **Content**: Diabetes care guidelines
- **API**: None, web scraping required

### B. American Heart Association (AHA)
- **URL**: https://www.ahajournals.org/
- **Content**: Cardiology guidelines
- **API**: None

### C. CDC Guidelines
- **URL**: https://www.cdc.gov/mmwr/guidelines.html
- **Content**: CDC recommendations
- **Format**: Published articles

### D. WHO Guidelines
- **URL**: https://www.who.int/publications/i/guidelines
- **Content**: International health guidelines

---

## 4. Guidelines API (if available)

```python
class GuidelinesClient:
    """Client for medical guidelines."""
    
    def search_guidelines(
        self,
        condition: str,
        organization: str = None
    ) -> List[Dict]:
        """Search for clinical guidelines."""
        
        # This would need to be customized per source
        # Below is a conceptual implementation
        
        results = []
        
        # Search ECRI (example - may need auth)
        ecri_results = self._search_ecri(condition, organization)
        results.extend(ecri_results)
        
        # Search WHO
        who_results = self._search_who(condition)
        results.extend(who_results)
        
        return results
    
    def _search_ecri(self, condition: str, org: str = None) -> List[Dict]:
        """Search ECRI Guidelines Trust."""
        # Placeholder - requires API integration
        return []
    
    def _search_who(self, condition: str) -> List[Dict]:
        """Search WHO guidelines."""
        url = "https://www.who.int/publications/i/guidelines"
        
        # Web scraping implementation would go here
        return []
```

---

## 5. G-I-N (Guidelines International Network)

- **URL**: https://g-i-n.net/
- **Content**: International guideline network
- **Resources**: Guideline libraries from various countries

---

## 6. Specific Guideline Examples

### Diabetes Management Guidelines
```
Organization: American Diabetes Association
Title: Standards of Care in Diabetes - 2024
URL: https://care.diabetesjournals.org/content/47/Supplement_1
Key Topics:
- Diagnostic criteria
- Glycemic targets
- Pharmacologic approaches
- Complications screening
```

### Hypertension Guidelines
```
Organization: ACC/AHA
Title: 2017 ACC/AHA Guideline for the Prevention, Detection, Evaluation, and Management of High Blood Pressure in Adults
URL: https://www.ahajournals.org/doi/10.1161/HYP.0000000000000065
```

### COVID-19 Guidelines
```
Organization: NIH
Title: COVID-19 Treatment Guidelines
URL: https://www.covid19treatmentguidelines.nih.gov/
```

---

## 7. Integration Approach

### For Guidelines, Recommended Approach:

1. **Manual Curation**: Identify key guidelines for your domain
2. **PDF Processing**: Parse PDF guidelines into chunks
3. **Vector Store**: Index in your RAG system
4. **Priority**: Use as authoritative source

```python
# Example: Process guideline PDFs
from langchain_community.document_loaders import PyPDFLoader

def process_guideline(pdf_path: str, metadata: Dict) -> List[Document]:
    """Process a clinical guideline PDF."""
    
    loader = PyPDFLoader(pdf_path)
    pages = loader.load_and_split()
    
    # Add metadata
    for page in pages:
        page.metadata.update({
            "source_type": "guideline",
            "organization": metadata.get("organization"),
            "title": metadata.get("title"),
            "year": metadata.get("year"),
            "url": metadata.get("url")
        })
    
    return pages

# Usage
guideline_docs = process_guideline(
    "ada_guidelines_2024.pdf",
    {
        "organization": "American Diabetes Association",
        "title": "Standards of Care in Diabetes - 2024",
        "year": 2024,
        "url": "https://care.diabetesjournals.org/content/47/Supplement_1"
    }
)

# Add to vector store
vector_store.add_documents(guideline_docs)
```

---

## 8. Summary of Medical Guidelines Sources

| Source | URL | Access | Content |
|--------|-----|--------|---------|
| **ECRI** | guidelines.ecri.org | Registration | Guidelines database |
| **ADA** | care.diabetesjournals.org | Open | Diabetes |
| **AHA** | ahajournals.org | Some open | Cardiology |
| **CDC** | cdc.gov/mmwr | Open | Various |
| **WHO** | who.int/publications | Open | International |
| **NIH COVID** | covid19treatmentguidelines.nih.gov | Open | COVID-19 |

---

## Recommendation for Implementation

1. **Start with domain-specific guidelines** for your primary use case
2. **Parse PDFs** into structured chunks for RAG
3. **Add metadata**: Organization, year, version for citation
4. **Weight heavily**: Guidelines should be top-tier evidence sources