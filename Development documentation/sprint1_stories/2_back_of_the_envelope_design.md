## 2. Back-of-the-Envelope Design

```mermaid
graph TD
    A[DOCX File] --> B{Mammoth.js: DOCX to HTML}
    B --> C[HTML Stream]
    C --> D{Cheerio.js: HTML DOM Traversal}
    D --> E{Section Identification Logic}
    E --> F[Raw JSON Structure (Title, Sections, Content)]
    F --> G{Keyword Extraction (TF-IDF/Compromise)}
    G --> H[Keywords & Entities]
    H --> I{Taxonomy Mapping (Basic)}
    I --> J[Enriched ParsedCareer JSON]

    subgraph "Parser Module Structure (server/src/parser)"
        direction TB
        MainParser[index.ts: CareerParser class] --> UtilDocxToHtml[docxToHtml.ts]
        MainParser --> UtilHtmlToJson[htmlToJson.ts]
        MainParser --> UtilKeywordExtractor[keywordExtractor.ts]
        MainParser --> UtilTaxonomyMapper[taxonomyMapper.ts]
        
        UtilHtmlToJson --> UtilSectionDetector[utils/sectionDetector.ts]
        MainParser --> Types[types/index.ts: ParsedCareer, etc.]
    end
```