# Sprint 1: Word Wizard

## 1. Narrative Goal

In this sprint, we're building the heart of our application: the DOCX parser module. This critical component will transform career description documents into structured data that powers our entire application. By the end of these four days, we'll have a robust, reliable parser that can intelligently extract sections like "Career Snapshot," "Role Overview," and "Skills Needed" from Word documents, even when formatting varies. We'll implement a pipeline that converts DOCX files to HTML, walks the DOM to identify sections (including handling unmatched sections gracefully), and generates a clean JSON structure. We'll also add keyword extraction and basic tagging capabilities to make careers searchable. This sprint is about creating the "magic" that transforms unstructured documents into valuable, structured career information, laying the groundwork for more advanced semantic understanding in later stages.

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

## 3. Task Checklist

### Parser Module Setup

- [ ] Create parser module directory structure within `server/src/`:
  ```bash
  # (Assuming already in server directory from Sprint 0)
  mkdir -p src/parser src/parser/utils src/parser/__tests__/fixtures src/parser/types
  ```

- [ ] Install parser-specific dependencies in the `server/` directory:
  ```bash
  npm install mammoth cheerio compromise natural slugify
  npm install -D @types/cheerio @types/natural # (if available for natural version used)
  ```
  *   `mammoth`: DOCX to HTML conversion.
  *   `cheerio`: Server-side jQuery-like HTML parsing and traversal.
  *   `compromise`: Natural Language Processing (NLP) for entity extraction.
  *   `natural`: NLP tools, including TF-IDF and tokenizers.
  *   `slugify`: For generating URL-friendly slugs (re-confirming from Sprint 0).

- [ ] Define core parser types in `server/src/parser/types/index.ts`:
  ```typescript
  // server/src/parser/types/index.ts
  export interface CareerSection {
    title: string;
    content: string;
    displayOrder: number;
    section_type?: string; // e.g., 'standard', 'MISCELLANEOUS', or specific like 'CAREER_SNAPSHOT'
  }

  export interface CareerTag {
    name: string;
    category: string; // e.g., 'skill', 'technology', 'industry'
    relevanceScore: number;
  }

  export interface ParsedCareer {
    title: string;
    slug: string;
    snapshot: string; // Derived from a specific section or early content
    overview: string; // Content before the first recognized section or from a specific "Role Overview"
    sections: CareerSection[];
    tags: CareerTag[];
    rawHtml?: string; // Optional: for debugging the HTML output from Mammoth
    quality_score?: number; // Optional: heuristic score for parsing quality (0.0-1.0)
    parsing_notes?: string[]; // Optional: notes/warnings from the parsing process
    template_version?: string; // Optional: detected version of the DOCX template
  }

  export interface ParserOptions {
    extractRawHtml?: boolean;
    // Mappings for different DOCX template versions/styles
    sectionMappings?: Record<string, string>; // e.g., { "Job Summary": "CAREER_SNAPSHOT" }
    templateVersion?: string;
    versionSpecificMappings?: Record<string, Record<string, string>>;
    defaultSectionTitle?: string; // Title for unmatched sections
    taxonomyPath?: string; // Path to a custom taxonomy JSON file
  }
  ```

### DOCX to HTML Conversion

- [ ] Create `server/src/parser/docxToHtml.ts`:
  ```typescript
  // server/src/parser/docxToHtml.ts
  import * as mammoth from 'mammoth';
  import * as fs from 'fs/promises'; // Using promise-based fs
  import logger from '../../utils/logger'; // Assuming logger from Suggestion 14

  export async function convertDocxToHtml(filePath: string): Promise<string> {
    try {
      await fs.access(filePath); // Check if file exists and is accessible
      const result = await mammoth.convertToHtml({ path: filePath });
      const html = result.value;
      if (result.messages && result.messages.length > 0) {
        logger.warn('Mammoth.js conversion messages:', { filePath, messages: result.messages });
      }
      return html;
    } catch (error) {
      logger.error('Error converting DOCX to HTML from file:', { filePath, error });
      throw new Error(`Failed to convert DOCX to HTML from ${filePath}: ${(error as Error).message}`);
    }
  }

  export async function convertDocxBufferToHtml(buffer: Buffer): Promise<string> {
    try {
      const result = await mammoth.convertToHtml({ buffer });
      const html = result.value;
      if (result.messages && result.messages.length > 0) {
        logger.warn('Mammoth.js buffer conversion messages:', { messages: result.messages });
      }
      return html;
    } catch (error) {
      logger.error('Error converting DOCX to HTML from buffer:', { error });
      throw new Error(`Failed to convert DOCX buffer to HTML: ${(error as Error).message}`);
    }
  }
  ```

### HTML to Structured JSON Conversion

