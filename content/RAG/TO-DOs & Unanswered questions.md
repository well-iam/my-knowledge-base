## ST TO-DOs
- where models come from
- analyze efficiency
- verbose UI
## Long term TO-DOs
- Currently there are three configs: .env, config.yaml and default.yaml. Consider pruning some.

## Unanswered Questions
- Where are conversations stored?
- If for a given file there's the .docx and .pdf version, which one should go through the ingestion pipeline and why?
- Should you take care of question/documents "prefixes" when switching model? (e5 was trained preprending either "passage: " or "query: " before embedding a text.)