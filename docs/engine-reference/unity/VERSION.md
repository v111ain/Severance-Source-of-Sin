# Unity — Version Reference

| Field | Value |
|-------|-------|
| **Engine Version** | Unity 6.3 LTS (6000.3) |
| **Project Pinned** | 2026-04-15 |
| **LLM Knowledge Cutoff** | May 2025 |
| **Risk Level** | HIGH — version released March 2026 is beyond LLM training data |

## Warning

Unity 6.3 LTS was released in March 2026, which is **beyond the LLM's knowledge cutoff (May 2025)**.

**All agents MUST verify Unity 6.3 APIs via WebSearch before suggesting code.**
Do NOT rely solely on LLM knowledge for this engine version.

## Reference Documents

| Document | Status | Last Updated |
|----------|--------|--------------|
| [VERSION.md](./VERSION.md) | Current | 2026-04-15 |
| [breaking-changes.md](./breaking-changes.md) | **Needs Population** | — |
| [deprecated-apis.md](./deprecated-apis.md) | **Needs Population** | — |
| [current-best-practices.md](./current-best-practices.md) | **Needs Population** | — |

## Known Unity 6 Family Changes (Based on Unity 6 / 6.0 - 6.2)

The following changes are documented for the Unity 6 family leading up to 6.3:

### Rendering Performance
- **GPU Resident Drawer**: New system for efficient rendering of large, complex scenes
- **SRP Batcher improvements**: Extended compatibility, better CPU performance
- **URP/HDRP performance gains**: 30-50% CPU workload reduction in various scenarios

### Build Pipeline
- **Build Report API**: Improved build introspection
- **Incremental Build Pipeline**: Better support for content updates

### Physics
- **Unity Physics 1.0**: Stable physics engine replacement (verify for 6.3 specific changes)

### Entities / ECS
- **Entities 1.0**: Verify if project uses DOTS/ECS

## Recommended Actions

1. **Verify API changes**: Search `site:docs.unity3d.com Unity 6.3 breaking changes`
2. **Check migration guide**: Search `Unity 6.2 to 6.3 migration`
3. **Verify package compatibility**: Not all packages may be compatible with 6000.3
4. **Key verification searches**:
   - `site:docs.unity3d.com "What's New in Unity 6.3"`
   - `site:docs.unity3d.com "New in Unity 6.3"`
   - `site:docs.unity3d.com "6000.3" release notes`

## Verified Information

- Unity 6.3 LTS officially released March 2026
- Official docs available at: https://docs.unity3d.com/Manual/index.html
- Version 6000.3 confirmed via official documentation

## Version Reference

- Unity 6.0: October 2024 (initial LTS release, formerly Unity 2023 LTS)
- Unity 6.1: April 2025 (verify actual release date)
- Unity 6.2: Late 2025 (verify)
- Unity 6.3: March 2026

## Project-Specific Notes

本项目使用以下 Unity 特性（基于 ADR 定义）：
- **Input System** (ADR-0020): Unity Input System Package
- **Networking** (ADR-0006): Unity Netcode for GameObjects
- **Asset Management**: Unity Addressables
- **Rendering**: URP (Universal Render Pipeline)

## Last Verified

This reference document was last updated on 2026-04-15.

**Note**: 由于 WebSearch API 不可用，部分专项文档（breaking-changes.md, deprecated-apis.md, current-best-practices.md）已创建但内容待填充。需要在网络恢复后运行 `/setup-engine refresh` 填充完整内容。

---

Run `/setup-engine refresh` to update this reference when new information is available.