- [ ] Create `server/src/parser/utils/sectionDetector.ts`:
  ```typescript
  // server/src/parser/utils/sectionDetector.ts
  import { Element as CheerioElement } from 'cheerio'; // Use CheerioElement type

  // Standardized internal section keys
  export const STANDARD_SECTION_KEYS = {
    CAREER_SNAPSHOT: 'CAREER_SNAPSHOT',
    ROLE_OVERVIEW: 'ROLE_OVERVIEW',
    SKILLS_NEEDED: 'SKILLS_NEEDED',
    EDUCATION_REQUIREMENTS: 'EDUCATION_REQUIREMENTS',
    CAREER_PATH_GROWTH: 'CAREER_PATH_GROWTH',
    INDUSTRY_DEMAND: 'INDUSTRY_DEMAND',
    SALARY_RANGE: 'SALARY_RANGE',
    RELATED_CAREERS: 'RELATED_CAREERS',
    MISCELLANEOUS: 'MISCELLANEOUS', // For unmatched sections
  } as const; // Use 'as const' for stricter typing of keys

  export type StandardSectionKey = typeof STANDARD_SECTION_KEYS[keyof typeof STANDARD_SECTION_KEYS];

  // Default mappings of common DOCX headings to our standard keys
  export const DEFAULT_SECTION_MAPPINGS: Record<string, StandardSectionKey> = {
    'career snapshot': STANDARD_SECTION_KEYS.CAREER_SNAPSHOT,
    'job snapshot': STANDARD_SECTION_KEYS.CAREER_SNAPSHOT,
    'position snapshot': STANDARD_SECTION_KEYS.CAREER_SNAPSHOT,
    'summary': STANDARD_SECTION_KEYS.CAREER_SNAPSHOT,
    'role overview': STANDARD_SECTION_KEYS.ROLE_OVERVIEW,
    'job overview': STANDARD_SECTION_KEYS.ROLE_OVERVIEW,
    'position overview': STANDARD_SECTION_KEYS.ROLE_OVERVIEW,
    'job description': STANDARD_SECTION_KEYS.ROLE_OVERVIEW,
    'skills needed': STANDARD_SECTION_KEYS.SKILLS_NEEDED,
    'required skills': STANDARD_SECTION_KEYS.SKILLS_NEEDED,
    'key skills': STANDARD_SECTION_KEYS.SKILLS_NEEDED,
    'skills and qualifications': STANDARD_SECTION_KEYS.SKILLS_NEEDED,
    'education & requirements': STANDARD_SECTION_KEYS.EDUCATION_REQUIREMENTS,
    'education and requirements': STANDARD_SECTION_KEYS.EDUCATION_REQUIREMENTS,
    'education': STANDARD_SECTION_KEYS.EDUCATION_REQUIREMENTS,
    'qualifications': STANDARD_SECTION_KEYS.EDUCATION_REQUIREMENTS,
    'requirements': STANDARD_SECTION_KEYS.EDUCATION_REQUIREMENTS,
    'career path & growth': STANDARD_SECTION_KEYS.CAREER_PATH_GROWTH,
    'career path and growth': STANDARD_SECTION_KEYS.CAREER_PATH_GROWTH,
    'career progression': STANDARD_SECTION_KEYS.CAREER_PATH_GROWTH,
    'growth opportunities': STANDARD_SECTION_KEYS.CAREER_PATH_GROWTH,
    'industry demand': STANDARD_SECTION_KEYS.INDUSTRY_DEMAND,
    'job outlook': STANDARD_SECTION_KEYS.INDUSTRY_DEMAND,
    'salary range': STANDARD_SECTION_KEYS.SALARY_RANGE,
    'compensation': STANDARD_SECTION_KEYS.SALARY_RANGE,
    'related careers': STANDARD_SECTION_KEYS.RELATED_CAREERS,
    'similar roles': STANDARD_SECTION_KEYS.RELATED_CAREERS,
  };

  // Helper to normalize text for matching
  const normalizeText = (text: string): string => text.toLowerCase().trim().replace(/&/g, 'and');

  export function mapHeadingToStandardSection(
    headingText: string,
    customMappings: Record<string, string> = {} // Allow overriding defaults
  ): StandardSectionKey | null {
    const normalizedHeading = normalizeText(headingText);
    const combinedMappings = { ...DEFAULT_SECTION_MAPPINGS, ...customMappings };

    for (const [pattern, standardKey] of Object.entries(combinedMappings)) {
      if (normalizedHeading.includes(normalizeText(pattern))) {
        return standardKey as StandardSectionKey;
      }
    }
    return null; // No standard mapping found
  }

  // Heuristic to identify if an element is a main career title
  export function isCareerTitleCandidate(element: CheerioElement, $: cheerio.CheerioAPI): boolean {
    const tagName = element.tagName?.toLowerCase();
    const text = $(element).text().trim();
    if (!text) return false;

    // H1 is a strong candidate
    if (tagName === 'h1') return true;
    // First strong P tag with few words
    if (tagName === 'p' && $(element).find('strong').length > 0 && text.split(/\s+/).length <= 7) {
        // Check if it's the first significant paragraph
        let isFirstSignificantP = true;
        let prev = element.prevSibling;
        while(prev) {
            if (prev.type === 'tag' && $(prev).text().trim()) {
                isFirstSignificantP = false;
                break;
            }
            prev = prev.prevSibling;
        }
        if(isFirstSignificantP) return true;
    }
    return false;
  }

  // Heuristic to identify if an element is a section heading
  export function isSectionHeadingCandidate(element: CheerioElement, $: cheerio.CheerioAPI): boolean {
    const tagName = element.tagName?.toLowerCase();
    const text = $(element).text().trim();
    if (!text) return false;

    if (tagName?.match(/^h[2-6]$/)) return true; // h2-h6
    if (tagName === 'p' && ($(element).find('strong').length > 0 || $(element).find('b').length > 0) ) {
        // Check if the text looks like a heading (e.g., title case, ends with colon, short)
        if (text.split(/\s+/).length < 10 && text === text.toUpperCase() || /^[A-Z][a-z]+(?:\s+[A-Z][a-z]+)*:?$/.test(text)) {
            return true;
        }
    }
    return false;
  }
  ```

