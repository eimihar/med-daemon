# PubMed/NCBI API Research

## E-utilities API Endpoints

| Endpoint | Purpose | URL Example |
|----------|---------|-------------|
| `esearch` | Search PubMed | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=diabetes` |
| `efetch` | Retrieve records | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=pubmed&id=123456&retmode=xml` |
| `esummary` | Get summary | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi?db=pubmed&id=123456` |
| `elink` | Find related | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/elink.fcgi?dbfrom=pubmed&id=123456` |

## PubMed Search Capabilities

### MeSH Terms
Medical Subject Headings provide standardized indexing:
- `MeSH Terms`: Broad medical concepts
- `Subheadings`: Specific aspects (therapy, diagnosis, complications)

Examples:
- `diabetes mellitus[MeSH Terms]`
- `diabetes mellitus/therapy[MeSH Subheading]`

### Field Tags

| Tag | Field | Example |
|-----|-------|---------|
| `[ti]` | Title | `cancer[ti]` |
| `[ab]` | Abstract | `immunotherapy[ab]` |
| `[tiab]` | Title/Abstract | `heart failure[tiab]` |
| `[au]` | Author | `smith j[au]` |
| `[dp]` | Publication Date | `2020/01/01:2023/12/31[dp]` |
| `[pt]` | Publication Type | `review[pt]` |
| `[mh]` | MeSH Terms | `diabetes mellitus[mh]` |

### Boolean Operators
- `AND`: Both terms must appear
- `OR`: Either term can appear
- `NOT`: Exclude term

### Proximity Search
```pubmed
"heart disease"[tiab:~2]  # terms within 2 words
```

## Rate Limits

| API Key | Requests/second |
|---------|-----------------|
| No key | 3 requests/second |
| With API key | 10 requests/second |

**Register for API key**: https://www.ncbi.nlm.nih.gov/account/

## Python Implementation

```python
import requests
import time
from typing import List, Dict

class PubMedClient:
    def __init__(self, api_key: str = None):
        self.base_url = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"
        self.api_key = api_key
        self.last_request_time = 0
        self.min_interval = 0.34 if api_key else 0.34  # Rate limiting
    
    def _rate_limit(self):
        elapsed = time.time() - self.last_request_time
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        self.last_request_time = time.time()
    
    def search(self, term: str, max_results: int = 20) -> List[str]:
        """Search PubMed and return PMIDs."""
        self._rate_limit()
        
        params = {
            "db": "pubmed",
            "term": term,
            "retmax": max_results,
            "retmode": "json",
            "sort": "relevance"
        }
        if self.api_key:
            params["api_key"] = self.api_key
        
        response = requests.get(f"{self.base_url}/esearch.fcgi", params=params)
        data = response.json()
        
        return data.get("esearchresult", {}).get("idlist", [])
    
    def fetch_abstracts(self, pmids: List[str]) -> List[Dict]:
        """Fetch abstracts for given PMIDs."""
        self._rate_limit()
        
        params = {
            "db": "pubmed",
            "id": ",".join(pmids),
            "retmode": "xml",
            "rettype": "abstract"
        }
        if self.api_key:
            params["api_key"] = self.api_key
        
        response = requests.get(f"{self.base_url}/efetch.fcgi", params=params)
        # Parse XML response (use xml.etree.ElementTree)
        return self._parse_abstracts(response.text)
    
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
        
        response = requests.get(f"{self.base_url}/esummary.fcgi", params=params)
        return response.json()
```

## PubMed Central (PMC)

### Access Methods
- **API**: https://www.ncbi.nlm.nih.gov/pmc/tools/idconverter/
- **Bulk Download**: FTP at `ftp://ftp.ncbi.nlm.nih.gov/pub/pmc/`
- **Format**: XML, JSON, PDF

### PMC ID Conversion
```python
# Convert PMID to PMCID
https://www.ncbi.nlm.nih.gov/pmc/utils/idconv/v1.0/?tool=my_tool&email=my_email&ids=123456
```

## Search Strategy for Medical Queries

1. **Start with MeSH**: Use standardized terms for better recall
2. **Add subheadings**: Narrow to specific aspects (therapy, diagnosis)
3. **Filter by publication type**: Clinical trials, reviews, meta-analyses
4. **Sort by relevance**: Use Best Match for general queries
5. **Date limits**: Recent research for up-to-date information

### Example: Find Diabetes Treatment Reviews
```
diabetes mellitus[MeSH Terms] AND (therapy[MeSH Subheading] OR treatment[MeSH Subheading]) AND review[pt] AND ("2018/01/01"[DP] : "3000"[DP])
```