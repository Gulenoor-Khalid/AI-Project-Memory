# Requirements Document: Construction Document Converter

## Introduction

The Construction Document Converter adapts the existing book-to-skill extraction pipeline to process construction-specific documents (specifications, manuals, safety guides, blueprints, compliance documents, material datasheets, and building codes) into reusable Claude Code skills optimized for on-site reference, compliance checking, safety procedures, and material specifications.

Unlike technical books which follow a linear chapter structure with prose and frameworks, construction documents have diverse formats (hierarchical specs, procedural safety guides, tabular material data, visual blueprints, regulatory checklists) and require domain-specific extraction patterns. This converter reuses the existing extraction infrastructure while introducing construction-specific knowledge organization, terminology handling, and use-case optimization.

---

## Glossary

- **Construction Document**: Any technical or regulatory document used in construction workflows (specifications, manuals, safety guides, blueprints, compliance documents, material datasheets, building codes, standards)
- **Document Type**: The category of construction document (e.g., Specification, Safety Manual, Material Datasheet, Building Code, Compliance Checklist, Blueprint, Installation Guide)
- **Extraction Pipeline**: The existing book-to-skill infrastructure (PDF/EPUB parsing, text extraction, file generation)
- **Skill**: A reusable Claude Code knowledge base organized into on-demand reference files
- **Section**: A logical division within a construction document (e.g., "Section 3.2 — Concrete Specifications")
- **Procedure**: A step-by-step sequence for performing a construction task (e.g., safety protocol, installation steps)
- **Specification**: A detailed technical requirement or standard (e.g., material grade, dimension tolerance, performance criteria)
- **Compliance Requirement**: A regulatory or code-based mandate that must be satisfied (e.g., OSHA safety rule, building code section)
- **Material Property**: A quantified characteristic of a construction material (e.g., tensile strength, fire rating, cost per unit)
- **On-Site Reference**: Knowledge accessed by workers or supervisors in the field during active construction
- **Compliance Checking**: Verification that work meets regulatory, code, or specification requirements
- **Safety Procedure**: A documented sequence for performing a task safely, including hazards and mitigations
- **Blueprint**: A visual representation of a building or structure (floor plans, elevations, sections, details)
- **Extraction Mode**: The strategy used to parse a document type (e.g., hierarchical for specs, procedural for safety guides, tabular for datasheets)

---

## Requirements

### Requirement 1: Support Multiple Construction Document Types

**User Story:** As a construction manager, I want to convert different types of construction documents (specifications, safety manuals, material datasheets, building codes, compliance checklists, installation guides) into skills, so that I can organize diverse knowledge sources into a unified reference system.

#### Acceptance Criteria

1. WHEN a user provides a construction document, THE Converter SHALL identify the document type (Specification, Safety Manual, Material Datasheet, Building Code, Compliance Checklist, Installation Guide, Blueprint, or Other)
2. WHEN the document type cannot be automatically detected, THE Converter SHALL ask the user to specify the type before proceeding
3. THE Converter SHALL support extraction from PDF and EPUB formats for all document types
4. WHEN a document is identified as a Specification, THE Converter SHALL extract hierarchical sections, subsections, and technical requirements
5. WHEN a document is identified as a Safety Manual, THE Converter SHALL extract procedures, hazards, mitigations, and compliance references
6. WHEN a document is identified as a Material Datasheet, THE Converter SHALL extract material properties, performance data, and usage guidelines
7. WHEN a document is identified as a Building Code, THE Converter SHALL extract regulatory sections, compliance criteria, and cross-references
8. WHEN a document is identified as a Compliance Checklist, THE Converter SHALL extract checklist items, verification criteria, and sign-off requirements
9. WHEN a document is identified as an Installation Guide, THE Converter SHALL extract step-by-step procedures, tools required, and troubleshooting steps

### Requirement 2: Extract Construction-Specific Knowledge Structures

**User Story:** As a site supervisor, I want the extracted skill to organize knowledge in construction-relevant ways (procedures, specifications, compliance rules, material properties), so that I can quickly find what I need on-site without reading through prose.

#### Acceptance Criteria