- [ ] Create `server/src/parser/htmlToJson.ts`:
  ```typescript
  // server/src/parser/htmlToJson.ts
  import * as cheerio from 'cheerio';
  import slugify from 'slugify';
  import { ParsedCareer, CareerSection, ParserOptions } from './types';
  import {
    mapHeadingToStandardSection,
    isCareerTitleCandidate,
    isSectionHeadingCandidate,
    STANDARD_SECTION_KEYS,
    StandardSectionKey
  } from './utils/sectionDetector';
  import logger from '../../utils/logger';

  export function convertHtmlToStructuredJson(
    html: string,
    options?: ParserOptions
  ): ParsedCareer {
    const $ = cheerio.load(html);
    const parsingNotes: string[] = [];

    const career: ParsedCareer = {
      title: '',
      slug: '',
      snapshot: '',
      overview: '',
      sections: [],
      tags: [], // Tags will be added by a subsequent step
      rawHtml: options?.extractRawHtml ? html : undefined,
      parsing_notes: parsingNotes,
    };

    const bodyChildren = $('body').children().toArray();
    let careerTitleFound = false;

    // Attempt to find career title first
    for (let i = 0; i < bodyChildren.length; i++) {
      const element = bodyChildren[i];
      if ($(element).text().trim() && isCareerTitleCandidate(element, $)) {
        career.title = $(element).text().trim();
        career.slug = slugify(career.title, { lower: true, strict: true, trim: true });
        careerTitleFound = true;
        // Remove the title element so it's not processed as a section or overview
        $(element).remove();
        break;
      }
    }

    if (!careerTitleFound) {
      career.title = 'Untitled Career Document'; // Fallback title
      career.slug = 'untitled-career-document';
      parsingNotes.push('Could not confidently identify a main career title.');
    }
    
    // Re-fetch body children after potential removal
    const contentElements = $('body').children().toArray();
    let currentSection: CareerSection | null = null;
    let currentSectionKey: StandardSectionKey | null = null;
    let displayOrder = 0;
    let contentBuffer = ''; // To accumulate content for sections

    contentElements.forEach((element, index) => {
      const rawText = $(element).text().trim();
      if (!rawText && element.tagName !== 'ul' && element.tagName !== 'ol') return; // Skip empty elements, but keep lists

      let elementHtml = '';
      // For lists, preserve structure better by getting HTML
      if (element.tagName === 'ul' || element.tagName === 'ol') {
        elementHtml = $(element).html() || ''; // Get inner HTML of lists
        // Convert list items to plain text with hyphens/numbers
        const listItems = $(element).find('li').map((_, li) => `- ${$(li).text().trim()}`).get();
        contentBuffer += listItems.join('\n') + '\n\n';
      } else {
        contentBuffer += rawText + '\n\n';
      }

      if (isSectionHeadingCandidate(element, $)) {
        const headingText = rawText;
        const mappedKey = mapHeadingToStandardSection(headingText, options?.sectionMappings);

        // If a new heading is found, save the previous section (if any)
        if (currentSection) {
          currentSection.content = currentSection.content.trim();
          if (currentSection.content) {
            career.sections.push(currentSection);
          }
          currentSection = null;
        }
         // If content buffer has data before this new section, it's part of overview or previous section
        if (contentBuffer.trim() && !currentSection) {
             if (!career.overview && !career.sections.length) {
                career.overview += contentBuffer.trim() + '\n\n';
            }
            // Reset buffer after assigning to overview or if it was part of a committed section
            contentBuffer = ''; 
        }


        if (mappedKey) {
          currentSectionKey = mappedKey;
          currentSection = {
            title: headingText, // Use original heading text
            content: '',
            displayOrder: displayOrder++,
            section_type: mappedKey,
          };
        } else {
          // Unmatched heading, treat as miscellaneous or part of previous content
          parsingNotes.push(`Unmapped heading: "${headingText}". Treating as standard content or new 'Miscellaneous' section.`);
           currentSectionKey = STANDARD_SECTION_KEYS.MISCELLANEOUS;
           currentSection = {
                title: headingText, // Keep original heading
                content: '',
                displayOrder: displayOrder++,
                section_type: STANDARD_SECTION_KEYS.MISCELLANEOUS,
            };
        }
        contentBuffer = ''; // Reset buffer for new section's content
      } else if (currentSection) {
        // Add content to the current section
        currentSection.content += contentBuffer;
        contentBuffer = ''; // Clear buffer as it's consumed
      } else {
        // Content before any recognized section is part of the overview
        if (!careerTitleFound || index < 2) { // Heuristic: early content is likely overview
            career.overview += contentBuffer;
            contentBuffer = ''; // Clear buffer
        }
      }
    });

    // Add any remaining content in currentSection or contentBuffer
    if (currentSection) {
      if(contentBuffer.trim()) currentSection.content += contentBuffer.trim();
      currentSection.content = currentSection.content.trim();
      if (currentSection.content) {
        career.sections.push(currentSection);
      }
    } else if (contentBuffer.trim()) {
        // If there's leftover content and no current section, append to overview or a new misc section
        if (!career.overview) {
             career.overview = contentBuffer.trim();
        } else {
            career.sections.push({
                title: options?.defaultSectionTitle || 'Additional Information',
                content: contentBuffer.trim(),
                displayOrder: displayOrder++,
                section_type: STANDARD_SECTION_KEYS.MISCELLANEOUS
            });
            parsingNotes.push('Orphaned content appended to Additional Information.');
        }
    }
    
    career.overview = career.overview.trim();

    // Derive snapshot:
    const snapshotSection = career.sections.find(s => s.section_type === STANDARD_SECTION_KEYS.CAREER_SNAPSHOT);
    if (snapshotSection?.content) {
      career.snapshot = snapshotSection.content;
    } else if (career.overview) {
      // Use first paragraph of overview as fallback snapshot
      career.snapshot = career.overview.split('\n\n')[0].substring(0, 255); // Limit length
      if (!snapshotSection) parsingNotes.push('Snapshot derived from overview.');
    } else {
        parsingNotes.push('Could not determine career snapshot.');
    }

    // Clean up section contents
    career.sections.forEach(sec => sec.content = sec.content.trim());

    return career;
  }
  ```

