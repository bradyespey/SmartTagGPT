# SmartTagGPT

Apple Shortcut that finds untagged Apple Reminders, asks OpenAI to choose one approved tag, applies the tag, and writes a local audit log.

This repository is public. It intentionally contains no API keys, reminder exports, reminder titles, personal logs, or other private reminder data.

## Current Status

The `Add Tags To Reminders` shortcut is installed and synced through iCloud. After a Mac refresh it began running without applying tags.

Root cause under investigation:

- The shortcut still references an old fine-tuned `gpt-3.5-turbo-0125` model.
- When OpenAI returns an error instead of a completion, the response has no `choices` array.
- The shortcut does not check `error.message` before extracting `choices[].message.content`, so it produces an empty `Repeat Results` value and writes a blank tag.
- The original fine-tuned model belongs to a different OpenAI API organization than the replacement API key. Keys from another organization cannot invoke that model.

The existing GPT-3.5 fine-tuned model remains usable for inference from its owning organization until OpenAI's scheduled GPT-3.5 fine-tune shutdown on October 23, 2026.

Do not run the shortcut against all untagged reminders until the one-reminder test below succeeds.

## Security

- Store the OpenAI key only in the local Shortcut header or another local credential location.
- Never commit an API key, bearer header, reminder export, JSONL training data, model/job IDs, or shortcut log.
- Screenshots must hide the complete bearer token.
- Revoke any key shown in a screenshot or shared outside the intended local Shortcut.
- Keep OpenAI auto top-up disabled.
- Reminder titles sent through the shortcut are transmitted to OpenAI for classification.

## Live Shortcut: `Add Tags To Reminders`

### 1. Find untagged reminders

Shortcuts does not provide a reliable empty-tag filter. The live workaround uses `Find Reminders` with **All** conditions:

- `Tags does not contain A` through `Tags does not contain Z`
- `Tags does not contain 0` through `Tags does not contain 9`
- Sort by `Date Created`
- Order `Latest First`
- Limit currently disabled

For testing, enable **Limit: 1**. Restore the unlimited setting only after a successful read-back test.

### 2. Repeat through reminders

Use `Repeat with each item in Reminders`.

For each reminder, call:

```text
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer [LOCAL OPENAI API KEY]
Content-Type: application/json
```

Current test request:

```json
{
  "model": "gpt-4.1-mini",
  "messages": [
    {
      "role": "user",
      "content": "Return only one tag that best categorizes the following reminder: '[Repeat Item]'. Choose only from: #Activities, #Auto, #Comment, #Computer, #Darktrace, #Errands, #Finance, #Games, #Gift, #Health, #Home, #Jobs, #Listen, #Logging, #Projects, #Read, #Shopping, #Travel, #Watch. Do not include explanation or punctuation."
    }
  ],
  "max_tokens": 20,
  "temperature": 0.2
}
```

The approved tag list should match the tags currently visible in Reminders. Remove obsolete tags from the prompt only after reviewing existing tagged reminders.

### 3. Parse the response

The current live flow is:

1. `Get Dictionary from Contents of URL`
2. `Get Value for choices in Dictionary`
3. `Repeat with each item in Dictionary Value`
4. `Get Value for message.content in Repeat Item 2`
5. `End Repeat`
6. `Set Tags of Repeat Item to Repeat Results`

Add these safety checks before setting the tag:

1. Read `error.message` from the response dictionary.
2. If an error exists, append the error to the log and stop the shortcut.
3. Confirm `Repeat Results` is not empty.
4. Confirm the returned text exactly matches one approved tag.
5. Only then set the reminder's Tags field.

### 4. Audit log

After a successful tag update:

1. Get the current date.
2. Format it using short date and short time.
3. Create: `Reminder: [Repeat Item], Tag: [Repeat Results], Time: [Date]`.
4. Append it to `edited_reminders.txt` in the Shortcuts folder.

The log contains reminder titles and must remain private and untracked.

## Recovery Test

1. Revoke the API key partially shown in the troubleshooting screenshot.
2. For temporary use of the existing fine-tune, sign into the OpenAI API organization that owns it and create a replacement project key there with a small usage budget and no auto top-up.
3. Update the Shortcut Authorization header locally and use the exact fine-tuned model ID shown by that organization.
4. Alternatively, use the main account's key with the base model `gpt-4.1-mini` and the constrained tagging prompt.
   - Enter that exact value. Do not add an `ft:` prefix, date, organization name, or suffix unless OpenAI supplied the complete ID from a successful new fine-tuning job.
5. Add `Show Result` immediately after `Get Contents of URL` temporarily.
6. Enable `Limit: 1` in `Find Reminders`.
7. Run the shortcut manually.
8. Confirm the response contains `choices[0].message.content` with one approved tag.
9. Confirm that exact tag appears on the one reminder.
10. Remove `Show Result`, then test a limit of 5 before processing the remaining reminders.

If the response contains `error`, record its message without recording the API key or complete reminder library.

## Legacy Fine-Tuning Workflow

The repository still contains the historical fine-tuning utilities:

- `convert_csv_to_jsonl.py`
- `fine_tuning.py`
- `check_fine_tune_status.py`
- `fine_tuning_request_test.py`

Historical flow:

1. Export tagged reminders from Apple Shortcuts to CSV.
2. Convert title/tag pairs to chat-format JSONL.
3. Upload the private JSONL file with purpose `fine-tune`.
4. Create and monitor a fine-tuning job.
5. Store the resulting model ID locally.
6. Use that model ID in the Shortcut.

The existing scripts and old fine-tuned GPT-3.5 model references are legacy and should not be rerun unchanged. Current OpenAI fine-tuning supports newer model families and requires a fresh compatibility and privacy review before uploading reminder-derived training data.

## Private Local Files

If the legacy scripts are updated later, keep all generated files outside the public repository:

```text
$HOME/Projects/Files/Reminders/openai_api_key.json
$HOME/Projects/Files/Reminders/openai_model_id.json
$HOME/Projects/Files/Reminders/openai_job_id.json
$HOME/Projects/Files/Reminders/exported_tagged_reminders.csv
$HOME/Projects/Files/Reminders/fine_tune_dataset.jsonl
```

The repository `.gitignore` excludes logs, exports, JSONL datasets, API-key files, and other local generated data.

## Troubleshooting

### Shortcut runs but tags stay empty

- Temporarily show the raw API result.
- Check `error.message` before `choices`.
- Verify the model exists and the project key can access it.
- Confirm `choices[0].message.content` is exactly one approved tag.
- Confirm `Set Tags` targets the outer `Repeat Item`, not `Repeat Item 2`.

### Shortcut finds too many reminders

- Allow Shortcuts to share large amounts of reminder data only if expected.
- Use Limit 1, then 5, before removing the limit.
- The A-Z and 0-9 filter is the current workaround for finding reminders with no tags.

### Tag output contains explanations

- Keep the prompt constrained to one approved tag.
- Reduce `max_tokens` to 20.
- Validate the returned tag before writing it.

## Source Documentation

This README consolidates the current shortcut screenshots and these older notes:

- `Automatically Add Tags to Reminders.md`
- `Automatically Add Tags to Reminders.1.md`
- `Automatically Add Tags to Reminders - Final Guide.md`
- `Automating Reminder Tagging with a Fine-Tuned GPT Model.md`

The older notes remain in the temporary archive until deletion is explicitly approved after the repaired shortcut is verified.
