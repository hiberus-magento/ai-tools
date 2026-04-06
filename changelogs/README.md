# Changelogs

This directory contains detailed changelogs for each version of Hiberus AI Tools.

## Structure

Each version has its own markdown file following semantic versioning:

```
changelogs/
├── README.md           # This file
├── 1.0.0.md           # Initial release
├── 1.1.0.md           # Minor version (new features, backwards compatible)
├── 1.1.1.md           # Patch version (bug fixes)
└── 2.0.0.md           # Major version (breaking changes)
```

## Semantic Versioning

We follow [Semantic Versioning 2.0.0](https://semver.org/):

- **MAJOR** (X.0.0): Breaking changes, incompatible API changes
- **MINOR** (1.X.0): New features, backwards compatible
- **PATCH** (1.0.X): Bug fixes, backwards compatible

## Changelog Format

Each changelog file contains:

1. **Overview** - Summary of the release
2. **New Features** - What's new (for MINOR/MAJOR)
3. **Changes** - What changed (improvements, deprecations)
4. **Bug Fixes** - What was fixed (for PATCH)
5. **Breaking Changes** - What breaks compatibility (for MAJOR)
6. **Technical Details** - Implementation specifics
7. **Migration Guide** - How to upgrade (if needed)
8. **Acknowledgments** - Credits and thanks

## Version History

| Version | Release Date | Type | Description |
|---------|--------------|------|-------------|
| [1.0.0](1.0.0.md) | 2024-04-06 | Initial | RFP Analysis Pipeline with 4 agents |

## Upcoming Releases

See each version file for roadmap and planned features.

## How to Read Changelogs

- 📋 **Before upgrading**: Read the changelog for the target version
- ⚠️ **Breaking changes**: Look for "Breaking Changes" section in MAJOR versions
- 🔧 **Migration**: Follow "Migration Guide" if present
- 🐛 **Bug fixes**: Check if your issue is resolved in PATCH versions

## Contributing

When contributing changes that will be part of a release:

1. Document your changes clearly
2. Follow the established format
3. Update the appropriate version file
4. Include migration notes if needed