1. WHEN extracting a Specification document, THE Converter SHALL organize content into hierarchical sections (e.g., Section 3.2.1 — Concrete Strength Requirements)
2. WHEN extracting a Safety Manual, THE Converter SHALL identify and structure procedures as: Procedure Name → Hazards → Steps → Mitigations → Compliance References
3. WHEN extracting a Material Datasheet, THE Converter SHALL identify and structure material properties as: Property Name → Value → Unit → Tolerance → Application Context
4. WHEN extracting a Building Code, THE Converter SHALL identify and structure compliance requirements as: Code Section → Requirement → Applicability → Penalties for Non-Compliance
5. WHEN extracting a Compliance Checklist, THE Converter SHALL identify and structure checklist items as: Item → Verification Criteria → Responsible Party → Sign-Off Requirements
6. WHEN extracting an Installation Guide, THE Converter SHALL identify and structure procedures as: Task Name → Prerequisites → Tools Required → Steps → Verification → Troubleshooting
7. THE Converter SHALL extract and preserve all quantified specifications (dimensions, tolerances, performance thresholds, material grades, cost data)
8. THE Converter SHALL extract and preserve all regulatory references (code sections, standard numbers, compliance citations)
9. THE Converter SHALL extract and preserve all visual or tabular data (comparison matrices, decision tables, material selection guides, performance charts)

### Requirement 3: Create Construction-Optimized Skill Output Structure

**User Story:** As a field worker, I want the skill to be organized for quick on-site reference with minimal scrolling, so that I can find answers while working without losing focus.

#### Acceptance Criteria

1. THE Converter SHALL generate a master SKILL.md file containing: document overview, quick-reference procedures, compliance checklist, and section index
2. THE Converter SHALL generate section files (one per major section) containing: section title, specifications, procedures, compliance requirements, and cross-references
3. THE Converter SHALL generate a procedures.md file containing: all step-by-step procedures with hazards, mitigations, and verification steps
4. THE Converter SHALL generate a specifications.md file containing: all quantified requirements, tolerances, material grades, and performance criteria
5. THE Converter SHALL generate a compliance.md file containing: all regulatory requirements, code references, and compliance verification criteria
6. THE Converter SHALL generate a materials.md file containing: all material properties, performance data, selection criteria, and cost information
7. THE Converter SHALL generate a glossary.md file containing: all construction-specific terms, abbreviations, and acronyms with definitions
8. THE Converter SHALL generate a quick-reference.md file containing: decision tables, material selection matrices, and common troubleshooting guides
9. WHEN a document contains visual elements (blueprints, diagrams, charts), THE Converter SHALL extract descriptions and reference information, noting that visual details are in the original document

### Requirement 4: Support Construction-Specific Use Cases

**User Story:** As a compliance officer, I want to use the skill to verify that work meets code requirements, so that I can quickly check compliance without manually searching through building codes.

#### Acceptance Criteria

1. WHEN a user asks about compliance, THE Skill SHALL retrieve relevant code sections, requirements, and verification criteria from the compliance.md file
2. WHEN a user asks about a procedure, THE Skill SHALL retrieve step-by-step instructions, hazards, and mitigations from the procedures.md file
3. WHEN a user asks about material specifications, THE Skill SHALL retrieve material properties, performance data, and selection criteria from the materials.md file
4. WHEN a user asks about a specific section or requirement, THE Skill SHALL retrieve the relevant section file with full context
5. WHEN a user asks "what are the compliance requirements for X?", THE Skill SHALL search the compliance.md file and return applicable code sections and verification criteria
6. WHEN a user asks "how do I perform X safely?", THE Skill SHALL search the procedures.md file and return step-by-step instructions with hazards and mitigations
7. WHEN a user asks "what material should I use for X?", THE Skill SHALL search the materials.md file and return material options with properties and selection criteria
8. WHEN a user asks "what does section X say?", THE Skill SHALL retrieve the relevant section file and provide the full specification

### Requirement 5: Adapt Extraction Pipeline for Construction Documents

**User Story:** As a tool developer, I want to reuse the existing extraction infrastructure while adding construction-specific parsing, so that I can minimize code duplication and leverage proven extraction methods.

#### Acceptance Criteria

