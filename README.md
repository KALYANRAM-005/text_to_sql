> 🎥 **Presentation video:** [Watch on YouTube](https://www.youtube.com/watch?v=REPLACE-WITH-VIDEO-ID)

# Text-to-SQL Demo: Ask a Database Questions in Plain English

This project turns a natural-language question into a SQL query and runs it on a database. For example:

> **Question:** What are the names of singers who are older than 40?
> **SQL:** `select name from singer where age > 40`
> **Result:** Joe Sharp, Rose White, John Nizinik

Everything runs from one Jupyter notebook, **`demo.ipynb`**. No command line or shell scripts are needed once the environment is set up.

The system uses a two-stage text-to-SQL approach. The models used here, a schema classifier and a T5 generator, were **trained and fine-tuned for this project** on the [Spider](https://yale-lily.github.io/spider) benchmark.

---

## How it works

A question goes through four steps:

| Step | What happens | Code |
|------|--------------|------|
| 1. Preprocess | Load the database schema and find database values that match words in the question | `preprocessing.py` |
| 2. Rank the schema | A **RoBERTa-large** classifier scores every table and column for relevance to the question | `schema_item_classifier.py`, `utils/classifier_model.py` |
| 3. Build the input | Keep the top 4 tables and top 5 columns per table, then add matched values and foreign keys | `text2sql_data_generator.py` |
| 4. Generate SQL | **T5** writes a SQL skeleton followed by the SQL, using beam search. The first candidate that runs on the database is returned | `text2sql.py`, `utils/text2sql_decoding_utils.py` |

Splitting the work this way (schema linking first, then SQL generation) shrinks the input T5 has to read and makes the problem easier.

## Project layout

| Path | Purpose |
|------|---------|
| `demo.ipynb` | **The demo. Start here.** |
| `preprocessing.py`, `schema_item_classifier.py`, `text2sql_data_generator.py`, `text2sql.py` | The four pipeline stages |
| `evaluate_text2sql_ckpts.py`, `utils/spider_metric/` | Spider evaluation code (not needed for the demo) |
| `utils/` | Model definition, datasets, SQL decoding, helpers |
| `data/` | Spider questions and `tables.json` (not in git) |
| `database/` | One SQLite file per database, 166 in total (not in git) |
| `models/` | The two trained checkpoints (not in git) |
| `requirements.txt` | Python dependencies |

---

## Setup

These steps were tested on Windows 11 with Miniconda and an NVIDIA RTX 3050 (6 GB). The pipeline also runs on CPU, but much more slowly.

### 1. Create the environment
The project needs Python 3.8 and the older library versions it was written for. In an **Anaconda / Miniconda Prompt**:

```sh
conda create -n nlp_env python=3.8.5 -y
conda activate nlp_env
```

### 2. Install PyTorch
For an NVIDIA GPU (CUDA 11.3 build, works on RTX 20/30 series):
```sh
pip install torch==1.11.0+cu113 torchvision==0.12.0+cu113 --extra-index-url https://download.pytorch.org/whl/cu113
```
For CPU only, use `pip install torch==1.11.0 torchvision==0.12.0`.

### 3. Install the other packages
`requirements.txt` lists `spacy==2.2.3`, which can't be installed with current pip. The project never imports spacy, so skip it:

```powershell
# Windows PowerShell
Get-Content requirements.txt | Where-Object { $_ -notmatch '^spacy' } | Set-Content req_nospacy.txt
pip install -r req_nospacy.txt
```
```sh
# macOS / Linux
grep -v '^spacy' requirements.txt > req_nospacy.txt
pip install -r req_nospacy.txt
```

Then add the notebook kernel and the NLTK data:
```sh
pip install ipykernel
python -m ipykernel install --user --name nlp_env --display-name "nlp_env"
python nltk_downloader.py
```

### 3b. Windows only: fix the "DLL load failed while importing _sqlite3" error
VS Code starts the environment's `python.exe` without activating conda, so Python 3.8 can't find a few DLLs and the notebook kernel crashes. Copy three files from the environment's `Library\bin` into its `DLLs` folder (adjust the path if your Miniconda is elsewhere):

```powershell
$env_ = "$env:USERPROFILE\miniconda3\envs\nlp_env"
foreach ($d in 'sqlite3.dll','libssl-1_1-x64.dll','libcrypto-1_1-x64.dll') {
    Copy-Item "$env_\Library\bin\$d" "$env_\DLLs\$d" -Force
}
```

### 4. Get the data, databases and trained models

**Download** (replace the placeholder links with your own):

| What | Download | Zip file(s) inside |
|------|----------|--------------------|
| Data and databases | [data.zip](https://example.com/REPLACE-WITH-DATA-LINK) and [database.zip](https://example.com/REPLACE-WITH-DATABASE-LINK) | `data.zip`, `database.zip` |
| **Trained models** (one link, both models) | [models](https://example.com/REPLACE-WITH-MODELS-LINK) | `text2sql_schema_item_classifier.zip` (RoBERTa schema classifier) and `text2sql-t5-base.zip` (fine-tuned T5) |

**Where to put the zip files.** Everything goes in the project folder, the one that contains `demo.ipynb`:

| Zip file | Put it in | Extract it into |
|----------|-----------|-----------------|
| `data.zip` | the project folder | the project folder |
| `database.zip` | the project folder | the project folder |
| `text2sql_schema_item_classifier.zip` | the `models/` folder (create it if it doesn't exist) | the `models/` folder |
| `text2sql-t5-base.zip` | the `models/` folder | the `models/` folder |

Extract each zip **"here"**, into the folder it sits in, and don't create an extra folder named after the zip. Each zip already contains a top-level folder with the same name as the zip. In Windows File Explorer, use right-click, then **Extract All**, and delete the suggested `\<zip name>` from the end of the destination path. Or run this from the project folder in PowerShell:

```powershell
Expand-Archive data.zip -DestinationPath .
Expand-Archive database.zip -DestinationPath .
Expand-Archive models\text2sql_schema_item_classifier.zip -DestinationPath models
Expand-Archive models\text2sql-t5-base.zip -DestinationPath models
```

After extracting you can delete the zip files and any `__MACOSX` folders.

**Check the result.** The final layout must look exactly like this, with no double nesting such as `data/data/spider`:

```
demo.ipynb
data/spider/dev.json
data/spider/tables.json
database/concert_singer/concert_singer.sqlite      (166 database folders)
models/text2sql_schema_item_classifier/dense_classifier.pt
models/text2sql_schema_item_classifier/config.json
models/text2sql-t5-base/checkpoint-39312/pytorch_model.bin
models/text2sql-t5-base/checkpoint-39312/config.json
```

If a folder is nested one level too deep, move its contents up one level. Section 1 of the notebook prints `OK` or `MISSING` for each required path, so you can check it there too.

---

## Run the demo

### In VS Code
1. Open the project folder in VS Code, then open `demo.ipynb`.
2. Click **Select Kernel** (top right) and choose **`nlp_env`** (under Python Environments or Jupyter Kernel). If it doesn't appear, reload the window.
3. Click **Run All**, or run the cells from top to bottom.

### In a browser
```sh
pip install notebook
jupyter notebook demo.ipynb
```

### What each section does
| Section | Description |
|---------|-------------|
| 1. Setup | Imports and the configuration cell. Also checks that all files are in place (prints `OK` or `MISSING`) |
| 2. Load the models | Loads RoBERTa and T5 once. Takes a minute or two |
| 3. The pipeline | Defines the pipeline. `text_to_sql(question, db_id)` runs everything |
| 4. Explore | `list_databases()` and `show_schema(db_id)` |
| 5. Ask a question | Three example questions on `concert_singer` |
| 6. Try your own | Edit `MY_DB` and `MY_QUESTION`, then re-run the cell |
| 7. Accuracy check | Runs the first 20 Spider dev questions and compares the result of the predicted SQL with the gold SQL |

For each question the notebook prints:
- the tables and columns the classifier kept, with their probabilities
- the SQL skeleton
- the final SQL
- the query result

### Ask your own question
In section 6, change two variables:
```python
MY_DB = "concert_singer"                  # any name from list_databases()
MY_QUESTION = "List the stadium names and their capacities, largest first."
```
The question has to match the database you chose.

### Settings (section 1, configuration cell)
| Setting | Default | Meaning |
|---------|---------|---------|
| `T5_PATH` | `models/text2sql-t5-base/checkpoint-39312` | Which fine-tuned T5 checkpoint to load. Point it at another checkpoint folder to compare models |
| `NUM_BEAMS` | 8 | Beam search width. Lower it to 4 if you run out of GPU memory |
| `TOPK_TABLES` / `TOPK_COLUMNS` | 4 / 5 | How many tables and columns per table reach T5 |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `DLL load failed while importing _sqlite3` when the kernel starts | Do step 3b above |
| `ModuleNotFoundError` in the notebook | The wrong kernel is selected. Choose `nlp_env` |
| `CUDA out of memory` | Lower `NUM_BEAMS`, close other GPU programs, or restart the kernel |
| Everything runs slowly, with no GPU found | Check `torch.cuda.is_available()`. You need the CUDA build of PyTorch from step 2 |
| `MISSING ./models/...` in section 1 | The checkpoint folder is nested or in the wrong place. See the layout in step 4 |
| `pip install` fails on `spacy` | Skip it, as in step 3 |
| Result is `sql placeholder` | None of the 8 beam candidates ran on the database. Rephrase the question |

## Limitations
- The models were trained on Spider, so they work best on questions that resemble Spider's, about its 166 databases.
- Predictions can be wrong, for example by adding an extra `limit 1`. Always check the SQL.
- It only generates read-only `SELECT` queries and runs them on local SQLite files.

## Credits
- **Spider** dataset: Yu et al., EMNLP 2018. [Dataset](https://yale-lily.github.io/spider)
- Base models that were fine-tuned: [T5](https://huggingface.co/t5-base) and [RoBERTa](https://huggingface.co/roberta-large)
