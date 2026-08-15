# Changelog

## 0.1.0
- Initial release
- Six-phase pipeline: bound the range, harvest, cluster, triage, evidence, emit
- Three artifacts from one pass over the diffs: CHANGELOG entry, build-in-public post, résumé bullets
- Clustering by file-touch map rather than commit type or calendar window
- Three-bucket triage (shipped / infrastructure / invisible) with an honest invisible-commit count
- Counting rules restricting numbers to repo-countable, author-stated, or diff-computed sources
- Attribution guidance for repos with agent-authored commits
- Retroactive range boundaries for repos with no tags
