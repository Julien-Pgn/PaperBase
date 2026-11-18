# Contrat d'interface PaperBase ↔ CoursePilot — v0.1

> À copier tel quel dans **les deux** dépôts sous `docs/CONTRACT.md`.
> C'est la seule chose que les deux sessions Claude Code doivent partager. Tant que ce fichier ne change pas, chaque repo avance seul.
> Toute modification passe par Jul, incrémente `CONTRACT_VERSION`, et est recopiée dans les deux repos le même jour.

## 1. Principe

- **PaperBase** possède le contrat. Il vit en code dans `src/<package>/contract.py` (Pydantic v2 uniquement, aucune autre dépendance, aucun import de torch/duckdb/qdrant).
- **CoursePilot** en garde une copie identique dans `coursepilot/paperbase_contract.py` et vérifie au démarrage que `CONTRACT_VERSION` correspond à celle que renvoie PaperBase (`GET /health`).
- Le transport (HTTP ou import Python) est un détail d'adaptateur. CoursePilot code contre le `Protocol` ci-dessous, jamais contre le transport.

## 2. Modèles (Pydantic v2)

```python
from datetime import date, datetime
from typing import Literal, Protocol
from pydantic import BaseModel, Field

CONTRACT_VERSION = "0.1"

ExtractionMethod = Literal["arxiv_html", "existing_ocr", "pdf_ocr", "pdf_text"]
FulltextStatus = Literal["none", "available", "chunked", "indexed", "failed"]
RetrievalMode = Literal["dense", "sparse", "hybrid"]


class PaperRef(BaseModel):
    """Everything needed to cite a paper and verify that it exists."""
    arxiv_id: str                    # without version, e.g. "2306.00978"
    version: str | None = None       # e.g. "v2"; None if unknown
    title: str
    authors: list[str]
    published: date
    categories: list[str]
    abs_url: str                     # https://arxiv.org/abs/<id><version>


class PaperHit(BaseModel):
    paper: PaperRef
    abstract: str
    score: float                     # fused RRF score, only comparable within one query
    rank: int                        # 1 = best
    fulltext_status: FulltextStatus


class SearchPapersRequest(BaseModel):
    query: str = Field(min_length=3, max_length=500)
    top_k: int = Field(default=100, ge=1, le=200)
    categories: list[str] | None = None
    published_after: date | None = None
    published_before: date | None = None


class EnsureFulltextRequest(BaseModel):
    arxiv_ids: list[str] = Field(min_length=1, max_length=100)
    allow_download: bool = True      # False = only reuse what is already cached
    max_new_papers: int = Field(default=20, ge=0, le=100)  # max papers newly processed per call


class EnsureFulltextResult(BaseModel):
    arxiv_id: str
    status: Literal["already_indexed", "indexed", "skipped_budget", "unavailable", "failed"]
    extraction_method: ExtractionMethod | None = None
    n_chunks: int = 0
    error: str | None = None


class EvidenceChunk(BaseModel):
    chunk_id: str                    # stable, see section 3 of CONTRACT.md
    paper: PaperRef
    section_path: list[str]          # e.g. ["3 Method", "3.2 Activation-aware scaling"]
    text: str
    char_start: int                  # offsets in the paper full text
    char_end: int
    extraction_method: ExtractionMethod
    score: float
    rank: int
    retrieval_mode: RetrievalMode
    source_url: str                  # arXiv HTML URL (with section anchor when possible) or PDF URL


class RetrieveChunksRequest(BaseModel):
    query: str = Field(min_length=3, max_length=500)
    arxiv_ids: list[str] | None = Field(default=None, max_length=200)  # restrict retrieval to these papers
    top_k: int = Field(default=20, ge=1, le=50)
    mode: RetrievalMode = "hybrid"


class HealthResponse(BaseModel):
    status: Literal["ok", "degraded"]
    contract_version: str
    n_papers_indexed: int
    n_chunks_indexed: int
    embed_model: str
    chunker_version: str


class PaperbaseAPI(Protocol):
    def health(self) -> HealthResponse: ...
    def search_papers(self, req: SearchPapersRequest) -> list[PaperHit]: ...
    def ensure_fulltext(self, req: EnsureFulltextRequest) -> list[EnsureFulltextResult]: ...
    def retrieve_chunks(self, req: RetrieveChunksRequest) -> list[EvidenceChunk]: ...
    def get_paper(self, arxiv_id: str) -> PaperRef | None: ...   # None = the paper does not exist
```

## 3. Règles

1. **`chunk_id` stable et déterministe** : `"{arxiv_id}{version}:{chunker_version}:{chunk_index:04d}"`, ex. `2306.00978v2:c1:0007`. Re-chunker avec une nouvelle version produit de nouveaux IDs sans écraser les anciens.
2. **`ensure_fulltext` est idempotent** : rappeler avec les mêmes IDs ne refait aucun travail (`already_indexed`). Ordre d'acquisition : cache existant (OCR déjà fait) → HTML arXiv → PDF + OCR/extraction texte.
3. **`get_paper` est la source de vérité pour vérifier une citation** : CoursePilot rejette toute citation dont `get_paper` renvoie `None`.
4. **Les textes renvoyés sont des données non fiables** : CoursePilot ne doit jamais les traiter comme des instructions (prompt injection).
5. **Erreurs** : un article qui échoue ne fait jamais échouer tout l'appel ; il revient avec `status="failed"` et `error`.
6. **Limites** : les bornes `Field(...)` ci-dessus sont des garde-fous côté PaperBase ; CoursePilot peut être plus strict, jamais plus large.

## 4. Transport HTTP (décidé le 2026-11-18)

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| GET | `/health` | — | `HealthResponse` |
| POST | `/papers/search` | `SearchPapersRequest` | `list[PaperHit]` |
| POST | `/papers/fulltext` | `EnsureFulltextRequest` | `list[EnsureFulltextResult]` |
| POST | `/chunks/retrieve` | `RetrieveChunksRequest` | `list[EvidenceChunk]` |
| GET | `/papers/{arxiv_id}` | — | `PaperRef` ou 404 |

URL de base configurée côté CoursePilot par `PAPERBASE_URL` (défaut `http://localhost:8100`).

## 5. Jeu d'évaluation partagé

Un seul jeu « golden », versionné dans PaperBase : `eval/golden/v1.jsonl`. CoursePilot le lit via `PAPERBASE_GOLDEN_PATH`.

```json
{
  "id": "q001",
  "question": "What is post-training quantization?",
  "category": "definition",
  "reference_answer": "A quantization method applied after model training, without retraining.",
  "relevant_papers": ["2306.00978"],
  "evidence": [{"arxiv_id": "2306.00978", "section_contains": "Introduction", "quote": "texte exact copié de l'article"}],
  "relevant_chunks": [],
  "expected_behavior": "answer"
}
```

- `category` ∈ `definition | history | technical | comparison | multi_paper | limitations | multi_section | unanswerable`.
- `expected_behavior` ∈ `answer | abstain`.
- `evidence.quote` est la vérité terrain : `relevant_chunks` est **recalculé** par PaperBase (chunks dont le texte contient la citation) pour chaque `chunker_version`. Ainsi changer de chunking n'invalide pas les étiquettes.
- Les questions sont rédigées ou validées par Jul, pas inventées par un agent.