### Keyword Extraction and Basic Tagging

- [ ] Create `server/src/parser/keywordExtractor.ts`:
  ```typescript
  // server/src/parser/keywordExtractor.ts
  import natural from 'natural';
  import { ParsedCareer, CareerTag } from './types';
  import nlp from 'compromise'; // compromise is commonjs, so default import
  import logger from '../../utils/logger';

  export class TfIdfKeywordExtractor {
    private tfidf: natural.TfIdf;
    private domainStopwords: Set<string>;

    constructor() {
      this.tfidf = new natural.TfIdf();
      // Common English stopwords + custom domain-specific ones
      this.domainStopwords = new Set([
        ...natural.stopwords,
        'career', 'job', 'role', 'position', 'work', 'skills', 'experience',
        'year', 'years', 'month', 'months', 'day', 'days', 'required', 'preferred',
        'company', 'team', 'inc', 'llc', 'corp', 'eg', 'etc', 'related', 'overview',
        'snapshot', 'salary', 'education', 'requirements', 'demand', 'growth',
        'various', 'responsibilities', 'duties', 'qualification', 'degree', 'field',
        'including', 'such', 'ensure', 'provide', 'develop', 'maintain', 'manage',
        'analyze', 'design', 'implement', 'communicate', 'collaborate', 'report', 'model'
      ]);
    }

    public extractTopKeywords(careerData: ParsedCareer, maxKeywords: number = 10): CareerTag[] {
      let combinedText = careerData.title + '. ' + careerData.snapshot + '. ' + careerData.overview;
      careerData.sections.forEach(section => {
        // Weight skills section more heavily if present
        const weight = section.title.toLowerCase().includes('skill') ? 3 : 1;
        for (let i = 0; i < weight; i++) {
          combinedText += '. ' + section.title + '. ' + section.content;
        }
      });

      const tokenizer = new natural.WordPunctTokenizer();
      const tokens = tokenizer.tokenize(combinedText.toLowerCase()) || [];
      
      const filteredTokens = tokens.filter(token =>
        token.length > 2 &&           // Min word length
        !this.domainStopwords.has(token) && // Not a stopword
        !/^\d+$/.test(token) &&         // Not a number
        /^[a-zA-Z\-]+$/.test(token)     // Contains only letters or hyphens
      );

      if (filteredTokens.length === 0) {
        logger.warn('No suitable tokens found for TF-IDF analysis.', { title: careerData.title });
        return [];
      }
      
      // Reset TF-IDF for each document to ensure scores are relative to this document only
      this.tfidf = new natural.TfIdf(); 
      this.tfidf.addDocument(filteredTokens.join(' ')); // Add as a single string document

      const keywords: CareerTag[] = [];
      this.tfidf.listTerms(0 /* document index */)
        .slice(0, maxKeywords)
        .forEach(item => {
          if (item.term && item.tfidf > 0.1) { // Basic relevance threshold
            keywords.push({
              name: item.term,
              category: 'keyword', // Default category, to be refined by taxonomy
              relevanceScore: parseFloat(item.tfidf.toFixed(4)),
            });
          }
        });
      return keywords;
    }
  }

  export function extractEntitiesWithCompromise(careerData: ParsedCareer): CareerTag[] {
    let combinedText = careerData.title + '. ' + careerData.snapshot + '. ' + careerData.overview;
    careerData.sections.forEach(section => {
      combinedText += '. ' + section.title + '. ' + section.content;
    });

    const doc = nlp(combinedText);
    const entities: CareerTag[] = [];

    // Extract known technologies (example list, can be expanded)
    const techTerms = [
      'Python', 'R', 'SQL', 'Java', 'JavaScript', 'TypeScript', 'C#', 'C++', 'Go', 'Ruby', 'PHP', 'Swift', 'Kotlin',
      'React', 'Angular', 'Vue.js', 'Node.js', 'Express.js', 'Django', 'Flask', 'Spring Boot', '.NET',
      'TensorFlow', 'PyTorch', 'scikit-learn', 'Keras', 'Pandas', 'NumPy', 'Spark', 'Hadoop',
      'AWS', 'Azure', 'Google Cloud', 'GCP', 'Docker', 'Kubernetes', 'Git', 'Jenkins', 'CI/CD',
      'Tableau', 'Power BI', 'Excel', 'Salesforce', 'SAP', 'Oracle', 'MongoDB', 'PostgreSQL', 'MySQL'
    ];

    techTerms.forEach(term => {
      if (doc.has(term)) {
        entities.push({ name: term, category: 'technology', relevanceScore: 0.9 });
      }
    });

    doc.nouns().isSingular().out('array').forEach((noun: string) => {
       if (noun.length > 3 && !techTerms.map(t => t.toLowerCase()).includes(noun.toLowerCase())) { // Avoid double-adding tech
          // Further check if it's a plausible skill/tool/concept
          // This is a very basic heuristic
          if (/^[A-Z]/.test(noun) || noun.includes('-')) { // Proper nouns or hyphenated terms
            entities.push({ name: noun, category: 'concept', relevanceScore: 0.6 });
          }
       }
    });
    
    doc.organizations().out('array').forEach((org: string) => {
      entities.push({ name: org, category: 'organization', relevanceScore: 0.7 });
    });

    // Deduplicate (simple name-based deduplication for now)
    const seen = new Set();
    return entities.filter(e => {
        const duplicate = seen.has(e.name.toLowerCase());
        seen.add(e.name.toLowerCase());
        return !duplicate;
    });
  }
  ```

