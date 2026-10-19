# Bank Statement Summary for GPT

A [Codex](https://github.com/openai/codex) skill that turns bank statements into one summary CSV per month. It handles PDF, scanned PDF, Excel and CSV statements, in Thai or English, from any bank. The CSV has **one row per account, per currency, per month** and a fixed set of 14 columns, ready for accounting work.

For the Claude Code version, see [bank-statement-summary-for-claude](https://github.com/sinuthotmail/bank-statement-summary-for-claude).

> **ภาษาไทย:** สกิลสำหรับ Codex ที่แปลงสเตตเมนต์ธนาคาร (PDF, PDF สแกน, Excel, CSV ภาษาไทยหรืออังกฤษ) เป็น CSV สรุปรายบัญชีรายเดือน 14 คอลัมน์
> - ทุกแถวกระทบยอดได้ และห้ามแต่งตัวเลข: ถ้าสเตตเมนต์ไม่มีตัวเลขนั้น จะเว้นว่างและเขียนเหตุผลไว้
> - GPT-6.1 Sol High ทั้งทำงานและตรวจทุกกลุ่มเทียบกับสเตตเมนต์จริง โดย Worker กับ Reviewer เป็นคนละ agent

## Output

The columns, in this order:

`NO, Account Name, Product, Bank Name, Account type, Acc No., Opening Balance, Total Deposit, Total Withdrawal, Interest Income, Bonus, Withholding Tax, Currency, Closing Balance`

Interest, bonus and withholding tax have their own columns, so they are kept out of the deposit and withdrawal totals. Every row is then checked with:

```
Opening + Deposit − Withdrawal + Interest + Bonus − Withholding Tax = Closing
```

**Never invent a number.** If a value isn't in the statement, its cell is left blank and the reason goes in the notes file. The only derived figures are sums of printed transactions, and each is labelled as summed.

The output folder holds the files below. The `tranbank_` prefix comes from the process name, Tranbank.
- `tranbank_<YYYY-MM>.csv`: one per month, UTF-8 with BOM so Thai displays correctly in Excel.
- `tranbank_<YYYY-MM>_notes.csv`: every blank cell, every row that doesn't reconcile, and the notes for each row.
- `tranbank_skipped_files.csv`: each input file that produced no row (duplicate, empty, not a statement, …), with the evidence and what to do next. Written in Thai.

## How it works

1. The main session collects the files, extracts their text and groups the statements by bank layout.
2. A separate **GPT-6.1 Sol High** worker builds each group's rows.
3. A fresh **GPT-6.1 Sol High** reviewer checks them against the statements.
4. Findings go back to the same worker to fix, then to the same reviewer to re-check.
5. After two fix rounds, a fresh worker handles any unresolved rows, using the same model and effort.
6. The orchestrator merges the results into the CSVs and reports coverage and unresolved findings.

| Role | Model | Reasoning effort |
|---|---|---|
| Orchestrator | current main session | current session setting |
| Worker, including recovery | `gpt-6.1-sol` | `high` |
| Reviewer | `gpt-6.1-sol` | `high` |

The agent launch settings and the model log are defined in `SKILL.md`. The worker and the reviewer are always separate agents, even though they run on the same model.

## Requirements

- Codex with agent delegation, and access to `gpt-6.1-sol` at `high` reasoning effort.
- Python 3.9+ with `pdfplumber` (PDF) and `openpyxl` (Excel):
  ```bash
  pip install -r requirements.txt
  ```

## Install

**With the skill installer.** In Codex, run:

```
$skill-installer install https://github.com/sinuthotmail/bank-statement-summary-for-gpt/tree/main/skills/bank-statement-summary
```

**By hand.** Clone the repo, then link the skill into your Codex skills folder:

```bash
git clone https://github.com/sinuthotmail/bank-statement-summary-for-gpt.git
ln -s "$PWD/bank-statement-summary-for-gpt/skills/bank-statement-summary" ~/.codex/skills/bank-statement-summary
```

## Use

In Codex:

```text
$bank-statement-summary แปลง statement ใน ~/Downloads/statements.zip เป็น CSV โดยใช้ PDF เท่านั้น
```

You can also ask in plain words, e.g. "สรุป statement พวกนี้เป็น CSV", and the skill triggers on its own.

- **Passwords:** a PDF password written in the file name as `PW<token>` is used automatically. For any other password, the skill asks you once, for all the files together.
- **Zips:** zips from Windows that store Thai file names in a non-UTF-8 encoding are handled.

## Files

```
skills/bank-statement-summary/
  SKILL.md                    roles, model and effort settings, and the orchestrator's workflow
  agents/openai.yaml          Codex UI metadata (display name, short description, default prompt)
  references/rules.md         row rules: periods, duplicates, balances, totals, interest/WHT, field conventions
  references/worker.md        the worker's brief
  references/review.md        the reviewer's brief
  scripts/unzip_statements.py extract zips: Thai file names, nested zips, PW<token> passwords
  scripts/bulk_extract.py     text and tables of every file, written to a mirrored text tree, 8 files in parallel
  scripts/extract_text.py     text and tables of a single file
  scripts/write_csv.py        write the CSV template, check reconciliation, write the notes file
  scripts/merge.py            merge the worker groups into the per-month CSVs and the skipped list;
                              check coverage, duplicates and the month-to-month balance chain
tests/test_scripts.py         script tests on synthetic data
```

## Development

```bash
python3 -m unittest discover tests
```

The tests use synthetic data only. Real statements never belong in this repository.

## Privacy

During a session, Codex reads the statement contents through its configured tools and model. The output CSVs hold account numbers and balances. Keep statements and outputs out of this repository; the included `.gitignore` excludes PDFs, zips, CSVs, spreadsheets and images.

## Status

This is step 1 of a multi-step process. Later steps will build on this CSV, so the column names and their order are fixed.

## License

MIT. See [LICENSE](LICENSE).
