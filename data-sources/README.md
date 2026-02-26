# Data Sources Documentation

This folder contains detailed documentation for each medical data source.

---

## Index

| Document | Description |
|----------|-------------|
| [pubmed.md](./pubmed.md) | PubMed biomedical literature database |
| [rxnorm.md](./rxnorm.md) | RxNorm drug database (names, interactions) |
| [fda-drugs.md](./fda-drugs.md) | FDA drug labels, recalls, adverse events |
| [medlineplus.md](./medlineplus.md) | Consumer health information |
| [guidelines.md](./guidelines.md) | Clinical practice guidelines |

---

## Quick Reference

### Primary Data Sources

| Source | Type | Primary Use | API |
|--------|------|-------------|-----|
| **PubMed** | Literature | Research, citations | E-utilities |
| **RxNorm** | Drugs | Drug names, interactions | REST API |
| **FDA** | Drugs | Labels, recalls | openFDA |
| **MedlinePlus** | Consumer | Patient info | Web scraping |
| **ClinicalTrials.gov** | Trials | Clinical trial info | REST API |

### Source Priority (Evidence Level)

```
Level 1: Systematic Reviews, Meta-analyses
  └── PubMed (filtered by publication type)
  
Level 2: Clinical Guidelines
  └── guidelines (curated PDFs)
  
Level 3: Clinical Trials
  └── ClinicalTrials.gov
  
Level 4: Peer-reviewed Research
  └── PubMed
  
Level 5: Drug Information
  └── RxNorm, FDA, MedlinePlus
```

---

## Integration Priority

### Phase 1 (MVP)
1. **PubMed** - Primary literature source for RAG
2. **RxNorm** - Basic drug lookup

### Phase 2
3. **FDA** - Drug labels, warnings
4. **MedlinePlus** - Consumer-friendly explanations

### Phase 3
5. **ClinicalTrials.gov** - Trial information
6. **Guidelines** - Curated clinical guidelines

---

## Data Flow

```
                    ┌─────────────────┐
                    │   User Query    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   LLM Agent     │
                    │  (LangChain)    │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────▼────────┐  ┌──────▼──────┐  ┌─────────▼────────┐
│ Vector Store    │  │  Drug APIs  │  │  Literature API  │
│ (Chroma/Pinecone│  │  (RxNorm)   │  │   (PubMed)       │
│                 │  │  (FDA)      │  │                   │
└────────┬────────┘  └──────┬──────┘  └─────────┬────────┘
         │                  │                    │
         │      ┌───────────┴───────────┐        │
         │      │                       │        │
┌────────▼──────▼──────┐  ┌────────────▼────────┴────────┐
│ Medical Knowledge   │  │ Drug Information              │
│ - PubMed chunks     │  │ - Names, interactions         │
│ - Guidelines        │  │ - Labels, warnings            │
└─────────────────────┘  └────────────────────────────────┘
```

---

## Key Implementation Notes

1. **PubMed E-utilities**: Get free API key (10x rate limit)
2. **RxNorm**: No API key needed, respectful rate limiting
3. **FDA openFDA**: 1 req/sec, cache responses
4. **All sources**: Cache frequently accessed data

---

## See Also

- [Research Documents](../research/)
- [Architecture Documents](../architecture/)