- [ ] Create `server/src/parser/taxonomyMapper.ts`:
  ```typescript
  // server/src/parser/taxonomyMapper.ts
  import * as fs from 'fs';
  import * as path from 'path';
  import { CareerTag } from './types';
  import logger from '../../utils/logger';

  interface TaxonomyTerm {
    canonical: string; // The preferred term for this category
    variations: string[]; // Includes the canonical term and its synonyms/variations
  }

  interface TaxonomyCategoryDefinition {
    categoryName: string;
    terms: TaxonomyTerm[];
  }

  // Example simplified taxonomy structure
  // This could be loaded from a JSON file for more flexibility
  const DEFAULT_TAXONOMY: TaxonomyCategoryDefinition[] = [
    {
      categoryName: 'skill',
      terms: [
        { canonical: 'Programming', variations: ['programming', 'coding', 'software development', 'scripting'] },
        { canonical: 'Data Analysis', variations: ['data analysis', 'data analytics', 'statistical analysis'] },
        { canonical: 'Project Management', variations: ['project management', 'pm', 'project coordination'] },
      ],
    },
    {
      categoryName: 'technology',
      terms: [
        { canonical: 'Python', variations: ['python', 'py'] },
        { canonical: 'JavaScript', variations: ['javascript', 'js', 'ecmascript'] },
        { canonical: 'SQL', variations: ['sql', 'structured query language'] },
        { canonical: 'AWS', variations: ['aws', 'amazon web services'] },
      ],
    },
    {
      categoryName: 'industry',
      terms: [
        { canonical: 'Technology', variations: ['technology', 'tech', 'it', 'software'] },
        { canonical: 'Healthcare', variations: ['healthcare', 'medical', 'health'] },
        { canonical: 'Finance', variations: ['finance', 'financial', 'banking'] },
      ],
    },
  ];

  export class TaxonomyMapper {
    private taxonomy: TaxonomyCategoryDefinition[];

    constructor(taxonomyPath?: string) {
      if (taxonomyPath && fs.existsSync(taxonomyPath)) {
        try {
          const taxonomyData = fs.readFileSync(taxonomyPath, 'utf8');
          this.taxonomy = JSON.parse(taxonomyData);
          logger.info(`Custom taxonomy loaded from ${taxonomyPath}`);
        } catch (error) {
          logger.warn(`Failed to load custom taxonomy from ${taxonomyPath}, using default. Error: ${error}`);
          this.taxonomy = DEFAULT_TAXONOMY;
        }
      } else {
        this.taxonomy = DEFAULT_TAXONOMY;
      }
    }

    public mapTags(tags: CareerTag[]): CareerTag[] {
      const mappedTags: CareerTag[] = [];
      const addedCanonicalTags = new Set<string>(); // To avoid adding the same canonical tag multiple times

      tags.forEach(tag => {
        let matched = false;
        for (const categoryDef of this.taxonomy) {
          for (const termDef of categoryDef.terms) {
            if (termDef.variations.map(v => v.toLowerCase()).includes(tag.name.toLowerCase())) {
              // Map to the canonical term and category
              const canonicalKey = `${categoryDef.categoryName}:${termDef.canonical}`;
              if (!addedCanonicalTags.has(canonicalKey)) {
                mappedTags.push({
                  name: termDef.canonical,
                  category: categoryDef.categoryName,
                  relevanceScore: tag.relevanceScore, // Preserve original relevance or adjust
                });
                addedCanonicalTags.add(canonicalKey);
              }
              matched = true;
              break; // Found a match for this tag in this category
            }
          }
          if (matched) break; // Found a match for this tag, move to next tag
        }
        // If no specific mapping found, keep the original tag with its original category (e.g., 'keyword')
        if (!matched) {
          const originalKey = `${tag.category}:${tag.name}`;
           if (!addedCanonicalTags.has(originalKey)) { // Check if it wasn't already added by a more specific mapping
            mappedTags.push(tag);
            addedCanonicalTags.add(originalKey);
           }
        }
      });
      return mappedTags;
    }
  }
  ```