1. THE Converter SHALL reuse the existing PDF/EPUB extraction methods (pdftotext, PyPDF2, pdfminer, Docling, ebooklib, zipfile)
2. THE Converter SHALL reuse the existing file generation and skill directory structure
3. THE Converter SHALL add construction-specific document type detection (analyzing document structure, terminology, and formatting)
4. THE Converter SHALL add construction-specific section extraction (identifying hierarchical sections, procedures, specifications, compliance requirements)
5. THE Converter SHALL add construction-specific knowledge organization (grouping by procedure, specification, compliance, material)
6. THE Converter SHALL add construction-specific file generation (procedures.md, specifications.md, compliance.md, materials.md, quick-reference.md)
7. WHEN a document type is identified as technical (e.g., Material Datasheet with tables), THE Converter SHALL use Docling extraction mode to preserve tables and structured data
8. WHEN a document type is identified as procedural (e.g., Safety Manual with step-by-step content), THE Converter SHALL use text extraction mode and apply procedural parsing

### Requirement 6: Differ from Book-to-Skill Approach

**User Story:** As a construction professional, I want the converter to be optimized for construction workflows rather than book reading, so that the output is practical for on-site use rather than study.

#### Acceptance Criteria

1. UNLIKE book-to-skill which organizes by chapters, THE Converter SHALL organize by construction-relevant categories (procedures, specifications, compliance, materials)
2. UNLIKE book-to-skill which emphasizes frameworks and mental models, THE Converter SHALL emphasize actionable procedures, quantified specifications, and compliance requirements
3. UNLIKE book-to-skill which generates a glossary of key terms, THE Converter SHALL generate a glossary of construction-specific terms, abbreviations, and acronyms
4. UNLIKE book-to-skill which generates patterns.md for techniques, THE Converter SHALL generate procedures.md for step-by-step construction tasks
5. UNLIKE book-to-skill which generates cheatsheet.md for decision tables, THE Converter SHALL generate quick-reference.md for material selection, troubleshooting, and compliance verification
6. UNLIKE book-to-skill which uses "Core Frameworks" in SKILL.md, THE Converter SHALL use "Quick Reference" and "Common Procedures" in SKILL.md
7. UNLIKE book-to-skill which assumes linear reading, THE Converter SHALL optimize for random-access lookup (field worker asking specific questions)
8. UNLIKE book-to-skill which emphasizes author voice and thinking style, THE Converter SHALL emphasize clarity, precision, and regulatory compliance

### Requirement 7: Handle Construction Document Metadata and Context

**User Story:** As a project manager, I want the skill to preserve document metadata (version, effective date, regulatory body, applicability), so that I can verify that the information is current and applicable to my project.

#### Acceptance Criteria

1. THE Converter SHALL extract and preserve document metadata: title, author/issuing body, version, effective date, revision history
2. THE Converter SHALL extract and preserve applicability information: geographic jurisdiction, project type, regulatory body, standards referenced
3. THE Converter SHALL extract and preserve cross-references: related documents, standards, codes, and internal references
4. WHEN metadata indicates a document is superseded or has a newer version, THE Converter SHALL warn the user during skill generation
5. THE Converter SHALL include metadata in the master SKILL.md file for reference
6. WHEN a user asks about document version or applicability, THE Skill SHALL retrieve metadata from SKILL.md

### Requirement 8: Support Construction-Specific Terminology and Abbreviations

**User Story:** As a new worker, I want the skill to explain construction-specific terms and abbreviations, so that I can understand the document without prior construction experience.

#### Acceptance Criteria

1. THE Converter SHALL identify and extract all construction-specific terms, abbreviations, and acronyms from the document
2. THE Converter SHALL generate a comprehensive glossary.md with: term, definition, context, and related terms
3. THE Converter SHALL include common construction abbreviations (e.g., PSI, OSHA, ADA, LEED, BIM, MEP, RFI, PCO)
4. THE Converter SHALL include material grades and standards (e.g., ASTM, ACI, AWS, AISC)
5. WHEN a user asks "what does X mean?", THE Skill SHALL retrieve the definition from glossary.md
6. THE Converter SHALL cross-reference glossary terms to the sections where they are used

### Requirement 9: Preserve Quantified Data and Specifications

**User Story:** As a quality inspector, I want the skill to preserve all quantified specifications (dimensions, tolerances, performance thresholds), so that I can verify that work meets exact requirements.

#### Acceptance Criteria

