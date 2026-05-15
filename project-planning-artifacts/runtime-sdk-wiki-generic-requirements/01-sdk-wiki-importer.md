# Requirement: SDK Wiki Importer

## Goal

Allow users to import an external SDK Wiki Pack into CrewAgent Runtime so the SDK
LLM Wiki Module can use it during Agent sessions.

## Input

An SDK Wiki Pack directory or archive with:

```text
sdk-wiki.json
raw/
wiki/
index/
```

## Required Behavior

1. Read `sdk-wiki.json`.
2. Read `index/manifest.json`.
3. Validate `schemaVersion`, `sdkId`, `version`, and `entry`.
4. Validate required index files exist.
5. Validate content hash and index hash when present.
6. Validate page frontmatter can be parsed.
7. Register imported SDK Wiki under RuntimeStore.
8. Expose imported SDKs to the SDK LLM Wiki Module.

## RuntimeStore Layout

```text
RuntimeStore/
  sdk-wikis/
    <sdkId>/
      <version>/
        sdk-wiki.json
        raw/
        wiki/
        index/
```

## Failure Cases

- `SDK_WIKI_SCHEMA_INVALID`
- `SDK_WIKI_INDEX_INVALID`
- `SDK_WIKI_HASH_MISMATCH`
- `SDK_WIKI_PAGE_INVALID`
- `SDK_WIKI_VERSION_UNSUPPORTED`

## Acceptance Criteria

- Importing `sdk-wiki-packs/sam` registers SDK id `sam`.
- Runtime can list installed SDK Wikis.
- Runtime refuses an SDK Wiki Pack with missing `index/manifest.json`.
- Runtime refuses incompatible index schema instead of silently rebuilding it.