### Main Parser Module

- [ ] Create `server/src/parser/index.ts`:
  ```typescript
  // server/src/parser/index.ts
  import { convertDocxToHtml, convertDocxBufferToHtml } from './docxToHtml';
  import { convertHtmlToStructuredJson } from './htmlToJson';
  import { TfIdfKeywordExtractor, extractEntitiesWithCompromise } from './keywordExtractor';
  import { TaxonomyMapper } from './taxonomyMapper';
  import { ParsedCareer, ParserOptions, CareerTag } from './types';
  import logger from '../../utils/logger';

  export { ParsedCareer, CareerSection, CareerTag, ParserOptions }; // Re-export types

  export class CareerParser {
    private keywordExtractor: TfIdfKeywordExtractor;
    private taxonomyMapper: TaxonomyMapper;
    private options: ParserOptions;

    constructor(options: ParserOptions = {}) {
      this.keywordExtractor = new TfIdfKeywordExtractor();
      this.taxonomyMapper = new TaxonomyMapper(options.taxonomyPath);
      this.options = options;
    }

    private async baseParse(htmlProvider: () => Promise<string>): Promise<ParsedCareer> {
      const html = await htmlProvider();
      const parsedData = convertHtmlToStructuredJson(html, this.options);

      // Extract keywords using TF-IDF
      const tfidfKeywords = this.keywordExtractor.extractTopKeywords(parsedData, 15);
      
      // Extract entities using Compromise
      const compromiseEntities = extractEntitiesWithCompromise(parsedData);
      
      // Combine and de-duplicate tags before mapping
      const allExtractedTags: CareerTag[] = [];
      const seenTagNames = new Set<string>();

      [...tfidfKeywords, ...compromiseEntities].forEach(tag => {
        const lowerName = tag.name.toLowerCase();
        if (!seenTagNames.has(lowerName)) {
          allExtractedTags.push(tag);
          seenTagNames.add(lowerName);
        } else {
            // If duplicate, potentially update relevance score if one is higher
            const existingTag = allExtractedTags.find(et => et.name.toLowerCase() === lowerName);
            if (existingTag && tag.relevanceScore > existingTag.relevanceScore) {
                existingTag.relevanceScore = tag.relevanceScore;
            }
        }
      });

      // Map to taxonomy
      parsedData.tags = this.taxonomyMapper.mapTags(allExtractedTags);
      
      // Sort tags by relevance score (descending)
      parsedData.tags.sort((a, b) => b.relevanceScore - a.relevanceScore);


      if (!this.options.extractRawHtml) {
        delete parsedData.rawHtml;
      }

      // Basic quality scoring (placeholder - refine in ImportService or as a dedicated step)
      let score = 0.5;
      if (parsedData.title !== 'Untitled Career Document') score += 0.1;
      if (parsedData.snapshot) score += 0.1;
      if (parsedData.overview) score += 0.1;
      if (parsedData.sections.length > 2) score += 0.1;
      if (parsedData.tags.length > 3) score += 0.1;
      parsedData.quality_score = parseFloat(Math.min(1.0, score).toFixed(2));
      
      if (parsedData.quality_score < 0.6) {
          parsedData.parsing_notes?.push(`Low quality score (${parsedData.quality_score}) suggests review needed.`);
      }

      logger.info(`Parsed career: ${parsedData.title}, Quality: ${parsedData.quality_score}`);
      return parsedData;
    }

    public async parseFile(filePath: string): Promise<ParsedCareer> {
      logger.info(`Starting parsing for file: ${filePath}`);
      return this.baseParse(() => convertDocxToHtml(filePath));
    }

    public async parseBuffer(buffer: Buffer): Promise<ParsedCareer> {
      logger.info('Starting parsing from buffer');
      return this.baseParse(() => convertDocxBufferToHtml(buffer));
    }
  }
  ```

### CLI and Watch Script (for testing parser)