1. THE Converter SHALL extract and preserve all quantified specifications: dimensions, tolerances, performance thresholds, material grades, cost data
2. THE Converter SHALL preserve units of measurement (e.g., PSI, inches, degrees Celsius, hours)
3. THE Converter SHALL preserve tolerance ranges (e.g., ±0.5 inches, 10–15% variation)
4. THE Converter SHALL preserve performance criteria (e.g., minimum tensile strength, maximum deflection, fire rating)
5. THE Converter SHALL organize quantified data in specifications.md with: specification name, value, unit, tolerance, and application context
6. WHEN a user asks "what is the specification for X?", THE Skill SHALL retrieve the exact value, unit, and tolerance from specifications.md
7. THE Converter SHALL flag any ambiguous or conflicting specifications and warn the user during skill generation

### Requirement 10: Support Compliance Verification Workflows

**User Story:** As a compliance auditor, I want to use the skill to verify that work meets all applicable code requirements, so that I can conduct audits efficiently without manually searching through building codes.

#### Acceptance Criteria

1. THE Converter SHALL extract all regulatory requirements, code sections, and compliance criteria from the document
2. THE Converter SHALL organize compliance requirements in compliance.md with: code section, requirement, applicability, verification criteria, and penalties for non-compliance
3. THE Converter SHALL create a compliance checklist in quick-reference.md with: item, verification criteria, responsible party, and sign-off requirements
4. WHEN a user asks "is X compliant?", THE Skill SHALL retrieve applicable code sections and verification criteria from compliance.md
5. WHEN a user asks "what are the compliance requirements for X?", THE Skill SHALL retrieve all applicable requirements and verification criteria
6. THE Converter SHALL cross-reference compliance requirements to the sections where they are specified
7. THE Converter SHALL identify and flag any conflicting compliance requirements and warn the user during skill generation

---

## Workflow Differences from Book-to-Skill

### Input Differences
- **Book-to-Skill**: Accepts technical books (PDF/EPUB) with linear chapter structure
- **Construction Converter**: Accepts construction documents (PDF/EPUB) with diverse structures (hierarchical specs, procedural guides, tabular data, regulatory checklists)

### Document Type Detection
- **Book-to-Skill**: Asks "Technical or text-heavy?" to choose extraction method
- **Construction Converter**: Asks "What type of construction document?" to choose parsing strategy (Specification, Safety Manual, Material Datasheet, Building Code, Compliance Checklist, Installation Guide, Blueprint, Other)

### Knowledge Organization
- **Book-to-Skill**: Organizes by chapters, emphasizes frameworks and mental models
- **Construction Converter**: Organizes by construction-relevant categories (procedures, specifications, compliance, materials), emphasizes actionable procedures and quantified requirements

### Output Files
- **Book-to-Skill**: SKILL.md, chapters/, glossary.md, patterns.md, cheatsheet.md
- **Construction Converter**: SKILL.md, sections/, procedures.md, specifications.md, compliance.md, materials.md, glossary.md, quick-reference.md

### Use Cases
- **Book-to-Skill**: Study, reference, apply author's frameworks while working
- **Construction Converter**: On-site reference, compliance checking, safety procedures, material specifications, field worker lookup

### Metadata Handling
- **Book-to-Skill**: Extracts title, author, pages, chapters
- **Construction Converter**: Extracts title, issuing body, version, effective date, jurisdiction, applicability, regulatory references

### Terminology
- **Book-to-Skill**: Glossary of key concepts and frameworks
- **Construction Converter**: Glossary of construction-specific terms, abbreviations, acronyms, material grades, standards

---

## Constraints and Assumptions

1. The converter reuses the existing extraction pipeline (PDF/EPUB parsing, text extraction, file generation)
2. Construction documents are primarily text-based (specifications, manuals, guides, codes); visual elements (blueprints, diagrams) are noted but not extracted as images
3. The converter assumes documents are in English or can be translated to English
4. The converter assumes documents follow standard construction document formatting (sections, subsections, procedures, specifications, compliance requirements)
5. The converter does not validate compliance or provide legal advice; it organizes information for reference and verification
6. The converter does not modify or interpret regulatory requirements; it extracts and organizes them as written
7. The converter assumes users have basic construction knowledge or access to the glossary for unfamiliar terms

