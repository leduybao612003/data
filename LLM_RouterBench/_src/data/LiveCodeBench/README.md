# LiveCodeBench Dataset

This directory contains the LiveCodeBench dataset files used for code generation evaluation.

## Data Download Instructions

The dataset files are not included in this repository due to size constraints. Download them using the following command:

```bash
huggingface-cli download \
  --repo-type dataset livecodebench/code_generation_lite \
  --include "test.jsonl" "test2.jsonl" "test3.jsonl" "test4.jsonl" "test5.jsonl" "test6.jsonl" \
  --local-dir ./lcb_release_v6
```

After downloading, move the files to this directory:
```bash
mv ./lcb_release_v6/*.jsonl ./data/LiveCodeBench/
```

## Prerequisites

Make sure you have the Hugging Face CLI installed:
```bash
pip install huggingface_hub[cli]
```

## Dataset Information

- **Source**: [livecodebench/code_generation_lite](https://huggingface.co/datasets/livecodebench/code_generation_lite)
- **Files**: 6 JSONL files containing programming problems and test cases
- **Total Records**: ~1,055 code generation problems
- **Format**: Each line contains a JSON object with problem details, test cases, and metadata

## File Structure

After download, this directory should contain:
- `test.jsonl` (400 records)
- `test2.jsonl` (111 records)  
- `test3.jsonl` (101 records)
- `test4.jsonl` (101 records)
- `test5.jsonl` (167 records)
- `test6.jsonl` (175 records)

## Usage

The LiveCodeBench evaluator will automatically load all JSONL files from this directory when running evaluations. The modified evaluator discovers and loads all `*.jsonl` files in this directory, combining them into a single dataset for comprehensive testing.

## Data Format

Each JSONL file contains one JSON object per line with the following structure:
- `question_title`: The title of the programming problem
- `question_content`: Detailed problem description and constraints
- `starter_code`: Initial code template (if any)
- `public_test_cases`: JSON string containing input/output test cases
- `metadata`: Additional information including function names and difficulty
- `platform`: Source platform (e.g., "atcoder", "codeforces")
- `contest_id`: Contest identifier
- `contest_date`: Date of the contest
- `difficulty`: Problem difficulty level
- `question_id`: Unique problem identifier

## Troubleshooting

If you encounter issues:

1. **Authentication**: You may need to login to Hugging Face:
   ```bash
   huggingface-cli login
   ```

2. **Network Issues**: If download fails, try downloading individual files:
   ```bash
   huggingface-cli download livecodebench/code_generation_lite test.jsonl --local-dir ./lcb_release_v6
   huggingface-cli download livecodebench/code_generation_lite test2.jsonl --local-dir ./lcb_release_v6
   # ... repeat for other files
   ```

3. **File Permissions**: Ensure you have write permissions in this directory.

## License

Please refer to the [original dataset repository](https://huggingface.co/datasets/livecodebench/code_generation_lite) for licensing information.