- [ ] Create `server/src/parser/cli.ts` for manual parsing:
  ```typescript
  // server/src/parser/cli.ts
  import * as fs from 'fs';
  import * as path from 'path';
  import { CareerParser } from './index'; // Use the main CareerParser
  import logger from '../../utils/logger'; // Use the main logger

  async function main() {
    const args = process.argv.slice(2);
    if (args.length < 1) {
      logger.error('Usage: ts-node src/parser/cli.ts <docx-file-path> [output-json-path]');
      process.exit(1);
    }

    const docxFilePath = args[0];
    const outputJsonPath = args[1] || path.join(
      path.dirname(docxFilePath),
      `${path.basename(docxFilePath, path.extname(docxFilePath))}.parsed.json`
    );

    try {
      const parser = new CareerParser({ extractRawHtml: true }); // Enable raw HTML for CLI debug
      const parsedCareer = await parser.parseFile(docxFilePath);

      fs.writeFileSync(outputJsonPath, JSON.stringify(parsedCareer, null, 2));
      logger.info(`Successfully parsed "${parsedCareer.title}" and saved to ${outputJsonPath}`);
      logger.info(`  Sections: ${parsedCareer.sections.length}, Tags: ${parsedCareer.tags.length}, Quality: ${parsedCareer.quality_score}`);
      if(parsedCareer.parsing_notes && parsedCareer.parsing_notes.length > 0) {
        logger.warn('  Parsing Notes:', parsedCareer.parsing_notes);
      }

    } catch (error) {
      logger.error('CLI parsing failed:', { error });
      process.exit(1);
    }
  }

  main();
  ```

- [ ] Create `server/src/parser/watch.ts` for auto-parsing files in `docs/` (optional, for dev convenience):
  ```typescript
  // server/src/parser/watch.ts
  import * as fs from 'fs';
  import * as path from 'path';
  import { CareerParser } from './index';
  import chokidar from 'chokidar'; // More robust file watcher
  import logger from '../../utils/logger';

  const DOCS_INPUT_DIR = path.resolve(__dirname, '../../../docs'); // Original DOCX files
  const PARSED_OUTPUT_DIR = path.resolve(__dirname, '../../../docs/parsed'); // Parsed JSON output

  if (!fs.existsSync(PARSED_OUTPUT_DIR)) {
    fs.mkdirSync(PARSED_OUTPUT_DIR, { recursive: true });
  }

  const parser = new CareerParser();

  async function processFile(filePath: string) {
    if (!filePath.toLowerCase().endsWith('.docx')) return;
    logger.info(`File event detected: ${filePath}`);

    try {
      const parsedCareer = await parser.parseFile(filePath);
      const outputFileName = `${path.basename(filePath, path.extname(filePath))}.json`;
      const outputPath = path.join(PARSED_OUTPUT_DIR, outputFileName);
      
      fs.writeFileSync(outputPath, JSON.stringify(parsedCareer, null, 2));
      logger.info(`Parsed "${parsedCareer.title}" from ${path.basename(filePath)} -> ${outputFileName}`);
    } catch (error) {
      logger.error(`Error parsing ${path.basename(filePath)}:`, { error });
    }
  }

  // Initialize watcher.
  const watcher = chokidar.watch(DOCS_INPUT_DIR, {
    ignored: /(^|[\/\\])\../, // ignore dotfiles
    persistent: true,
    ignoreInitial: false, // Process existing files on start
    depth: 0, // Only watch top-level files in DOCS_INPUT_DIR
    awaitWriteFinish: { // Wait for writes to complete
        stabilityThreshold: 2000,
        pollInterval: 100
    }
  });
  
  logger.info(`Watching for .docx files in ${DOCS_INPUT_DIR}...`);
  logger.info(`Parsed JSON will be saved to ${PARSED_OUTPUT_DIR}`);

  watcher
    .on('add', filePath => processFile(filePath))
    .on('change', filePath => processFile(filePath)) // Re-parse on change
    .on('error', error => logger.error('Watcher error:', { error }));

  // Install chokidar: npm install chokidar @types/chokidar
  // Ensure chokidar is added as a dev dependency if not already
  ```
  *Note: Added `chokidar` for more robust file watching. Install it: `npm install chokidar` and `npm install -D @types/chokidar` in `server/`.*

- [ ] Add/update scripts in `server/package.json`:
  ```json
  "scripts": {
    // ... existing scripts
    "parse": "ts-node src/parser/cli.ts",
    "parse:watch": "ts-node src/parser/watch.ts"
  },
  ```

### Unit Tests for Parser Logic

- [ ] Create a sample DOCX file in `server/src/parser/__tests__/fixtures/SampleCareer.docx` (use content from `sample_career_docx.md`).
- [ ] Write tests for `docxToHtml.ts` (`server/src/parser/__tests__/docxToHtml.test.ts`):
    *   Test successful conversion of the sample DOCX.
    *   Test handling of non-existent files.
- [ ] Write tests for `htmlToJson.ts` (`server/src/parser/__tests__/htmlToJson.test.ts`):
    *   Use HTML output from the sample DOCX (or a manually crafted HTML fixture).
    *   Test title extraction.
    *   Test section extraction and mapping (e.g., "Role Overview", "Skills Needed").
    *   Test snapshot derivation.
    *   Test handling of documents with slight variations in headings.
    *   Test handling of documents with unmatched headings (should go to "Miscellaneous" or similar).
- [ ] Write tests for `keywordExtractor.ts` (`server/src/parser/__tests__/keywordExtractor.test.ts`):
    *   Test TF-IDF keyword extraction with sample `ParsedCareer` data.
    *   Test Compromise entity extraction.
- [ ] Write tests for `taxonomyMapper.ts` (`server/src/parser/__tests__/taxonomyMapper.test.ts`):
    *   Test mapping of known keywords/entities to taxonomy categories.
    *   Test handling of unmapped tags.
