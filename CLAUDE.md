# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Thera is a Julia-based pipeline for extracting drug-indication pairings from PubMed publications using both traditional MeSH analysis and AI-powered extraction via Llama 3.2. The project is optimized for HPC/SLURM environments with GPU acceleration.

## Development Commands

### Environment Setup
```bash
# Install Julia dependencies
julia --project=. -e 'using Pkg; Pkg.instantiate()'

# Quick start (auto-detects environment)
julia quick_start.jl

# Manual local setup
julia setup_and_run.jl

# HPC setup
julia hpc_setup_and_run.jl
```

### Core Analysis Commands
```bash
# Traditional MeSH-based analysis
julia scripts/extraction/pubmed_drug_indications.jl

# SLURM-compatible version
julia scripts/extraction/slurm_pubmed_drug_indications.jl

# AI-powered PubMed extraction (requires Ollama/Llama 3.2)
julia scripts/extraction/llama_drug_extractor.jl

# DrugBank indication text extraction (requires Ollama/Llama 3.2)
julia scripts/extraction/drugbank_llama_extractor.jl

# Enhanced naive extraction (single drug with raw response preservation)
julia scripts/extraction/enhanced_naive_llama_extractor.jl [drug_name]

# Batch enhanced naive extraction (multiple drugs efficiently)
julia scripts/extraction/batch_enhanced_naive_extractor.jl [start_index] [batch_size]

# Dual GPU batch enhanced naive extraction (maximum efficiency)
julia scripts/extraction/dual_gpu_batch_enhanced_naive.jl [start_index] [total_batch_size]
```

### Testing
```bash
# Validate HPC setup
julia validate_hpc_setup.jl

# Validate local setup  
julia validate_setup.jl
```

### SLURM Job Management
```bash
# Submit single drug AI extraction job
sbatch llama_extraction.slurm

# Submit dual GPU extraction job (2 drugs simultaneously)
./submit_dual_gpu_job.sh [drug1] [drug2]

# Submit DrugBank indication extraction job
./submit_drugbank_job.sh [drug_name]

# Submit batch enhanced naive extraction jobs
./submit_batch_enhanced_naive.sh                    # Submit single batch (1-200)
./submit_batch_enhanced_naive.sh 201 200            # Submit specific batch
./submit_batch_enhanced_naive.sh --multiple         # Submit multiple batches
./submit_batch_enhanced_naive.sh --status           # Show processing status

# Submit dual GPU batch enhanced naive extraction jobs
./submit_all_dual_gpu_batches.sh                    # Queue all remaining drugs (400 per job)
./submit_all_dual_gpu_batches.sh --status           # Show processing status

# Check job status
squeue -u $USER

# View logs
tail -f logs/llama_extraction_*.out
tail -f logs/dual_gpu_llama_extraction_*.out
tail -f logs/batch_enhanced_naive_*.out
tail -f logs/dual_gpu_batch_enhanced_*.out
```

## Architecture Overview

### Core Components

1. **Traditional Analysis Pipeline**
   - `scripts/extraction/pubmed_drug_indications.jl`: Main MeSH-based drug-disease analysis
   - `scripts/extraction/slurm_pubmed_drug_indications.jl`: SLURM-compatible version
   - Uses MeSH semantic type T047 (Disease or Syndrome) for classification

2. **AI-Powered Extraction Pipeline**
   - `scripts/extraction/`: Llama 3.2 based extraction scripts
   - `llama_drug_extractor.jl`: Optimized disease-parallel extraction approach

3. **HPC/SLURM Integration**
   - `scripts/slurm/`: SLURM job scripts with GPU allocation
   - `llama_extraction.slurm`: Main SLURM job template
   - Automated Ollama server management and model loading

### Data Flow

```
DrugBank XML → approved_drugs_extractor.jl → Approved Drugs Dict
MeSH Descriptors → mesh_t047_extractor.jl → Disease Classifications
                          ↓
     PubMed API ← scripts/extraction/pubmed_drug_indications.jl ← Both Inputs
                          ↓
               Drug-Disease Associations
                          ↓
     Llama 3.2 ← AI Extraction Scripts ← Publication Text
                          ↓
            Enhanced Indication Results
```

### Configuration System

- **Demo vs Production Mode**: Toggle `DEMO_MODE` in `pubmed_drug_indications.jl:41`
- **Environment Detection**: `quick_start.jl` auto-detects HPC vs local
- **Ollama Configuration**: `config/llama_config.jl` and environment variables
- **SLURM Settings**: Job parameters in `*.slurm` files

### Output Formats

- **Traditional Analysis**: JSON files with MeSH-based associations
- **AI Extraction**: JSON with confidence scores and evidence
- **Intelligent Extraction**: Two-phase verification with confirmation rates

## Development Patterns

### File Organization
- **Core scripts**: Root directory
- **Utilities**: `scripts/` with subdirectories by function
- **Configuration**: `config/` directory
- **Documentation**: `docs/` with usage and implementation guides
- **Testing**: `tests/` with unit and integration tests

### HPC Environment Patterns
- Always check for SLURM environment before using GPU resources
- Use `module load ollama julia` for HPC environments
- Implement proper cleanup of background processes (Ollama server)
- Include timeout and retry logic for model loading

### Error Handling
- Implement robust API rate limiting for PubMed requests
- Add checkpoint/resume functionality for long-running extractions
- Validate Ollama server connectivity before processing
- Log progress extensively for debugging SLURM jobs

## Key Dependencies

- **HTTP.jl**: PubMed API communication
- **JSON3.jl**: JSON processing and output
- **Dates.jl**: Timestamp handling
- **Ollama**: Llama 3.2 model hosting (HPC environments)
- **SLURM**: Job scheduling and GPU allocation

## Environment Variables

- `OLLAMA_HOST`: Ollama server endpoint (default: http://127.0.0.1:11434)
- `OLLAMA_MODELS`: Model storage directory
- `SLURM_JOB_ID`: Auto-set by SLURM for job identification

## Performance Considerations

- Use batch processing for large drug sets
- Implement parallel processing where possible
- Monitor GPU memory usage during AI extraction
- Consider storage I/O patterns for large output files
- Enable checkpointing for resumable long-running jobs