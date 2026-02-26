# PubMed Data Source

## Overview

**PubMed** is the primary biomedical literature database maintained by the National Library of Medicine (NLM). It contains over 35 million citations and abstracts for biomedical literature.

- **URL**: https://pubmed.ncbi.nlm.nih.gov/
- **API**: NCBI E-utilities (https://eutils.ncbi.nlm.nih.gov/)
- **Free**: Yes
- **Update Frequency**: Daily

---

## Database Contents

### Types of Content
- **Journal Articles**: Peer-reviewed research
- **MeSH-indexed Articles**: Articles indexed with Medical Subject Headings
- **Preprints**: COVID-19 related preprints (selective)
- **Clinical Trials**: Links to ClinicalTrials.gov
- **Reviews**: Systematic reviews, meta-analyses
- **Editorials & Letters**: Opinion pieces

### Coverage
- **Date Range**: 1946 - Present
- **Languages**: 40+ languages (English abstracts for non-English)
- **Journals**: ~5,200 journals indexed

---

## Access Methods

### 1. E-utilities API (Recommended)

#### Search Endpoint
```
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi
```

Parameters:
| Parameter | Description | Example |
|-----------|-------------|---------|
| `db` | Database name | `pubmed` |
| `term` | Search query | `diabetes[MeSH Terms]` |
| `retmax` | Max results | `20` |
| `retstart` | Start index | `0` |
| `sort` | Sort order | `relevance`, `pub_date` |
| `api_key` | Optional API key | `your-key` |

#### Fetch Endpoint
```
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi
```

Parameters:
| Parameter | Description | Example |
|-----------|-------------|---------|
| `db` | Database | `pubmed` |
| `id` | PMID(s) | `123456` or `123,457` |
| `retmode` | Return format | `xml`, `json`, `text` |
| `rettype` | Record type | `abstract`, `medline` |

#### Summary Endpoint
```
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi
```

---

## Search Syntax

### MeSH Terms
```
diabetes mellitus[MeSH Terms]
```

### Field-Specific Search
```
cancer[ti]           # Title only
immunotherapy[ab]    # Abstract only
heart failure[tiab]  # Title or Abstract
```

### Boolean Operators
```
cancer AND immunotherapy
cancer OR tumor
cancer NOT benign
```

### Date Ranges
```
2020/01/01:2023/12/31[dp]
"last 5 years"[dp]
```

### Publication Types
```
review[pt]           # Review articles
meta-analysis[pt]    # Meta-analyses
clinical trial[pt]   # Clinical trials
randomized[pt]       # Randomized controlled trials
```

### Combining Filters
```
diabetes[mh] AND therapy[sh] AND (review[pt] OR meta-analysis[pt]) AND 2020:2023[dp]
```

---

## Python Implementation

```python
import requests
import time
import xml.etree.ElementTree as ET
from typing import List, Dict, Optional
from dataclasses import dataclass

@dataclass
class PubMedArticle:
    pmid: str
    title: str
    abstract: str
    authors: List[str]
    journal: str
    pub_date: str
    mesh_terms: List[str]
    publication_type: List[str]
    doi: Optional[str] = None

class PubMedClient:
    """Client for PubMed E-utilities API."""
    
    BASE_URL = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"
    
    def __init__(self, api_key: Optional[str] = None):
        self.api_key = api_key
        self.last_request = 0
        self.min_interval = 0.34 if api_key else 0.34
    
    def _rate_limit(self):
        """Respect API rate limits."""
        elapsed = time.time() - self.last_request
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        self.last_request = time.time()
    
    def search(
        self,
        query: str,
        max_results: int = 20,
        start: int = 0,
        sort: str = "relevance"
    ) -> List[str]:
        """Search PubMed and return PMIDs."""
        self._rate_limit()
        
        params = {
            "db": "pubmed",
            "term": query,
            "retmax": max_results,
            "retstart": start,
            "retmode": "json",
            "sort": sort
        }
        if self.api_key:
            params["api_key"] = self.api_key
        
        response = requests.get(
            f"{self.BASE_URL}/esearch.fcgi",
            params=params,
            timeout=30
        )
        response.raise_for_status()
        
        data = response.json()
        return data.get("esearchresult", {}).get("idlist", [])
    
    def fetch_articles(
        self,
        pmids: List[str],
        retmode: str = "xml"
    ) -> List[PubMedArticle]:
        """Fetch article details by PMIDs."""
        if not pmids:
            return []
        
        self._rate_limit()
        
        params = {
            "db": "pubmed",
            "id": ",".join(pmids),
            "retmode": retmode,
            "rettype": "medline"
        }
        if self.api_key:
            params["api_key"] = self.api_key
        
        response = requests.get(
            f"{self.base_url}/efetch.fcgi",
            params=params,
            timeout=30
        )
        
        return self._parse_medline(response.text)
    
    def get_summaries(self, pmids: List[str]) -> List[Dict]:
        """Get article summaries."""
        self._rate_limit()
        
        params = {
            "db": "pubmed",
            "id": ",".join(pmids),
            "retmode": "json"
        }
        if self.api_key:
            params["api_key"] = self.api_key
        
        response = requests.get(
            f"{self.BASE_URL}/esummary.fcgi",
            params=params,
            timeout=30
        )
        
        data = response.json()
        return data.get("result", {})

    def _parse_medline(self, text: str) -> List[PubMedArticle]:
        """Parse MEDLINE format response."""
        articles = []
        records = text.split("\n\n")
        
        for record in records:
            if not record.strip():
                continue
            
            article_data = {}
            mesh_terms = []
            authors = []
            pub_types = []
            
            for line in record.split("\n"):
                if line.startswith("TI  "):
                    article_data["title"] = line[4:].strip()
                elif line.startswith("AB  "):
                    article_data["abstract"] = line[4:].strip()
                elif line.startswith("FAU"):
                    authors.append(line[4:].strip())
                elif line.startswith("AU  "):
                    authors.append(line[4:].strip())
                elif line.startswith("JT  "):
                    article_data["journal"] = line[4:].strip()
                elif line.startswith("DP  "):
                    article_data["pub_date"] = line[4:].strip()
                elif line.startswith("MH  "):
                    mesh_terms.append(line[4:].strip())
                elif line.startswith("PT  "):
                    pub_types.append(line[4:].strip())
                elif line.startswith("AID"):
                    if "doi" in line.lower():
                        article_data["doi"] = line.split()[-1]
            
            if article_data.get("title"):
                articles.append(PubMedArticle(
                    pmid="",  # Would need to extract from record
                    title=article_data.get("title", ""),
                    abstract=article_data.get("abstract", ""),
                    authors=authors,
                    journal=article_data.get("journal", ""),
                    pub_date=article_data.get("pub_date", ""),
                    mesh_terms=mesh_terms,
                    publication_type=pub_types,
                    doi=article_data.get("doi")
                ))
        
        return articles


# Usage Example
client = PubMedClient(api_key="your-api-key")

# Search for diabetes treatment reviews
pmids = client.search(
    query="diabetes mellitus[mh] AND therapy[sh] AND review[pt]",
    max_results=10,
    sort="relevance"
)

# Fetch article details
articles = client.fetch_articles(pmids)

for article in articles:
    print(f"Title: {article.title}")
    print(f"Abstract: {article.abstract[:200]}...")
    print(f"MeSH Terms: {article.mesh_terms[:5]}")
```

---

## PubMed Central (PMC)

### Overview
- **URL**: https://www.ncbi.nlm.nih.gov/pmc/
- **Content**: Full-text open-access biomedical articles
- **Articles**: ~2.1 million

### PMC ID Format
- Format: `PMC` + number (e.g., `PMC1234567`)
- Can be converted from PMID using the ID Converter API

### Access Full Text
```python
def get_full_text(pmcid: str) -> str:
    """Fetch full text article from PMC."""
    url = f"https://www.ncbi.nlm.nih.gov/pmc/articles/{pmcid}/?format=flat"
    response = requests.get(url)
    return response.text
```

---

## Evidence Level Classification

When indexing PubMed articles, assign evidence levels:

| Level | Publication Type | Description |
|-------|------------------|-------------|
| 1 | Meta-analysis, Systematic Review | Highest evidence |
| 2 | Randomized Controlled Trial | Strong evidence |
| 3 | Controlled Trial (no randomization) | Moderate evidence |
| 4 | Cohort Study, Case-Control | Lower evidence |
| 5 | Case Report, Expert Opinion | Lowest evidence |

---

## Best Practices

1. **Use API Key**: Register for free API key (10x rate limit)
2. **Cache Results**: Cache frequent queries
3. **MeSH Terms**: Prefer MeSH over keywords for better recall
4. **Filter by Publication Type**: Target high-quality evidence
5. **Sort by Relevance**: For general queries
6. **Sort by Date**: For latest research

---

## Rate Limits

| Method | Without API Key | With API Key |
|--------|-----------------|--------------|
| Requests/sec | 3 | 10 |
| Requests/day | 100 | 1,000 |

---

## References

- PubMed: https://pubmed.ncbi.nlm.nih.gov/
- E-utilities: https://www.ncbi.nlm.nih.gov/books/NBK25497/
- MeSH: https://www.ncbi.nlm.nih.gov/mesh/
- PubMed Tutorial: https://www.nlm.nih.gov/bsd/disted/pubmedtutorial/