- [ ] Write tests for the main `CareerParser` class (`server/src/parser/__tests__/parser.test.ts`):
    *   Test `parseFile` and `parseBuffer` methods, ensuring the full pipeline works.
    *   Verify structure of the final `ParsedCareer` object.

## 4. Creative Add-Ons (Stretch)

*   [ ] **Section Confidence Scoring (Refined):** In `htmlToJson.ts`, when mapping sections, calculate a confidence score (0.0-1.0) based on how closely the found heading matches known patterns (e.g., exact match vs. fuzzy match, presence of strong formatting). Store this in `CareerSection.confidenceScore` and factor it into the overall `ParsedCareer.quality_score`.
*   [ ] **Fuzzy Section Title Matching:** Implement fuzzy string matching (e.g., using `fuse.js` or `fuzzyset.js`) in `sectionDetector.ts` to better handle variations in section headings if simple `includes` isn't robust enough. Install: `npm install fuse.js` or `npm install fuzzyset.js`.
*   [ ] **Synonym Expansion using WordNet JSON (Advanced):** In `TaxonomyMapper` or `KeywordExtractor`, load a WordNet JSON file. When a keyword is extracted, find its synonyms. If a synonym maps to a canonical term in your taxonomy, boost the relevance of that canonical term. This requires a pre-processed WordNet JSON.
*   [ ] **Advanced Template Version Detection:** Implement more sophisticated logic in `CareerParser` to detect different DOCX template versions based on structural cues or embedded metadata (if controllable), and apply different `ParserOptions` accordingly.

## 5. Run & Verify

### Build & Test Commands

1.  Ensure `chokidar` is installed in `server/`: `npm install chokidar && npm install -D @types/chokidar`
2.  Build the server (compiles TypeScript):
    ```bash
    cd server
    npm run build
    ```
3.  Run parser tests:
    ```bash
    npm test -- --testPathPattern=parser # Runs tests specifically in the parser directory
    # Or, to run all server tests:
    # npm test
    ```
    *   **In VS Code:** Use the Test Explorer to run tests in `server/src/parser/__tests__`.

### Manual Parser Testing (CLI)

1.  Place a sample `.docx` file (e.g., `DataScientist.docx` based on `sample_career_docx.md`) into the project's root `docs/` directory.
2.  Run the parser CLI tool from the `server/` directory:
    ```bash
    # Example:
    npm run parse ../docs/DataScientist.docx ../docs/DataScientist.parsed.json
    # Or just:
    # npm run parse ../docs/DataScientist.docx (output will be DataScientist.parsed.json in ../docs/)
    ```
3.  Inspect the generated `DataScientist.parsed.json` file in the `docs/` directory.
    *   Verify correct title, slug.
    *   Verify sections are identified with correct titles and content.
    *   Verify snapshot and overview are populated.
    *   Verify tags are generated with categories and relevance scores.
    *   Check `parsing_notes` and `quality_score`.

### Watch Script (Optional Development Utility)

1.  (In a separate terminal) Start the watch script from the `server/` directory:
    ```bash
    npm run parse:watch
    ```
2.  Add or modify `.docx` files in the root `docs/` directory.
3.  Observe the console output from the watch script.
4.  Check for new/updated `.json` files in `docs/parsed/`.

### Success Criteria

- ✅ All parser unit tests pass with ≥ 85% line coverage for the parser module.
- ✅ CLI tool (`npm run parse`) successfully converts a sample DOCX to a structured JSON file.
- ✅ The generated JSON correctly identifies career title, slug, snapshot, overview, and individual sections with their content.
- ✅ Keywords and entities are extracted and mapped to basic tags with categories and relevance scores.
- ✅ Parser handles variations in common section headings (as defined in `sectionDetector.ts`).
- ✅ Unmatched sections are handled gracefully (e.g., collected under a "Miscellaneous" title or noted in `parsing_notes`).
- ✅ (Optional) `parse:watch` script detects changes and re-parses DOCX files in the `docs/` directory.

## 6. Artifacts to Commit

- **Branch name**: `sprint-1-word-wizard`
- **Mandatory files**:
    - `server/src/parser/` directory and all its contents (`index.ts`, `docxToHtml.ts`, `htmlToJson.ts`, `keywordExtractor.ts`, `taxonomyMapper.ts`, `utils/sectionDetector.ts`, `types/index.ts`).
    - `server/src/parser/__tests__/` directory with all test files and fixtures.
    - Updated `server/package.json` and `server/package-lock.json` with new dependencies.
    - (If applicable) Sample `.docx` file used for testing, e.g., in `docs/` or `server/src/parser/__tests__/fixtures/`.
- **PR Checklist**:
    - [ ] Parser module converts DOCX files to structured JSON as per `ParsedCareer` type.
    - [ ] Section detection correctly identifies standard sections and handles unknown ones.
    - [ ] Keyword/entity extraction and basic taxonomy mapping are functional.
    - [ ] CLI tool for manual parsing works.
    - [ ] Unit tests for parser components achieve target coverage and pass.
    - [ ] Code is well-commented, especially complex parsing logic and heuristics.
    - [ ] Logger (from Suggestion 14 / Sprint 0) is used for informative output and error reporting within the parser.
